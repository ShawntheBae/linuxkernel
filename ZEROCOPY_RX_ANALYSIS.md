# TCP 제로카피 수신(TCP_ZEROCOPY_RECEIVE)과 page_pool 고갈 문제 분석

> 기준 소스: Linux **7.3.0-rc5** (이 저장소, 브랜치 `zerocopy`)
> 상태: **분석/설계 문서**. 코드는 아직 수정하지 않았음.
> 개정: 외부 리뷰를 반영해 해결 방안을 **A′ (VMA 슬롯 방식)**으로 재설계했다 (6.1.1~6.1.3).
> 표기: 📌 = 코드로 확인한 사실, 💡 = 제안/설계, ⚠️ = 주의/리스크

---

## 0. 요약 (TL;DR)

1. 📌 제로카피 수신은 `vm_insert_pages()`로 skb의 frag 페이지를 사용자 VMA에 그대로 매핑한다. 이때 **page `_refcount`가 +1** 된다 (`folio_get()`, [mm/memory.c:2510](mm/memory.c#L2510)).
2. 📌 그런데 같은 `getsockopt()` 호출 안에서, 매핑 직후 `tcp_recv_skb()` → `tcp_eat_recv_skb()`로 **끝까지 읽은 skb는 곧바로 해제**된다 ([net/ipv4/tcp.c:2328-2331](net/ipv4/tcp.c#L2328-L2331)). 일부만 읽은 skb는 큐에 남고, clone이나 deferred free가 있으면 실제 페이지 반환이 늦어질 수 있다.
3. 📌 skb 해제 시 page_pool은 **`page_ref_count(page) == 1`일 때만** 페이지를 재활용한다 ([net/core/page_pool.c:826-831](net/core/page_pool.c#L826-L831)). 사용자가 아직 매핑을 들고 있으므로 refcount는 2 → 재활용 실패 → `page_pool_return_netmem()`으로 **페이지가 pool에서 영구 이탈**(DMA unmap 후 일반 페이지가 됨).
4. 결과적으로 **마지막 pp 참조가 사용자 매핑 제거보다 먼저 반환되는 페이지는 재활용되지 않는다.** 일반적인 사용 패턴에서는 대부분의 매핑 페이지가 여기에 해당할 것으로 보이지만, **그 비율과 RX 드롭에 미치는 영향은 측정이 필요하다**(6.0). 재활용에 실패한 만큼 pool은 새 페이지를 할당(+DMA map)해야 하고, 고대역폭에서는 할당 지연/실패 → RX 링 리필 실패 → 드롭으로 이어질 수 있다. 이것이 "타이밍 때문에 반환이 제대로 안 되는" 현상의 정체다.
   - 핵심은 **순서 문제**: `skb 해제(pp put)` 가 `사용자 unmap` 보다 **먼저** 일어나면 재활용 실패, **나중에** 일어나면 재활용 성공.
5. 📌 추가로, skb가 해제되면 소켓 메모리 회계(`sk_rmem_alloc`)도 줄어든다. 그래서 **사용자가 들고 있는 페이지는 소켓 한도에서 빠진다.** 프로세스 RSS에는 잡히지만([mm/memory.c:2516](mm/memory.c#L2516)), 사용자가 매핑한 VMA 크기만큼 TCP 윈도우 제약 없이 페이지를 붙잡아 둘 수 있다.
6. 💡 해결 방향(권장, **A′**): **"지연된 pp 참조 해제"를 VMA 단위로** — 매핑한 페이지마다 page_pool 참조(`pp_ref_count`)를 하나 더 잡아 **mmap된 VMA의 슬롯 배열**에 기록한다. 그 주소의 매핑이 사라진 뒤(다음 RECEIVE의 zap, munmap, 프로세스 종료)에 참조를 놓는다. 그러면 마지막 pp put 시점에 `_refcount == 1`이 되어 재활용된다. VMA는 fork 복사·분할·이동을 금지해 고정하므로, 슬롯 수가 곧 보유 페이지 상한이 된다.
   - 업스트림에서 같은 문제를 푼 선례: **devmem TCP**(`SO_DEVMEM_DONTNEED`, `sk_user_frags`) 와 **io_uring zcrx**(refill queue). 설계 시 반드시 참고.

---

## 1. 배경: 제로카피 수신의 전체 데이터 경로

```
 NIC (DMA)                드라이버 NAPI poll (softirq)          TCP 스택                         사용자
 ─────────                ─────────────────────────────          ───────                          ──────
 RX 링 디스크립터 ◀── page_pool_alloc() 로 페이지 리필
     │  DMA write (헤더/페이로드 분리: header split)
     ▼
 완료 이벤트 ───────▶ napi_build_skb()/skb_add_rx_frag()
                       skb_mark_for_recycle()  (pp_recycle=1)
                       napi_gro_receive()  ── GRO 병합 ──▶ ip_rcv → tcp_v4_rcv
                                                            tcp_rcv_established
                                                            tcp_queue_rcv / tcp_try_coalesce
                                                            → sk_receive_queue (sk_rmem_alloc += truesize)
                                                                                          poll() 깨어남
                                                                                          getsockopt(TCP_ZEROCOPY_RECEIVE)
                                                            tcp_zerocopy_receive()  ◀──────┘
                                                              zap_vma_range(이전 매핑)
                                                              vm_insert_pages()  → page _refcount +1
                                                              tcp_recv_skb() → tcp_eat_recv_skb()
                                                                 → skb 해제 → napi_pp_put_page()
                                                                 → (_refcount==2) → pool 이탈 ❌
                                                                                          데이터 처리
                                                                                          madvise(MADV_DONTNEED)
                                                                                          → _refcount 0 → buddy로 free
```

### 1.1 제로카피가 성립하기 위한 전제 조건 📌

`can_map_frag()` ([net/ipv4/tcp.c:1905](net/ipv4/tcp.c#L1905)) 가 결정한다.

| 조건 | 의미 | 실 NIC에서 필요한 설정 |
|---|---|---|
| `skb_frag_size(frag) == PAGE_SIZE` | frag 하나가 정확히 한 페이지 | MTU를 `4096 + 헤더`(예: IPv6 4096+40+32)로, 또는 HW-GRO/LRO |
| `skb_frag_off(frag) == 0` | 페이지 시작부터 데이터 | **header split** 필수 (`ethtool -G <dev> tcp-data-split on`) |
| `!PageCompound(page)` | 고차(order>0) 페이지 불가 | page_pool `order=0` |
| `page->mapping == NULL` | 파일 페이지 등 불가 | page_pool 페이지는 `_pp_mapping_pad`로 0 유지 |
| `skb_frags_readable(skb)` | devmem(net_iov) 아님 | — |
| frag_list 없음 | `skb_advance_to_frag()` 에서 거부 | — |

조건을 만족하지 못하는 부분은 `recv_skip_hint`로 사용자에게 알려 `read()`/copybuf로 복사 수신하게 한다.

### 1.2 `tcp_zerocopy_receive()` 흐름 📌 ([net/ipv4/tcp.c:2195](net/ipv4/tcp.c#L2195))

1. 주소 정렬 체크, `inq = tcp_inq(sk)`
2. `inq <= copybuf_len` 이면 `receive_fallback_to_copy()` (작은 데이터는 복사)
3. `find_tcp_vma()` — per-VMA lock 우선, 실패 시 `mmap_read_lock`
4. `TLB_CLEAN_HINT` 플래그가 없으면 **이번에 매핑할 범위를 먼저 `zap_vma_range()`** → 이전 호출에서 매핑했던 페이지의 `_refcount`를 여기서 내려놓음
5. skb를 순회하며 매핑 가능한 frag의 page를 최대 32개(`TCP_ZEROCOPY_PAGE_BATCH_SIZE`)씩 모아 `vm_insert_pages()`
6. 실패(`-EBUSY`) 시 TLB_CLEAN_HINT면 zap 후 재시도, 아니면 롤백하고 `recv_skip_hint` 증가
7. 남은 자투리(straggler)는 copybuf로 복사 (`tcp_zc_handle_leftover`)
8. **`copied_seq` 갱신 → `tcp_recv_skb(sk, seq, &offset)`** — 다 읽은 skb들을 `tcp_eat_recv_skb()`로 큐에서 제거/해제 → `tcp_cleanup_rbuf()`로 ACK/윈도우 갱신

> 8번이 핵심이다. 매핑(5번)과 skb 해제(8번)가 **같은 시스템콜 안에서, 사용자가 unmap할 기회가 오기 전에** 일어난다.

---

## 2. skb (`struct sk_buff`) 분석

### 2.1 skb는 "NIC에서 받자마자" 만들어지는가? 📌

정확히는 **아니다.** 단계가 나뉜다.

1. **NIC**는 skb를 모른다. 드라이버가 미리 RX 링에 넣어 둔 **page_pool 페이지의 DMA 주소**에 패킷 바이트를 쓰고 완료 디스크립터를 남길 뿐이다.
2. IRQ → `napi_schedule()` → **NET_RX softirq에서 드라이버의 NAPI poll 함수**가 완료 디스크립터를 읽으며 그때 skb를 만든다.
   - `napi_build_skb(data, frag_size)` ([net/core/skbuff.c:576](net/core/skbuff.c#L576)): 이미 데이터가 있는 버퍼(헤더 버퍼)를 skb의 head로 감싼다. `struct sk_buff` 자체는 per-CPU napi skb 캐시(slab)에서 꺼냄.
   - header split인 경우: 헤더는 head(linear)에, 페이로드 페이지는 `skb_add_rx_frag()`로 `skb_shinfo(skb)->frags[]`에 붙음.
   - `skb_mark_for_recycle(skb)` ([include/linux/skbuff.h:5467](include/linux/skbuff.h#L5467)) → `pp_recycle = 1`
3. 이후 GRO가 같은 플로우의 skb들을 하나로 병합 (frag 이어붙이기, 최대 `MAX_SKB_FRAGS`; 넘으면 frag_list).

즉 skb는 "메타데이터 껍데기", 실제 데이터 바이트는 page_pool 페이지에 있다. 제로카피는 이 **페이지**를 사용자에게 주는 것이다.

### 2.2 구조 (제로카피와 관련된 필드 위주) 📌

```
struct sk_buff  (include/linux/skbuff.h:886)
 ├─ next/prev, sk, dev, tstamp
 ├─ len (935), data_len (936)          len = 전체, data_len = frags에 있는 양
 ├─ pp_recycle:1 (962)                 frag/head 페이지가 page_pool 소속 → 해제 시 pool로 반환 시도
 ├─ destructor (924)                   TCP 수신 큐에선 sock_rfree → sk_rmem_alloc 감소
 ├─ unreadable:1 (1033)                devmem(net_iov) frag → CPU가 못 읽음
 ├─ alloc_cpu (1039)                   skb_attempt_defer_free()가 이 CPU로 해제를 미룸
 ├─ head, data, tail, end (1092~)      linear 영역(헤더)
 ├─ truesize (1096)                    소켓 메모리 회계에 쓰이는 "실제 메모리 크기"
 ├─ users                              skb 자체 참조 카운트
 └─ cb[48]                             TCP_SKB_CB(skb): seq, end_seq, tcp_flags, has_rxtstamp ...

skb_end_pointer(skb) 위치에 붙는 struct skb_shared_info (593)
 ├─ nr_frags, gso_size, gso_segs, gso_type
 ├─ frag_list                          GRO가 frag 배열을 넘칠 때 체인
 ├─ dataref                            데이터(head+frags) 공유 참조 카운트 (clone 시 증가)
 └─ frags[MAX_SKB_FRAGS]               skb_frag_t { netmem, offset, len }  (361)
```

### 2.3 참조 카운트가 4층이다 (문제 이해의 핵심)

| 카운터 | 대상 | 누가 올리고/내리나 |
|---|---|---|
| `skb->users` | sk_buff 구조체 | `skb_get()` / `kfree_skb()` |
| `shinfo->dataref` | head + frags 묶음 | `skb_clone()` (tcpdump, AF_PACKET 등) |
| `page->pp_ref_count` | page_pool 페이지의 **pool 내부 참조** | frag 분할, `skb_pp_frag_ref()`, `page_pool_ref_netmem()` |
| `page->_refcount` | 페이지 자체 (mm 관점) | **`vm_insert_pages()`(folio_get)**, `put_page()`, zap |

page_pool의 재활용 판단은 **`pp_ref_count`가 0이 되는 순간(마지막 pp put)에 `_refcount == 1`인지**로 결정된다. 제로카피는 `_refcount`만 올리고 `pp_ref_count`는 건드리지 않기 때문에, pool 입장에서는 "누군가 몰래 붙잡고 있는 페이지"가 되어 버린다.

### 2.4 skb 해제 경로 📌

```
tcp_eat_recv_skb()                       net/ipv4/tcp.c:1614
 ├─ sock_rfree()                         sk_rmem_alloc -= truesize  ← 회계에서 빠짐
 └─ skb_attempt_defer_free()             net/core/skbuff.c:7358
     ├─ alloc_cpu == 현재 CPU → 즉시 해제
     └─ 다른 CPU → 그 CPU의 defer_list에 넣고 (defer_max=128의 절반에서 IPI) 나중에 해제
         → skb_release_data()            net/core/skbuff.c:1099
             → __skb_frag_unref(frag, pp_recycle)
                 → napi_pp_put_page()    net/core/skbuff.c:1029
                     → page_pool_put_full_netmem()
                         → pp_ref_count-- ; 0이 되면
                         → __page_pool_put_page()      net/core/page_pool.c:840
                             ├─ _refcount==1 && !pfmemalloc → cache / ptr_ring 으로 재활용 ✅
                             └─ else → recycle_stat(released_refcnt)
                                      page_pool_return_netmem()   ← DMA unmap, put_page, pool 이탈 ❌
```

---

## 3. page_pool 동작 요약 📌

- 드라이버 RX 큐마다 하나의 pool. 페이지 공급 순서: `alloc.cache`(NAPI-local, lockless) → `ptr_ring`(다른 CPU에서 반환된 페이지) → **slow path**(`alloc_pages_bulk_node` + DMA map).
- 회계: `pages_state_hold_cnt`(pool이 할당한 수) − `pages_state_release_cnt`(pool을 떠난 수) = **inflight** ([net/core/page_pool.c:705](net/core/page_pool.c#L705)).
- 통계 (`CONFIG_PAGE_POOL_STATS`, [include/net/page_pool/types.h](include/net/page_pool/types.h)):
  - alloc: `fast`, `slow`, `slow_high_order`, `empty`(ring 비어 slow path 강제), `refill`, `waive`(NUMA 불일치로 버림)
  - recycle: `cached`, `cache_full`, `ring`, `ring_full`, **`released_refcnt`** ← 제로카피 문제의 직접 지표
- 조회: `./tools/net/ynl/pyynl/cli.py --spec Documentation/netlink/specs/netdev.yaml --dump page-pool-stats-get` 또는 드라이버별 `ethtool -S`.

> "고갈"의 정확한 의미: page_pool은 고정 크기 풀이 아니라 **재활용 캐시 + 할당기**다. 재활용이 실패하면 pool이 비고(`empty`↑), 매 패킷마다 slow path 할당 + DMA map(+IOMMU 시 매우 비쌈)을 하게 된다. 고대역폭에서 이 비용/지연이 RX 링 리필 속도를 못 따라가거나, `GFP_ATOMIC` 할당이 실패하면 **링에 버퍼가 비어 NIC가 드롭**한다 (드라이버 통계의 `rx_alloc_fail`, `rx_missed`, `rx_no_buffer` 류). page_pool을 쓰지 않고 자체 page-reuse(`page_count()==1` 검사, `pagecnt_bias`)를 하는 드라이버도 같은 이유로 재사용에 실패한다.

---

## 4. 핵심 문제: 반환 타이밍(순서) 분석

### 4.1 시퀀스

**Case A — 현재 동작 (거의 항상 이쪽)**

```
t0  getsockopt(ZC)  vm_insert_pages        _refcount 1→2, pp_ref 1
t1  getsockopt(ZC)  tcp_eat_recv_skb       pp_ref 1→0 → 검사: _refcount==2 → ❌ released_refcnt
                                           page_pool_return_netmem: DMA unmap, put_page → _refcount 1
t2  user            madvise(DONTNEED)      _refcount 1→0 → buddy 할당기로 free (pool 아님)
```

**Case B — 재활용되려면 필요한 순서**

```
t0  vm_insert_pages                        _refcount 2, pp_ref 1
t1  user unmap (zap + TLB flush 완료)        _refcount 1
t2  (지연된) pp put                          pp_ref 0 → _refcount==1 → ✅ pool로 재활용
```

현재 코드 구조에서는 **끝까지 읽은 skb**의 경우 t1(skb 해제)이 사용자 unmap보다 앞서는 것이 일반적이다. 반면 다음 경우에는 사용자 unmap이 먼저 일어나 Case B가 될 수 있다. 어느 쪽이 될지는 우연에 달려 있고, 이것이 "타이밍적인 오류"로 관찰되는 비결정성의 원인일 가능성이 높다.

- 일부만 읽은 skb: `tcp_recv_skb()`는 끝까지 읽은 skb만 제거한다([tcp.c:1637](net/ipv4/tcp.c#L1637)).
- skb clone(tcpdump 등)이 데이터를 함께 쥐고 있는 경우
- `skb_attempt_defer_free()`로 다른 CPU에서 늦게 해제되는 경우

> 참고: [tcp.c:2331](net/ipv4/tcp.c#L2331)의 주석 "SKB cannot be free cuz refcount = 2"는 부정확하다. 페이지의 `_refcount`가 2여도 **skb는 해제된다.** 페이지만 살아남아 pool에서 이탈할 뿐이다.

### 4.2 파생 문제

| # | 문제 | 메커니즘 |
|---|---|---|
| P1 | pool 재활용률 저하 (비율은 측정 필요) | 위 Case A |
| P2 | slow path 할당 + DMA map/unmap 폭증 | 매 페이지 `page_pool_return_netmem` + 새 `alloc_pages` |
| P3 | RX 링 리필 실패 → 드롭 | softirq의 `GFP_ATOMIC` 할당 실패, 메모리 압박 |
| P4 | **소켓 회계 우회** | skb 해제 시 `sock_rfree`로 `sk_rmem_alloc` 감소 → 윈도우가 다시 열려 송신측이 계속 보냄 → 사용자가 **새 주소 범위로** 계속 받으며 unmap하지 않으면 VMA 크기만큼 쌓임. RSS에는 반영되지만 소켓 한도와 TCP 윈도우로는 제한되지 않음. page_pool 할당은 `__GFP_ACCOUNT`가 없어 memcg 과금도 없는 것으로 보임(확인 필요) |
| P5 | pfmemalloc 페이지 | 메모리 압박 시 받은 페이지는 원래 재활용 불가 (P3과 겹치면 악순환) |
| P6 | TLB shootdown 비용 | `madvise(MADV_DONTNEED)`/zap마다 IPI. 멀티스레드 프로세스에서 비쌈 |

---

## 5. 그 밖에 page_pool(수신 페이지)이 고갈/정체되는 원인들

제로카피와 무관하게도 발생하며, 실험 시 원인 분리를 위해 알고 있어야 한다.

| 원인 | 메커니즘 | 확인 위치 / 방법 |
|---|---|---|
| **읽지 않는 소켓의 수신 큐** | 앱이 느리거나 멈추면 skb(=pp 페이지)가 `sk_receive_queue`에 최대 rcvbuf까지 머묾. 소켓 수 × `tcp_rmem[2]` | `ss -tm` (`r`/`rb`), inflight 증가 |
| **Out-of-order 큐** | 손실 시 `tcp_data_queue_ofo()` ([net/ipv4/tcp_input.c:5381](net/ipv4/tcp_input.c#L5381))가 재전송 올 때까지 페이지 보유. 고BDP에서 큼 | `nstat TcpExtTCPOFOQueue` |
| **truesize 팽창** | 작은 페이로드가 4K 페이지 하나를 통째 점유 → rcvbuf 금방 참 → prune/collapse | `TcpExtPruneCalled`, `TCPRcvQDrop` |
| **tcp_collapse** | 메모리 압박 시 데이터를 새 linear skb로 **복사** → 페이지는 반환되나 이후 데이터는 `can_map_frag` 실패 → 제로카피율 급락 | `TcpExtTCPRcvCollapsed`, tcp_mmap의 "% mmap'ed" |
| **frag 분할 페이지** | MTU 1500 드라이버가 페이지를 여러 frag로 쪼갬. frag 하나만 오래 살아도 페이지 전체가 pool에 못 돌아옴 | — |
| **skb clone 보유자** | tcpdump/AF_PACKET, NFQUEUE, `SO_TIMESTAMPING` 에러 큐 등이 clone을 오래 잡음 → `dataref` 유지 | 캡처 끄고 비교 |
| **deferred free** | `skb_attempt_defer_free()`: alloc CPU가 idle/바쁘면 해제 지연 (`net.core.skb_defer_max`) | sysctl 0으로 비교 |
| **ptr_ring full** | 다른 CPU에서 대량 반환 시 ring이 차면 그냥 반환(이탈) | `recycle.ring_full` |
| **NUMA 불일치** | pool nid와 다른 노드 페이지는 refill 시 버림 | `alloc.waive` |
| **pool 파괴 지연** | inflight가 남으면 `page_pool_release_retry()`가 "stalled pool shutdown ... inflight" 경고 ([net/core/page_pool.c:1235](net/core/page_pool.c#L1235)) — 링 재구성(ethtool -G/-L) 시 메모리 잔류 | dmesg |
| **드라이버 버그** | pp 페이지에 `put_page` 직접 호출, 잘못된 pool로 반환 → inflight 음수/누수 | `Negative(%d) inflight` WARN |

---

## 6. 수정 방안

### 6.0 사전 단계 — 먼저 측정 (코드 수정 최소)

수정 전에 "정말 released_refcnt로 빠지는지"를 수치로 확인해야 이후 효과를 비교할 수 있다.

```bash
# page_pool 통계 (전/후 diff)
./tools/net/ynl/pyynl/cli.py --spec Documentation/netlink/specs/netdev.yaml --dump page-pool-stats-get
# pool 이탈 추적
bpftrace -e 'tracepoint:page_pool:page_pool_state_release { @rel[args->pool] = count(); }
             tracepoint:page_pool:page_pool_state_hold    { @hold[args->pool] = count(); }'
# TCP 지표
nstat -az | egrep 'OFO|Collapse|Prune|RcvQDrop'
```

기대 결과: 제로카피 모드에서 `recycle.released_refcnt`와 `alloc.slow`가 증가한다. `released_refcnt` 증가량을 mmap된 페이지 수(tcp_mmap 출력의 MB × % mmap'ed × 256)로 나누면 **재활용 실패 비율의 근사치**가 나온다. 이 비율이 실제로 얼마인지가 첫 번째 측정 목표다.

⚠️ 이 나눗셈은 정확한 실패율이 아니다. `released_refcnt`는 pool 단위 카운터라 **같은 RX 큐의 다른 트래픽**과 **다른 반환 원인**(clone, 다른 소비자 등)이 섞인다. 정확히 재려면 다음 중 하나가 필요하다.
- **격리된 RX 큐**에서 측정: `ethtool -N`(ntuple)로 테스트 플로우만 한 큐로 보내고 다른 트래픽은 다른 큐로 보낸 뒤, 그 큐의 pool만 본다.
- **대상 페이지 별도 추적**: 매핑한 페이지(pfn)를 bpftrace로 기록하고, `page_pool_state_release`에서 그 페이지만 센다.

### 6.1 `net/ipv4/tcp.c` 수정 방안 비교

| 방안 | 개요 | 장점 | 단점 |
|---|---|---|---|
| **A′. 지연 pp 참조 해제 — VMA 슬롯 (권장)** | 매핑하는 페이지마다 `page_pool_ref_netmem()`으로 pp 참조를 +1 해서 **VMA의 슬롯 배열**에 기록. skb는 지금처럼 해제. 그 주소의 매핑이 사라지면(zap/munmap/exit) pp 참조 -1 | skb/수신 큐 로직 그대로, 재활용 순서 보장, hot path 할당 없음, 정리 지점 하나(`.close`) | `vm_ops` 훅 필요, 새 모드에서는 VMA 분할·이동·fork 상속 불가 |
| B. skb 자체를 보류 | 다 읽은 skb를 해제하지 않고 "zc pending" 리스트로 옮겼다가 반환 시 해제 | 회계(sk_rmem_alloc)가 자연스럽게 유지돼 P4도 해결 | 윈도우가 안 열려 처리량 저하, skb 단위라 부분 반환 어려움, 메모리 낭비(헤더 등) |
| C. 회계/제한만 추가 | 미반환 페이지 수를 세어 한도 초과 시 복사로 폴백 | 단순 | 재활용 문제(P1~P3)는 해결 안 됨 |
| D. 기존 메커니즘 사용 | devmem TCP 또는 io_uring zcrx | 업스트림에서 이미 이 문제를 설계적으로 해결 | dmabuf/io_uring 기반이라 API가 완전히 다름 |

#### 6.1.1 방안 A′ 상세 설계 — "VMA 슬롯" 방식 💡 (리뷰 반영 개정판)

> 초기 설계(소켓마다 xarray, 키 = 사용자 가상주소)는 리뷰에서 결함이 확인되어 폐기했다. 폐기 이유는 6.1.2에 정리했다.
> 개정판은 보류 기록을 **소켓이 아니라 mmap된 VMA 하나에** 묶는다. 이렇게 하면 처음 설계보다 오히려 구조가 단순해진다.

**핵심 아이디어 (그대로 유지)**

매핑한 page_pool 페이지마다 pp 참조(`pp_ref_count`)를 하나 더 잡아 둔다. 그 참조는 **그 페이지의 사용자 매핑이 사라진 뒤에만** 놓는다. 그러면 마지막 pp put 시점에 `_refcount == 1`이므로 재활용된다.

**(1) 활성화: mmap 전에 소켓 옵션 하나**

```c
int one = 1;
setsockopt(fd, IPPROTO_TCP, TCP_ZEROCOPY_RECYCLE, &one, sizeof(one));  /* 새 옵션 */
addr = mmap(NULL, len, PROT_READ, MAP_SHARED, fd, 0);
```

- 이 옵션을 켜지 않은 소켓은 기존 동작 그대로다.
- 구버전 커널은 이 옵션을 모르면 `-ENOPROTOOPT`을 돌려주므로, 사용자 공간이 기능 지원 여부를 바로 알 수 있다. 초기 설계에서 `zc.flags`의 미정의 비트가 무시되던 문제가 이걸로 해결된다.

**(2) 자료구조: VMA마다 고정 크기 슬롯 배열**

```c
/* 개념 코드 */
struct tcp_zc_region {
	struct kref	ref;          /* vm_ops->open/close 짝 맞추기 */
	struct sock	*owner;       /* mmap한 소켓. tcp_mmap()에서 owner = sk. 비교 전용 */
	unsigned long	nr_slots;     /* = vma_pages(vma), mmap 이후 변하지 않음 */
	unsigned long	nr_held;      /* 0이 아닌 슬롯 수 (상한 검증 기준, (4)의 갱신 규칙 참고) */
	netmem_ref	slots[];      /* slots[i] = (vm_start + i*PAGE_SIZE)에 매핑해 준 pp 페이지 */
};
```

- `tcp_mmap()`에서 **mmap 시점에 한 번만** 할당한다(`kvzalloc`). 할당에 실패하면 mmap이 실패한다.
  - 그래서 수신 경로(hot path)에는 메모리 할당이 없다. 리뷰의 "xa_store 실패 → 누수" 문제가 원천적으로 사라진다.
- 슬롯 인덱스 = `(addr - vma->vm_start) >> PAGE_SHIFT`. 키가 "어느 프로세스의 어느 주소"가 아니라 **"이 VMA의 몇 번째 페이지"**이므로, 프로세스 간 충돌이나 주소 재사용 문제가 생기지 않는다.
- `vma->vm_private_data = region`. `find_tcp_vma()`로 찾은 VMA에서 바로 꺼내 쓴다.
- `region->owner = sk`: 이 영역을 mmap한 소켓을 기록한다. ⚠️ `find_tcp_vma()`는 `vm_ops`만 확인하고 **어느 소켓의 VMA인지는 확인하지 않는다** ([tcp.c:2169-2192](net/ipv4/tcp.c#L2169-L2192)). 그래서 소켓 B로 RECEIVE하면서 소켓 A의 영역 주소를 넘길 수 있다. 소유자 검사는 (7)에서 다룬다.
  - 포인터 수명: VMA가 파일 참조를 쥐고 있으므로 region이 살아 있는 동안 소유 소켓도 살아 있다. 게다가 비교에만 쓰고 역참조하지 않는다.

**(3) VMA 고정: 복제·분할·이동 금지**

| 설정 | 막는 것 | 근거 📌 |
|---|---|---|
| `VM_DONTCOPY` | fork 시 자식에게 PTE 복사 | 현재는 `VM_MIXEDMAP`이 `VM_COPY_ON_FORK`에 포함돼 **복사된다** ([mm/memory.c:1570](mm/memory.c#L1570), [include/linux/mm.h:695](include/linux/mm.h#L695)). 사용자가 `MADV_DOFORK`로 풀 수도 없다(`VM_MIXEDMAP` ∈ `VM_SPECIAL`, [mm/madvise.c:1394](mm/madvise.c#L1394)) |
| `VM_DONTEXPAND` | mremap으로 확장 | |
| `.may_split` → `-EINVAL` | 부분 munmap / mprotect로 VMA 분할 | [mm/vma.c:554](mm/vma.c#L554) |
| `.mremap` → `-EINVAL` | VMA 이동 | 실패 시 커널이 새 VMA를 되돌려 unmap한다 ([mm/mremap.c:1295-1311](mm/mremap.c#L1295-L1311)) |

이 VMA는 생성부터 소멸까지 **하나의 mm, 하나의 주소 범위**로 고정된다. 그래서 슬롯 배열 크기가 곧 이 영역이 쥘 수 있는 페이지의 상한이 된다.

⚠️ mremap 실패 경로에서도 `copy_vma()`가 새 VMA에 `.open`을 부르고([mm/vma.c:2017](mm/vma.c#L2017)), 되돌릴 때 그 VMA에 `.close`가 불린다. 그래서 region은 **kref로** 관리한다(`.open` = get, `.close` = put). 마지막 put에서만 슬롯을 정리하고 해제한다. 이걸 빼면 use-after-free가 난다.

**(4) 수신 경로 변경 (tcp_zerocopy_receive)**

기존 `zap_vma_range()` 호출 두 곳([tcp.c:2080](net/ipv4/tcp.c#L2080), [tcp.c:2245](net/ipv4/tcp.c#L2245))을 아래 헬퍼로 바꾼다.

```c
static void tcp_zc_zap(struct vm_area_struct *vma, unsigned long addr, unsigned long len)
{
	struct tcp_zc_region *r = vma->vm_private_data;

	zap_vma_range(vma, addr, len);            /* ① PTE 제거 + TLB flush 완료 */
	if (!r)
		return;                               /* 기존 모드 */
	for (각 슬롯 i in [addr, addr+len)) {
		if (r->slots[i]) {
			napi_pp_put_page(r->slots[i]);    /* ② 추가 참조가 없으면 _refcount==1 → 재활용 ✅ */
			r->slots[i] = 0;
			r->nr_held--;
		}
	}
}
```

`vm_insert_pages()`가 성공한 페이지마다(최초 삽입과 `-EBUSY` 후 재삽입 **둘 다**) 다음을 한다.

```c
idx = (page_addr - vma->vm_start) >> PAGE_SHIFT;
if (r->slots[idx]) {                   /* 이전 페이지가 남아 있다면:        */
	napi_pp_put_page(r->slots[idx]);   /*   삽입 성공 = 그 주소의 PTE가 비어 있었음  */
	r->slots[idx] = 0;                 /*   = 이전 페이지는 이 VMA에서 unmap됨 → 안전 */
	r->nr_held--;
}
if (skb->pp_recycle && netmem_is_pp(nm)) {
	page_pool_ref_netmem(nm);          /* pp_ref_count +1 */
	r->slots[idx] = nm;
	r->nr_held++;
}                                      /* page_pool이 아닌 페이지는 추적하지 않음 (슬롯은 0) */
```

`nr_held` 갱신 규칙: **슬롯이 0 → 페이지가 될 때만 +1, 페이지 → 0이 될 때만 −1.** 교체는 "−1 후 +1"로 처리한다. 이 규칙을 지키면 항상 `nr_held == (0이 아닌 슬롯 수) ≤ nr_slots`가 성립하고, 이것이 상한 검증 기준이 된다(7.2 E6).

- 한 배치(최대 32페이지)는 항상 **한 skb 안에서만** 모인다([tcp.c:2297-2301](net/ipv4/tcp.c#L2297-L2301), "cannot unroll failed ops across skbs"). 그래서 배치 함수에 `skb->pp_recycle` 하나만 넘기면 된다.
- 이 작업은 `tcp_recv_skb()`가 skb를 해제하는 시점([tcp.c:2331](net/ipv4/tcp.c#L2331))보다 **앞에서** 일어난다. 그래서 skb 해제 때 `pp_ref_count`가 0이 되지 않는다.

**(5) 반환 시점: 슬롯은 "그 범위가 zap되거나 재사용될 때" 반환된다**

정확한 규칙: 슬롯의 pp 참조는 **다음 RECEIVE가 해당 범위를 실제로 zap하거나, 그 슬롯에 새 페이지를 넣을 때** 놓인다. "다음 RECEIVE를 부르기만 하면 반환된다"가 아니다.

| 경우 | 동작 |
|---|---|
| 다음 RECEIVE가 같은 범위를 zap | 기존 코드가 매핑할 범위 `[address, address + total_bytes_to_map)`를 먼저 zap한다 → `tcp_zc_zap()`이 **그 범위의** 슬롯 해제 → 재활용 |
| 사용자가 `madvise(DONTNEED)` 후 다음 RECEIVE | madvise에는 슬롯 반환 훅이 없다. 슬롯은 다음 RECEIVE가 그 범위를 zap할 때 해제된다(zap 자체는 no-op) |
| `TLB_CLEAN_HINT` 사용 (zap 생략) | 삽입이 `-EBUSY`면 zap 경로를 탄다. 성공하면 위 (4)의 "이전 페이지 put" 경로를 탄다 |
| munmap / 프로세스 종료 | `.close` → 남은 슬롯 전부 put. 📌 munmap과 exit 모두 **PTE 제거 + TLB flush가 끝난 뒤** `.close`가 불린다 ([mm/vma.c:1425-1441](mm/vma.c#L1425-L1441), [mm/mmap.c:1315-1328](mm/mmap.c#L1315-L1328)) |

📌 **다음 RECEIVE를 불러도 슬롯이 남는 경우** ([tcp.c:2225-2232](net/ipv4/tcp.c#L2225-L2232), [tcp.c:2240-2245](net/ipv4/tcp.c#L2240-L2245)):

- `inq <= copybuf_len` → `receive_fallback_to_copy()`로 처리되어 **VMA를 찾기 전에** 반환한다.
- `inq < PAGE_SIZE` → 역시 VMA를 찾기 전에 반환한다.
- 새로 받을 길이가 이전보다 짧으면 **그 길이만큼만** zap하므로, 이전 슬롯의 뒷부분은 남는다.
- 데이터를 다 받은 뒤(연결 종료 직전 등)에는 더 이상 zap이 일어나지 않으므로, **마지막 보류 페이지들은 munmap까지 남는다.**

명시적 반환 경로는 선택 사항이다.
- **v1에서 생략할 수 있다.** 이 경우 위 한계를 받아들이고, 보류 페이지는 최대 슬롯 수만큼 munmap 전까지 남는다.
- 즉시 반환이 필요하면 새 데이터 없이 부를 수 있는 경로가 필요하다. 예: `setsockopt(TCP_ZEROCOPY_RELEASE, {address, length})`. 구현은 작다(`find_tcp_vma()` → 소유자 검사 → `tcp_zc_zap()` 재사용). 측정에서 잔류 페이지가 문제로 보이면 추가한다.
- 소켓 종료 경로(`tcp_close`, `tcp_v4_destroy_sock`, `tcp_minisocks.c`)는 **고칠 필요가 없다.** VMA가 파일 참조를 쥐고 있어서([mm/vma.c:514-515](mm/vma.c#L514-L515), `.close` 다음에 `fput`) region이 소켓보다 항상 먼저 정리되기 때문이다.

**(6) 상한과 회계**

- **보장되는 것: region이 직접 보유하는 pp 참조 수(`nr_held`) ≤ 슬롯 수 = VMA 페이지 수.** 슬롯 하나는 페이지 하나만 쥐고, 새 페이지를 넣기 전에 이전 페이지를 반드시 놓는다.
- **보장되지 않는 것: "살아 있는 수신 페이지 수 ≤ 슬롯 수".** `VM_DONTCOPY`는 fork 상속만 막는다. 다른 커널 경로가 페이지 참조를 따로 잡는 것은 막지 못한다.
  - 📌 예: 수신 영역을 `vmsplice()`로 pipe에 넘기면 pipe가 페이지 참조를 쥔다. GUP 검사는 `VM_IO | VM_PFNMAP`만 거부하고 `VM_MIXEDMAP`은 거부하지 않는다([mm/gup.c:1207](mm/gup.c#L1207)).
  - 이후 슬롯을 재사용하면 이전 페이지는 (`_refcount > 1`이라) pool에서 이탈한 채 pipe에 남고, 슬롯에는 새 페이지가 들어간다. 데이터 손상은 없지만, 그 페이지는 이 한도에 포함되지 않는다.
  - 검토 가능한 대안: 새 모드 VMA에 `VM_IO`를 추가하면 GUP 자체가 막힌다. 다만 디버거의 메모리 읽기(`access_process_vm`) 등 다른 동작도 막히므로 부작용 검토가 필요하다.
- **cap은 소켓 전체 기준이어야 한다.** VMA 크기는 사용자가 정하므로 cap이 필요한데, region마다 따로 검사하면 cap 이하 VMA를 여러 개 만들어 우회할 수 있다. 영역 하나의 크기만 제한될 뿐 소켓 전체 보유량은 제한되지 않는다.
  - **v1(권장): 소켓당 활성 region 하나만 허용.** 이미 region이 있으면 두 번째 mmap은 `-EBUSY`로 실패한다. 마지막 kref put에서 해제 표시를 지운다. `.close`는 소켓 락 없이 불리므로 이 표시는 atomic 비트로 다룬다.
  - 일반화: 소켓에 `Σ region.nr_slots` 예약 카운터를 두고, mmap 시 예약(초과면 실패), 마지막 region 해제 시 반환한다.
  - 한도 값 예: `sk_rcvbuf`의 일정 배수 또는 sysctl.
- 이 상한은 **새 모드에만** 적용된다. 기존 모드(옵션 미사용)는 원래대로 VMA 크기만큼 매핑할 수 있고, 이 설계의 범위 밖이다.
- 회계 표현 정정: 매핑된 페이지는 `inc_mm_counter()`로 **프로세스 RSS에 잡힌다**([mm/memory.c:2516](mm/memory.c#L2516)). 빠지는 것은 **소켓 회계(`sk_rmem_alloc`)**다. page_pool은 `__GFP_ACCOUNT` 없이 할당하므로 할당 시점의 memcg 과금도 없는 것으로 보인다(실험으로 확인 필요). 이것은 `FOLL_PIN`(GUP pin)과는 다른, 일반 페이지 참조다.

**(7) 동시성**

| 슬롯을 만지는 곳 | 잡고 있는 락 | 비고 |
|---|---|---|
| RECEIVE (삽입, zap 후 해제) | 소켓 락 + VMA 읽기 락 | 같은 소켓의 RECEIVE끼리는 소켓 락으로 직렬화 |
| `.close` | mmap 락 (munmap 경로에서는 쓰기 락을 **읽기로 downgrade**한 뒤일 수 있음, [mm/vma.c:1419-1420](mm/vma.c#L1419-L1420)) | 안전성 근거는 락 종류가 아니라 **VMA가 이미 트리에서 분리되었다는 것**이다. 분리 전에 기존 per-VMA 읽기 락 보유자(진행 중이던 RECEIVE)와 동기화되고, 분리 후에는 새 RECEIVE가 이 VMA를 찾을 수 없다 |

⚠️ **소유자 검사가 없으면 위 직렬화가 깨진다.** VMA는 소켓 A가 mmap했어도, RECEIVE는 아무 소켓으로나 그 주소를 넘길 수 있다(`find_tcp_vma()`는 소유 소켓을 확인하지 않음). 이 경우 다음이 일어난다.

- 소켓 A와 B의 RECEIVE가 **서로 다른 소켓 락**을 잡는다.
- per-VMA 락은 읽기 락이라 둘이 동시에 잡을 수 있다.
- 그래서 같은 슬롯에 동시에 접근하게 된다. 결과는 pp 참조 이중 put(→ 아직 사용 중인 페이지가 pool로 돌아감 → 메모리 손상), 기록 덮어쓰기(→ 참조 누수), use-after-free다.

**해결:** `find_tcp_vma()` 직후, **슬롯이나 zap에 손대기 전에** 소유자를 검사한다.

```c
struct tcp_zc_region *r = vma->vm_private_data;

if (r && r->owner != sk) {        /* 새 모드 영역을 다른 소켓이 건드림 */
	if (mmap_locked)              /* find_tcp_vma()가 잡은 락을 그대로 푼다 */
		mmap_read_unlock(current->mm);
	else
		vma_end_read(vma);
	return -EINVAL;               /* zap도, 슬롯 접근도 하지 않는다 */
}
```

- 기존 `out:` 레이블([tcp.c:2317](net/ipv4/tcp.c#L2317))로 `goto`하지 않는다. `out:` 뒤의 `else` 분기는 `!zc->recv_skip_hint && SOCK_DONE`이면 `ret`을 `-EIO`로 덮어쓴다([tcp.c:2336-2338](net/ipv4/tcp.c#L2336-L2338)). 이 시점의 `recv_skip_hint`는 아직 사용자가 넘긴 값이라, 오류 코드가 입력에 따라 달라진다.
- 락 해제는 `find_tcp_vma()`가 어느 쪽 락을 잡았는지(`mmap_locked`)에 따라 갈린다. 둘 중 하나를 빠뜨리면 락 누수다.

이렇게 하면 한 region의 슬롯에 접근하는 RECEIVE는 모두 **소유 소켓의 락 하나**로 직렬화된다. 기존 모드(`r == NULL`)는 지금처럼 교차 사용을 허용한다(기존 동작 유지).

**(8) 반드시 지켜야 할 불변식 ⚠️**

0. **새 모드 region의 슬롯은 소유 소켓의 락을 잡은 상태에서만 만진다** (`r->owner == sk` 검사 후). `.close`는 예외다(VMA가 분리되고 기존 접근자와 동기화된 뒤라 다른 접근자가 없음).
1. **사용자 PTE가 남아 있는 페이지의 pp 참조를 놓지 않는다.** 놓는 곳은 세 곳뿐이다: zap 직후, 삽입 성공으로 PTE 부재가 증명된 직후, `.close`(PTE 제거 후). 설령 순서가 틀려도 page_pool의 `_refcount == 1` 검사가 그대로 있으므로 결과는 "재활용 실패"이고 데이터 유출은 아니다. 다만 이 검사에만 기대지 말고 순서 자체를 지킨다.
2. `vm_insert_pages()` 부분 성공 시 실제로 매핑된 페이지(`pages_remaining` 기준)만 슬롯에 기록한다. `-EBUSY` 재시도 성공분도 포함한다.
3. region은 kref로 관리한다. `.close`에서 무조건 해제하면 안 된다(mremap 실패 경로).
4. 기존 모드(`vm_private_data == NULL`)에서는 모든 훅이 no-op이어야 한다. `.may_split`도 0을 돌려줘 기존 동작을 유지한다.

**(9) 알려진 한계**

- **pool 파괴 지연:** 사용자가 VMA를 계속 쥔 채 수신을 멈추면, 슬롯에 남은 페이지(최대 VMA 크기)가 page_pool의 inflight로 남는다. 링 재구성 시 pool 파괴가 그만큼 지연된다. 상한은 **개수**를 제한할 뿐 **기간**은 제한하지 못한다. 필요하면 v2의 `RELEASE` sockopt나 유휴 시 반환 정책으로 보완한다.
- 기존 모드의 회계 문제(P4)는 그대로다. 새 모드를 쓰는 앱에만 상한이 적용된다.
- 명시적 반환 경로가 없으면, 마지막으로 받은 페이지들은 munmap 전까지 슬롯에 남는다((5) 참고).
- 외부 참조(vmsplice 등)로 수명이 연장된 페이지는 region 한도 밖이다((6) 참고).

#### 6.1.2 초기 설계(소켓 xarray)를 폐기한 이유 — 리뷰 반영 내역

| 리뷰 지적 | 확인 결과 | 개정판의 해결 |
|---|---|---|
| 가상주소만으로는 식별 불가 (fork, 주소 재사용, 다른 소켓의 VMA) | 📌 맞음. `VM_MIXEDMAP` VMA는 fork 시 PTE가 복사된다. `find_tcp_vma()`는 `vm_ops`만 확인한다 | 기록을 VMA에 묶고 `VM_DONTCOPY`, 분할·이동 금지 |
| `xa_store` 실패·덮어쓰기 → pp 참조 누수 | 📌 맞음. devmem은 예약 후 성공분만 확정한다([tcp.c:2441-2453](net/ipv4/tcp.c#L2441-L2453)) | mmap 시 고정 배열 할당 → hot path 할당 없음. 덮어쓰기 전에 이전 페이지 put |
| `nr_held`는 전체 메모리 상한이 아님 (fork 반례) | 📌 맞음 | fork 복사 차단 + region 크기 cap → region이 직접 쥔 참조는 슬롯 수 이하 (외부 참조는 3차 리뷰 참고) |
| "어디에도 안 잡힌다"는 과장 | 📌 맞음. RSS에는 반영된다 | 표현 정정: 빠지는 건 소켓 회계 |
| `-k`만으로는 누적 재현 불가 | 📌 맞음. 다음 RECEIVE가 같은 범위를 먼저 zap한다 | 6.2 / 7.2 실험 재설계 |
| 종료 정리 (close해도 VMA가 파일을 쥠, 훅 없음) | 📌 맞음. `tcp_vm_ops`가 비어 있다 | `.close` 훅 하나로 munmap·exit 모두 처리 |
| pool 파괴 지연은 한도로 해결되지 않음 | 맞음 | (9) 알려진 한계로 명시 |
| (2차 리뷰) 다른 소켓이 같은 VMA로 RECEIVE → 슬롯 동시 접근 | 📌 맞음. `find_tcp_vma()`는 소유 소켓을 확인하지 않고, 두 RECEIVE는 서로 다른 소켓 락을 잡는다. 개정판 초안의 "다른 소켓이 같은 슬롯을 만지는 일은 없다"는 서술은 틀렸다 | `region->owner` 기록, RECEIVE 진입 시 불일치면 슬롯·zap 접근 전에 `-EINVAL` |
| (3차) VMA별 cap은 소켓 전체를 제한하지 않음 | 📌 맞음 | v1: 소켓당 활성 region 1개 (또는 Σ 예약) |
| (3차) fork를 막아도 살아 있는 페이지 ≤ 슬롯 수는 아님 (vmsplice 등) | 📌 맞음. GUP는 `VM_MIXEDMAP` 읽기를 거부하지 않는다 | 보장 범위를 "region이 직접 쥔 pp 참조 ≤ 슬롯 수"로 축소, "추가 참조가 없으면 재활용"으로 표현 수정 |
| (3차) 다음 RECEIVE가 슬롯을 반드시 반환하지는 않음 | 📌 맞음. 작은 inq / copy fallback은 VMA 탐색 전에 반환, zap은 새 길이만큼만 | 반환 규칙을 "그 범위가 zap/재사용될 때"로 명확화, RELEASE 경로는 선택 사항으로 명시 |
| (3차) E6의 pool inflight 상한은 잘못된 기준 | 📌 맞음. inflight에는 RX 링·수신 큐·OFO 큐 페이지도 포함 | 검증 기준을 `nr_held ≤ nr_slots`로 변경, `nr_held` 갱신 규칙 추가 |
| (3차) 실패율 계산, `.close` 락 설명, SHA256 검증의 한계 | 📌 맞음. munmap은 락을 읽기로 downgrade한다 | 6.0 측정 주의사항, (7) 표, 7.3 테스트 수정 |
| (4차) 개념 코드 구조체에 `owner` 필드 누락 | 맞음 | `struct sock *owner` 추가, `tcp_mmap()`에서 `owner = sk` |
| (4차 검토 중 자체 발견) 소유자 검사 코드가 존재하지 않는 레이블로 `goto` | 📌 실제 함수에는 `out_unlock_vma`가 없다. 기존 `out:`으로 보내면 `ret`이 `-EIO`로 덮어써질 수 있다 | 검사 지점에서 락을 직접 풀고 즉시 반환 |

#### 6.1.3 초기 설계 대비 복잡도

| 항목 | 초기 설계 A | 개정판 A′ |
|---|---|---|
| 기록 구조 | 소켓마다 xarray (키 = 가상주소) | VMA마다 고정 배열 (키 = 페이지 인덱스) |
| 수신 경로 메모리 할당 | 있음 (실패 처리 필요) | **없음** |
| 새 uAPI | 플래그 + RELEASE sockopt + 구조체 | **활성화 sockopt 1개** |
| 정리 지점 | `tcp_close` / `tcp_disconnect` / `destroy_sock` / `minisocks` 초기화 | **`.close` 한 곳** |
| 상한 | 별도 카운터 + 검사 | 배열 크기 자체 (mmap 시 cap) |
| 새로 필요한 것 | — | `vm_ops` 훅 4개 + kref |
| tcp_mmap.c | `madvise` → RELEASE 교체 | **mmap 전에 setsockopt 한 줄** (`madvise`는 그대로 둬도 됨) |
| 수정 파일 수 | 6개 | 3개 (`uapi/tcp.h`, `linux/tcp.h`, `tcp.c`) + tcp_mmap.c |

### 6.2 `tools/testing/selftests/net/tcp_mmap.c` 수정 방안 💡

현재 수신 루프 ([tcp_mmap.c:217-258](tools/testing/selftests/net/tcp_mmap.c#L217-L258)):

```
poll → getsockopt(ZC) → (해시/SHA) → madvise(addr, len, MADV_DONTNEED) → recv_skip_hint만큼 read()
```

| 항목 | 현재 | 제안 |
|---|---|---|
| 새 모드 활성화 | 없음 | `-R` 옵션: accept한 소켓에 `setsockopt(TCP_ZEROCOPY_RECYCLE)` 후 mmap. **A′의 최소 변경은 이 한 줄이다.** `madvise`는 그대로 둬도 다음 RECEIVE에서 재활용된다 |
| 수신 슬롯 | 매핑 영역 1개(`chunk_size + map_align`), 같은 주소 반복 사용 | `-N <n>`: `n × chunk_size` 영역을 만들고 매 RECEIVE마다 **다음 슬롯 주소**로 순환. "반환 안 된 청크 여러 개"를 만들 수 있다 |
| `-k` (keepflag) | 📌 **파싱만 되고 사용되지 않음** (선언 88행, 설정 494행) | 구현: `madvise` 생략. ⚠️ 단독으로는 효과가 없다. 같은 주소를 다시 RECEIVE하면 커널이 먼저 zap하기 때문이다([tcp.c:2245](net/ipv4/tcp.c#L2245)). **`-N`과 함께 써야** 최대 n개 청크가 동시에 보유된다 |
| 소비자 지연 재현 | 없음 | `-D <usec>`: 처리 후 다음 RECEIVE까지 지연 → 타이밍 의존성 재현 |
| 작은 데이터 | `read()` 추가 syscall | `copybuf_address/copybuf_len` 사용 옵션 |
| 결과 출력 | 처리량/CPU/% mmap'ed | `zc.inq`, `zc.err`, (선택) 실행 전후 page_pool 통계를 외부 스크립트로 diff |
| 에러 경로 | `res == -1` 시 break → 매핑 남은 채 종료까지 대기 | 새 모드에서는 munmap 시 `.close`가 정리하므로 추가 처리 불필요 |

⚠️ selftest는 루프백(`lo`)으로도 돌릴 수 있지만 **루프백은 page_pool을 쓰지 않으므로** 재활용 문제 자체는 재현되지 않는다. 실제 NIC(header split 지원: mlx5, bnxt, gve, fbnic, idpf 등) 필요.

---

## 7. 검증 계획

### 7.1 환경

```bash
ethtool -G eth0 tcp-data-split on          # header split
ip link set eth0 mtu 4168                   # 4096 + 40(IPv6) + 32(TCP+TS)  (IPv4면 4096+20+32)
# 또는 HW-GRO 사용
sysctl -w net.ipv4.tcp_rmem="4096 131072 67108864"
```

### 7.2 실험 매트릭스

| # | 수신 모드 | 옵션 | 측정 / 기대 |
|---|---|---|---|
| E1 | 복사(`-z` 없음) | — | 기준선: `released_refcnt` ≈ 0 |
| E2 | 기존 ZC | `-z` | `released_refcnt` 증가량 vs mmap된 페이지 수 → **재활용 실패 비율 측정** (문제 정량화) |
| E3 | 기존 ZC, 보유 누적 | `-z -k -N <큰 n>` | RSS가 `n × chunk`까지 증가, 그동안 `sk_rmem_alloc`은 낮게 유지 (P4 재현) |
| E4 | 기존 ZC, 느린 소비자 | `-z -D <usec>` | slow alloc / 드롭 변화 |
| E5 | 새 모드 | `-z -R` | `released_refcnt` ↓, `recycle.cached/ring` ↑, `alloc.slow` ↓ |
| E6 | 새 모드, 보유 상한 | `-z -R -k -N <n>` | **`region.nr_held ≤ region.nr_slots`** 유지 (디버그 카운터 또는 tracepoint로 확인). 같은 소켓에 두 번째 region mmap 시 `-EBUSY`, cap 초과 mmap 시 실패 확인. pool inflight는 상한 기준이 아니라 **추세만** 관찰한다(RX 링·수신 큐·OFO 큐 페이지가 섞여 있음) |
| E7 | 새 모드 + 패킷 손실 | `-z -R` + `tc netem loss` | OFO 큐 영향 분리 |

지표: 처리량(Gbit), CPU usec/MB, `% mmap'ed`, page_pool alloc/recycle 통계, 드라이버 rx drop/alloc_fail, RSS / `ss -tm`, IOMMU on/off 비교.

### 7.3 정합성 테스트

- `-i` (SHA256 무결성)는 기본으로 켠다. 다만 **통과가 "조기 재활용이 없다"는 증명은 아니다.** 해시를 계산한 **뒤에** 페이지가 덮어써지면 놓친다. 조기 재활용을 잡으려면 다음을 추가한다.
  - 수신 직후 청크 해시를 계산하고, `-D`로 지연한 뒤 **슬롯이 반환되기 직전(다음 RECEIVE 직전)에 같은 영역을 다시 해시해** 비교한다. 다르면 매핑 중에 덮어써진 것이다.
  - 커널 쪽에서는 `tcp_zc_zap()`/슬롯 교체 시점에 "이 페이지가 이 VMA에 아직 매핑돼 있지 않은지"를 디버그 빌드에서 `WARN_ON`으로 검사한다.
- 소켓 A로 만든 새 모드 영역의 주소를 소켓 B의 RECEIVE에 넘기면 `-EINVAL`이 나오는지, A의 슬롯·매핑이 그대로인지 확인. A·B 두 스레드로 동시에 반복해 KASAN/`page_pool` inflight 이상이 없는지 확인.
- 새 모드 VMA에 대해 다음이 **거부되는지** 확인: 부분 `munmap`, `mprotect`로 분할, `mremap` 이동/확장. 그리고 `fork()` 후 자식에서 해당 영역 접근 시 매핑이 없는지 확인.
- munmap / 프로세스 kill / 소켓 close 후 munmap 순서별로: page_pool inflight가 원래대로 돌아오는지, `kmemleak` 경고가 없는지.
- mremap 실패 경로 반복 → KASAN에서 region use-after-free가 없는지 (kref 검증).
- 수신 중 `ip link set down` → "stalled pool shutdown" 경고가 보유 페이지 반환 후 사라지는지.
- `CONFIG_DEBUG_VM`, `CONFIG_DEBUG_NET`, KASAN 빌드에서 반복 실행.

---

## 8. 리스크 체크리스트

- [ ] **새 모드 region에 소유 소켓이 아닌 소켓이 RECEIVE하면 슬롯·zap 접근 전에 거부하는가 (동시성, 최우선)**
- [ ] 소유자 불일치로 거부할 때 `find_tcp_vma()`가 잡은 락(per-VMA 읽기 락 또는 mmap 읽기 락)을 정확히 하나 푸는가
- [ ] `tcp_mmap()`에서 `owner = sk`를 region 공개(`vm_private_data` 설정) 전에 초기화하는가
- [ ] 사용자 PTE가 남아 있는 페이지의 pp 참조를 놓는 경로가 없는가 (정보 유출)
- [ ] 슬롯 덮어쓰기 전에 이전 페이지를 반드시 put하는가 (누수)
- [ ] `-EBUSY` 재시도 성공분도 슬롯에 기록하는가
- [ ] 부분 매핑(`pages_remaining > 0`) 시 실제로 매핑된 페이지에만 pp 참조를 잡고, 매핑 안 된 페이지에는 잡지 않는가 (참조 누수)
- [ ] region kref: `.open`/`.close` 짝, mremap 실패 경로 (use-after-free)
- [ ] 슬롯 정리·region 해제·"활성 region" 비트 해제가 **마지막 kref put에서만** 일어나는가
- [ ] 새 모드 VMA의 fork 상속·분할·이동이 모두 막히는가
- [ ] cap이 **소켓 전체** 기준으로 적용되는가 (v1: 소켓당 region 1개)
- [ ] `nr_held` 갱신 규칙(0→페이지 +1, 페이지→0 −1)이 모든 경로에서 지켜지는가
- [ ] 새 옵션 미사용 시 기존 동작·성능이 동일한가 (훅이 모두 no-op)
- [ ] page_pool이 아닌 페이지(드라이버 자체 할당, 루프백)는 슬롯에 기록하지 않는가 (`skb->pp_recycle && netmem_is_pp()`)
- [ ] devmem(net_iov) frag와 섞여도 문제 없는가 (`skb_frags_readable`)
- [ ] pool 파괴 지연은 "알려진 한계"로 문서화했는가

---

## 9. 관련 코드 위치 모음

| 파일 | 함수/심볼 | 역할 |
|---|---|---|
| [net/ipv4/tcp.c:1867](net/ipv4/tcp.c#L1867) | `tcp_mmap` | 소켓 mmap: RO, `VM_MIXEDMAP`, `tcp_vm_ops` |
| [net/ipv4/tcp.c:1905](net/ipv4/tcp.c#L1905) | `can_map_frag` | 매핑 가능 조건 |
| [net/ipv4/tcp.c:2110](net/ipv4/tcp.c#L2110) | `tcp_zerocopy_vm_insert_batch` | 페이지 매핑 (방안 A의 pp ref 삽입 지점) |
| [net/ipv4/tcp.c:2195](net/ipv4/tcp.c#L2195) | `tcp_zerocopy_receive` | 본체 |
| [net/ipv4/tcp.c:1614](net/ipv4/tcp.c#L1614) | `tcp_eat_recv_skb` | skb 해제 + rmem 회계 |
| [net/ipv4/tcp.c:4753](net/ipv4/tcp.c#L4753) | `do_tcp_getsockopt` ZC 분기 | uAPI 진입 |
| [include/uapi/linux/tcp.h:509](include/uapi/linux/tcp.h#L509) | `struct tcp_zerocopy_receive`, `TCP_RECEIVE_ZEROCOPY_FLAG_*` | uAPI 정의 |
| [mm/memory.c:2464](mm/memory.c#L2464) | `validate_page_before_insert`, `insert_page_into_pte_locked` | `folio_get` 으로 refcount +1 |
| [net/core/page_pool.c:826](net/core/page_pool.c#L826) | `__page_pool_page_can_be_recycled` | `_refcount==1` 조건 |
| [net/core/page_pool.c:766](net/core/page_pool.c#L766) | `page_pool_return_netmem` | pool 이탈 |
| [include/net/page_pool/helpers.h:265](include/net/page_pool/helpers.h#L265) | `page_pool_unref_netmem`, `page_pool_ref_netmem`(315) | pp_ref_count 조작 |
| [net/core/skbuff.c:1029](net/core/skbuff.c#L1029) | `napi_pp_put_page` | skb frag → pool 반환 |
| [net/core/skbuff.c:7358](net/core/skbuff.c#L7358) | `skb_attempt_defer_free` | 원격 CPU 지연 해제 |
| [include/linux/skbuff.h:886](include/linux/skbuff.h#L886) | `struct sk_buff` | |
| [include/linux/skbuff.h:593](include/linux/skbuff.h#L593) | `struct skb_shared_info` | |
| [net/core/sock.c:1087](net/core/sock.c#L1087) | `sock_devmem_dontneed` | devmem의 사용자 반환 API (설계 참고) |
| [io_uring/zcrx.c](io_uring/zcrx.c) | io_uring zero-copy RX | refill queue 기반 반환 (설계 참고) |
| [tools/testing/selftests/net/tcp_mmap.c](tools/testing/selftests/net/tcp_mmap.c) | `child_thread` | 수신 측 레퍼런스 구현 |
| [mm/memory.c:1564](mm/memory.c#L1564) | `vma_needs_copy` | fork 시 PTE 복사 여부 (`VM_COPY_ON_FORK`) |
| [mm/vma.c:510](mm/vma.c#L510) | `remove_vma` | `.close` → `fput` 순서 |
| [mm/vma.c:1410](mm/vma.c#L1410) | `vms_complete_munmap_vmas` | PTE 제거 후 `remove_vma` |
| [mm/mremap.c:1295](mm/mremap.c#L1295) | `.mremap` 훅 호출 / 실패 시 되돌리기 | |
| [include/linux/mm.h:806](include/linux/mm.h#L806) | `struct vm_operations_struct` | `open`, `close`, `may_split`, `mremap` |

---

## 10. 다음 단계와 수정 순서

### 10.1 진행 단계

1. **6.0 측정**: E1/E2로 재활용 실패 비율을 먼저 확보한다 (코드 무수정).
2. `tcp_mmap.c`에 `-N`, `-k`, `-D` 옵션만 먼저 추가 → E3/E4로 문제를 정량화한다.
3. 방안 A′를 구현 → E5~E7, 7.3 정합성 테스트.
4. 결과를 devmem TCP / io_uring zcrx와 비교해, 일반 호스트 메모리 경로에서 이 방식이 갖는 이점(기존 앱 호환, dmabuf 불필요)을 정리한다.

### 10.2 A′ 코드 수정 순서

| 순서 | 파일 | 위치 | 할 일 |
|---|---|---|---|
| 1 | [include/uapi/linux/tcp.h](include/uapi/linux/tcp.h#L144) | `TCP_DELACK_MAX_US 46` 다음 | `#define TCP_ZEROCOPY_RECYCLE 47` |
| 2 | `include/linux/tcp.h` | `struct tcp_sock` | 활성화 비트 `zc_recycle:1` (빈 비트필드 자리 활용), "활성 region 있음" atomic 비트 (소켓당 region 1개 제한용) |
| 3 | [net/ipv4/tcp.c](net/ipv4/tcp.c#L3836) | `do_tcp_setsockopt()` / `do_tcp_getsockopt()` | `case TCP_ZEROCOPY_RECYCLE:` 비트 설정·조회 |
| 4 | [net/ipv4/tcp.c](net/ipv4/tcp.c#L1863) | `tcp_vm_ops`, `tcp_mmap()` | `struct tcp_zc_region`, kref, 훅 4개(`open`/`close`/`may_split`/`mremap`), 비트가 켜져 있으면 활성 region 검사(`-EBUSY`) + cap 검사 + region 할당(`owner = sk`) + `VM_DONTCOPY \| VM_DONTEXPAND` |
| 5 | [net/ipv4/tcp.c](net/ipv4/tcp.c#L2062) | `tcp_zerocopy_vm_insert_batch*()`, `tcp_zerocopy_receive()` | **`find_tcp_vma()` 직후 `region->owner == sk` 검사 (최우선)**, `zap_vma_range()` 두 곳을 `tcp_zc_zap()`으로 교체, 삽입 성공분 슬롯 기록 (`skb->pp_recycle` 전달) |
| 6 | [tools/testing/selftests/net/tcp_mmap.c](tools/testing/selftests/net/tcp_mmap.c) | `child_thread()` | `-R`, `-N`, `-k`, `-D` 옵션 |

초기 설계와 달리 `tcp_ipv4.c`(소켓 파괴), `tcp_minisocks.c`(accept 소켓 초기화)는 수정하지 않는다. 기록이 소켓이 아니라 VMA에 있기 때문이다.

브랜치: 원본은 `zerocopy` 브랜치에 두고, 작업은 별도 브랜치(예: `zc-defer-release`)에서 한다.
