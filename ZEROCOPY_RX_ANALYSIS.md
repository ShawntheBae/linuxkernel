# TCP 제로카피 수신(TCP_ZEROCOPY_RECEIVE)과 page_pool 고갈 문제 분석

> 기준 소스: Linux **7.3.0-rc5** (이 저장소, 브랜치 `zerocopy`)
> 상태: **분석/설계 문서**. 코드는 아직 수정하지 않았음.
> 표기: 📌 = 코드로 확인한 사실, 💡 = 제안/설계, ⚠️ = 주의/리스크

---

## 0. 요약 (TL;DR)

1. 📌 제로카피 수신은 `vm_insert_pages()`로 skb의 frag 페이지를 사용자 VMA에 그대로 매핑한다. 이때 **page `_refcount`가 +1** 된다 (`folio_get()`, [mm/memory.c:2510](mm/memory.c#L2510)).
2. 📌 그런데 같은 `getsockopt()` 호출 안에서, 매핑 직후 `tcp_recv_skb()` → `tcp_eat_recv_skb()`로 **skb가 곧바로 해제**된다 ([net/ipv4/tcp.c:2328-2331](net/ipv4/tcp.c#L2328-L2331)).
3. 📌 skb 해제 시 page_pool은 **`page_ref_count(page) == 1`일 때만** 페이지를 재활용한다 ([net/core/page_pool.c:826-831](net/core/page_pool.c#L826-L831)). 사용자가 아직 매핑을 들고 있으므로 refcount는 2 → 재활용 실패 → `page_pool_return_netmem()`으로 **페이지가 pool에서 영구 이탈**(DMA unmap 후 일반 페이지가 됨).
4. 결과적으로 **제로카피로 매핑된 페이지는 사실상 100% page_pool로 돌아오지 못한다.** pool은 매번 새 페이지를 할당(+DMA map)해야 하고, 고대역폭에서 할당 지연/실패 → RX 링 리필 실패 → 드롭으로 이어진다. 이것이 사용자가 말한 "타이밍 때문에 반환이 제대로 안 되는" 현상의 정체다.
   - 핵심은 **순서 문제**: `skb 해제(pp put)` 가 `사용자 unmap` 보다 **먼저** 일어나면 재활용 실패, **나중에** 일어나면 재활용 성공.
5. 📌 추가로, skb가 해제되면 소켓 메모리 회계(`sk_rmem_alloc`)도 줄어들기 때문에 **사용자가 들고 있는 페이지는 어떤 소켓 한도에도 잡히지 않는다** → 느린/악의적 소비자가 무제한으로 페이지를 고정(pin)할 수 있다.
6. 💡 해결 방향(권장): **"지연된 pp 참조 해제(deferred pp release)"** — 매핑한 페이지마다 page_pool 참조(`pp_ref_count`)를 하나 더 잡아 두고, 사용자가 명시적으로 반환(커널이 zap + TLB flush 완료)한 뒤에 그 참조를 놓는다. 그러면 마지막 pp put 시점에 `_refcount == 1`이 되어 재활용된다. 동시에 미반환 페이지 수를 소켓 단위로 제한한다.
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

현재 코드 구조상 t1(skb 해제)이 사용자 unmap보다 항상 앞선다. `skb_attempt_defer_free()`로 다른 CPU에서 늦게 해제되는 경우 드물게 B가 될 수 있지만 우연에 의존한다 — 이것이 "타이밍적인 오류"로 관찰되는 비결정성의 원인일 가능성이 높다.

### 4.2 파생 문제

| # | 문제 | 메커니즘 |
|---|---|---|
| P1 | pool 재활용률 ≈ 0 | 위 Case A |
| P2 | slow path 할당 + DMA map/unmap 폭증 | 매 페이지 `page_pool_return_netmem` + 새 `alloc_pages` |
| P3 | RX 링 리필 실패 → 드롭 | softirq의 `GFP_ATOMIC` 할당 실패, 메모리 압박 |
| P4 | **회계 우회 / 무제한 pin** | skb 해제 시 `sock_rfree`로 `sk_rmem_alloc` 감소 → 윈도우가 다시 열려 송신측이 계속 보냄 → 사용자가 unmap하지 않으면 메모리가 계속 쌓임. memcg/tcp_mem 어디에도 안 잡힘 |
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

기대 결과: 제로카피 모드에서 `recycle.released_refcnt ≈ mmap된 페이지 수`, `alloc.slow` 급증.

### 6.1 `net/ipv4/tcp.c` 수정 방안 비교

| 방안 | 개요 | 장점 | 단점 |
|---|---|---|---|
| **A. 지연 pp 참조 해제 (권장)** | 매핑하는 페이지마다 `page_pool_ref_netmem()`으로 pp 참조를 +1 해서 소켓에 기록. skb는 지금처럼 해제. 사용자가 반환하면 커널이 zap 후 pp 참조 -1 | skb/수신 큐 로직 거의 그대로, 재활용 순서 보장, 페이지 단위 정밀 제어 | 소켓별 추적 구조 필요, 새 uAPI |
| B. skb 자체를 보류 | 다 읽은 skb를 해제하지 않고 "zc pending" 리스트로 옮겼다가 반환 시 해제 | 회계(sk_rmem_alloc)가 자연스럽게 유지돼 P4도 해결 | 윈도우가 안 열려 처리량 저하, skb 단위라 부분 반환 어려움, 메모리 낭비(헤더 등) |
| C. 회계/제한만 추가 | 미반환 페이지 수를 세어 한도 초과 시 복사로 폴백 | 단순 | 재활용 문제(P1~P3)는 해결 안 됨 |
| D. 기존 메커니즘 사용 | devmem TCP 또는 io_uring zcrx | 업스트림에서 이미 이 문제를 설계적으로 해결 | dmabuf/io_uring 기반이라 API가 완전히 다름 |

#### 6.1.1 방안 A 상세 설계 💡

**(1) 자료구조**

devmem TCP가 이미 `sk->sk_user_frags` (xarray, [include/net/sock.h:600](include/net/sock.h#L600))로 "사용자가 들고 있는 frag"를 추적한다 ([net/ipv4/tcp.c:2448](net/ipv4/tcp.c#L2448) 부근 참고). 같은 패턴으로 소켓에 다음을 둔다.

```c
/* 개념 코드 — 실제 필드 위치/이름은 구현 시 결정 */
struct tcp_zc_held {
	struct xarray	pages;      /* key: user vaddr >> PAGE_SHIFT, value: netmem */
	u32		nr_held;     /* 현재 보류 중인 페이지 수 */
	u32		limit;       /* 상한 (예: sk_rcvbuf / PAGE_SIZE) */
};
```

- 키를 **사용자 가상주소**로 잡으면 "이 범위를 zap했으니 이 범위의 pp 참조를 놓는다"가 정확해진다.
- `struct tcp_sock`에 직접 넣기보다 첫 사용 시 lazily 할당하는 포인터가 tcp_sock 크기/캐시라인 영향이 적다.

**(2) 매핑 시 (tcp_zerocopy_receive 내부)**

```c
/* tcp_zerocopy_vm_insert_batch() 에서 vm_insert_pages() 성공한 페이지들에 대해 */
if (zc->flags & TCP_RECEIVE_ZEROCOPY_FLAG_DEFER_RELEASE) {
	netmem_ref nm = page_to_netmem(pages[i]);
	if (netmem_is_pp(nm)) {
		page_pool_ref_netmem(nm);        /* pp_ref_count +1 */
		xa_store(&held->pages, vaddr >> PAGE_SHIFT, (void *)nm, GFP_KERNEL);
		held->nr_held++;
	}
}
```

- 한도 체크: 매핑 **전에** `nr_held + pages_to_map > limit`이면 매핑 길이를 줄이고 `recv_skip_hint`로 복사 경로 유도 → P4(무제한 pin) 해결.
- 이후 `tcp_recv_skb()`가 skb를 해제해도 pp_ref_count가 남아 있으므로 `__page_pool_put_page()` 판단이 **보류**된다.

**(3) 반환 시 (새 진입점)**

두 가지 경로 중 택1 (또는 둘 다):

- (a) 다음 `TCP_ZEROCOPY_RECEIVE` 호출의 `zap_vma_range(vma, address, total_bytes_to_map)` 직후, 그 범위의 보류 페이지를 해제.
- (b) **명시적 반환 sockopt** 신설 — 예: `setsockopt(TCP_ZEROCOPY_RELEASE, {address, length})`
  1. `find_tcp_vma()` → `zap_vma_range()` (내부에서 TLB flush 완료까지 보장)
  2. 범위 내 xarray 항목마다 `page_pool_put_full_netmem(netmem_get_pp(nm), nm, false)` → 이 시점에 `_refcount==1` → **재활용 ✅**

(b)를 권장: 사용자가 "처리 끝" 시점을 정확히 알려주고, 커널이 zap까지 직접 하므로 순서가 보장된다. 사용자가 이미 `madvise(DONTNEED)`를 했더라도 zap은 no-op이고 결과는 같다.

**(4) 정리 경로 (누수 방지)**

- `tcp_disconnect()` / `tcp_close()` / `sk_destruct` 에서 남은 보류 페이지 전부 pp put.
  - 사용자가 아직 매핑을 들고 있어도 안전: `_refcount>1`이면 page_pool이 그냥 이탈시키고(`released_refcnt`), 페이지는 사용자 unmap 시 정상 free.
- ⚠️ 보류 중인 페이지는 pool의 inflight로 잡혀 있으므로, 소켓이 오래 살아 있으면 **드라이버 링 재구성 시 pool 파괴가 지연**된다 → 한도(limit)가 반드시 필요한 이유.

**(5) 반드시 지켜야 할 불변식 ⚠️**

1. **사용자 PTE가 남아 있는 페이지를 pool로 돌려보내면 절대 안 된다.** NIC가 그 페이지에 다음 패킷을 DMA로 쓰면 사용자 프로세스가 **다른 연결의 데이터를 보게 되는 정보 유출**이 된다. → pp put은 반드시 zap(+TLB flush) 이후. 기존 코드가 안전한 이유는 `_refcount==1` 검사 덕분이며, 방안 A도 이 검사를 우회하지 않는다(검사는 그대로 수행됨).
2. `TCP_RECEIVE_ZEROCOPY_FLAG_TLB_CLEAN_HINT` 경로는 이전 매핑을 zap하지 않으므로, 이 플래그와 함께 쓸 때 반환 경로가 zap을 대신해야 한다.
3. `vm_insert_pages()` 부분 성공 시 실제로 매핑된 페이지에만 pp 참조를 잡을 것 (`pages_remaining` 기준).
4. 동시성: `do_tcp_getsockopt`는 `sockopt_lock_sock(sk)`를 잡는다. 반환 sockopt도 같은 락 하에서 xarray를 조작. 단, `zap_vma_range()`는 mmap/VMA 락이 필요하므로 **소켓 락과 mmap 락의 순서**를 기존 `tcp_zerocopy_receive`와 동일하게(소켓 락 → VMA 락) 유지.
5. uAPI 호환: 새 플래그가 없으면 기존 동작 100% 유지. 📌 현재 `do_tcp_getsockopt`는 `zc.reserved`와 `zc.msg_flags`만 검사하고 **`zc.flags`의 미정의 비트는 검사하지 않는다** ([net/ipv4/tcp.c:4772-4775](net/ipv4/tcp.c#L4772-L4775)). 즉 구버전 커널은 새 플래그를 조용히 무시하므로, 사용자 공간이 기능 지원 여부를 알 수 있는 방법(예: 별도 sockopt 존재 여부, 또는 반환 필드)을 함께 설계해야 한다.

**(6) 회계(P4) 처리 선택지**

- 보류 페이지 수 × PAGE_SIZE를 `sk_rmem_alloc`에 계속 반영(= 윈도우를 닫아 송신측 압박) vs 별도 한도만 두고 윈도우는 열기.
- 전자는 안전하지만 처리량이 줄 수 있음. 실험 항목으로 둔다.

### 6.2 `tools/testing/selftests/net/tcp_mmap.c` 수정 방안 💡

현재 수신 루프 ([tcp_mmap.c:217-258](tools/testing/selftests/net/tcp_mmap.c#L217-L258)):

```
poll → getsockopt(ZC) → (해시/SHA) → madvise(addr, len, MADV_DONTNEED) → recv_skip_hint만큼 read()
```

| 항목 | 현재 | 제안 |
|---|---|---|
| 반환 방식 | `madvise(MADV_DONTNEED)` — 이미 skb가 해제된 뒤라 재활용에 무의미 | `-R` 옵션: `TCP_RECEIVE_ZEROCOPY_FLAG_DEFER_RELEASE` + `TCP_ZEROCOPY_RELEASE` 사용 |
| `-k` (keepflag) | 📌 **파싱만 되고 사용되지 않음** (선언 88행, 설정 494행) | 구현: madvise/반환 생략 → P4(무제한 pin) 재현용 |
| 소비자 지연 재현 | 없음 | `-D <usec>`: 처리 후 반환까지 지연 → 타이밍 의존성 재현 |
| 버퍼 개수 | 매핑 영역 1개 (`chunk_size + map_align`) | `-N <n>`: n개 슬롯 링으로 돌려 "보류 중 + 처리 중" 파이프라인 재현 |
| 작은 데이터 | `read()` 추가 syscall | `copybuf_address/copybuf_len` 사용 옵션 |
| 결과 출력 | 처리량/CPU/% mmap'ed | `zc.inq`, `zc.err`, 반환 호출 수, (선택) 실행 전후 page_pool 통계를 외부 스크립트로 diff |
| 에러 경로 | `res == -1` 시 break → 매핑 남은 채 종료까지 대기 | break 전에 반환 호출 |

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

| # | 수신 모드 | 소비자 | 측정 |
|---|---|---|---|
| E1 | 복사(`-z` 없음) | 즉시 | 기준선: released_refcnt ≈ 0 |
| E2 | 기존 ZC + madvise | 즉시 | released_refcnt ≈ mmap 페이지 수 (문제 재현) |
| E3 | 기존 ZC + `-k` | 반환 안 함 | 메모리 단조 증가 (P4) |
| E4 | 기존 ZC + `-D` | 지연 | 드롭/slow alloc 증가 |
| E5 | 신규 ZC + RELEASE | 즉시 | released_refcnt ↓, recycle.cached/ring ↑ |
| E6 | 신규 ZC + RELEASE + `-D` | 지연 | 한도 도달 시 복사 폴백, 메모리 상한 유지 |
| E7 | 신규 ZC + 패킷 손실 주입(`tc netem loss`) | 즉시 | OFO 큐 영향 분리 |

지표: 처리량(Gbit), CPU usec/MB, `% mmap'ed`, page_pool alloc/recycle 통계, 드라이버 rx drop/alloc_fail, `MemFree`/`ss -tm`, IOMMU on/off 비교.

### 7.3 정합성 테스트

- `-i` (SHA256 무결성) 필수 — 재활용된 페이지가 사용자 매핑 중에 덮어써지면 즉시 깨짐 (불변식 1 검증).
- 소켓 close/RST/프로세스 kill 중 보류 페이지가 있는 상황 → `kmemleak`, page_pool inflight가 0으로 돌아오는지, `ip link set down` 시 "stalled pool shutdown" 경고 없는지.
- `CONFIG_DEBUG_VM`, `CONFIG_DEBUG_NET`, KASAN 빌드에서 반복 실행.

---

## 8. 리스크 체크리스트

- [ ] 사용자 매핑 중인 페이지가 pool로 돌아가는 경로가 없는가 (정보 유출)
- [ ] 보류 페이지 상한이 있는가 (DoS / pool 파괴 지연)
- [ ] 소켓 종료/연결 리셋/fork(VMA 복제 — tcp VMA는 `VM_MIXEDMAP`) 시 누수 없는가
- [ ] 새 플래그 미사용 시 기존 동작·성능이 동일한가
- [ ] page_pool이 아닌 페이지(드라이버 자체 할당, 루프백)에서도 안전한가 (`netmem_is_pp()` 분기)
- [ ] devmem(net_iov) frag와 섞여도 문제 없는가 (`skb_frags_readable`)
- [ ] 32bit/compat getsockopt 구조체 크기 처리 (`offsetofend` switch, [net/ipv4/tcp.c:4781](net/ipv4/tcp.c#L4781))

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

---

## 10. 다음 단계 제안

1. **6.0 측정**으로 E1/E2 수치를 먼저 확보 (코드 무수정).
2. `tcp_mmap.c`에 `-k`, `-D`, `-N` 옵션만 먼저 추가 → E3/E4로 문제를 정량화.
3. 방안 A를 플래그 뒤에 구현 → E5~E7.
4. 결과를 devmem TCP / io_uring zcrx와 비교해, 일반 호스트 메모리 경로에서 이 방식이 갖는 이점(기존 앱 호환, dmabuf 불필요)을 정리.
