# 1. 테스트 시나리오 · 결과 · 분석

> **한눈에:** Client IP·호출 추적·failover(라우팅)는 A/B/C **공통 성립**. **갈림은 세션 하나** — mod_jk(C)·Redis(B)는 유지, ILB 단독(A)은 유실. **failover 시 세션 생존은 Redis(B)만.**


| 테스트 | 구성 | 확인 |
|--------|------|------|
| **A** | mod_proxy + L4 ILB, Redis 無 | L4 ILB 순정 한계 |
| **B** | mod_proxy + L4 ILB, Redis 有 | 세션 유실 해소 여부 |
| **C** | **mod_jk(AJP)** [고객 현행] | Redis 없이 세션 유지 여부 |

**측정 4항목**: ① Client IP 보존 ② 호출 추적(고유 ID) ③ 트래픽 분산 ④ 세션 유지. 환경 상세는 아래 [테스트 환경](#테스트-환경). 측정값 원본은 [문서 2](02-raw-test-data.md)에 있다.

---

## 테스트 환경 

| 계층 | 구성 | IP | 포트 |
|------|------|-----|------|
| 진입 | Application Gateway v2 (L7, 공인 IP) | `snet-agw` 10.10.1.0/24 | 80 |
| 웹 | Apache 2대 (Debian) | 10.10.2.4 / 10.10.2.5 | 80 |
| 내부 LB | Standard Internal LB (L4, 5-tuple) — **A·B만** | VIP 10.10.3.100 | 80 |
| WAS | Tomcat 9 (Debian) 2대 | 10.10.3.4 / 10.10.3.5 | HTTP 8080 / AJP 8009 |
| 세션 | Redis 단일 VM — **B만** | 10.10.3.50 | 6379 |

- **SW**: Korea Central / Standard_D2s_v5 / Apache `mod_proxy`·`mod_proxy_http`(A·B)·`libapache2-mod-jk`(C) / Tomcat 9 + `RemoteIpValve` / 세션매니저 Redisson 3.27.2(B).
- **앱**: 진단용 `index.jsp` — hostname·remoteAddr·요청헤더·sessionId·세션카운터(count) 출력.
- **검증**: `az vm run-command` 로 curl을 반복해 응답의 hostname/sessionId/count를 집계.
- 리소스는 검증 후 삭제, cloud-init(YAML)으로 재현 가능.

**공통 토폴로지**
```
 사용자(공인 IP)
      │
      ▼
 ┌─────────────┐  snet-agw 10.10.1.0/24
 │ AGW (L7)    │  XFF 삽입
 └──────┬──────┘
        ▼
 Apache1 10.10.2.4 : Apache2 10.10.2.5   snet-web 10.10.2.0/24 (:80)
        │
        ├─ (A·B) mod_proxy_http ─▶ L4 ILB VIP 10.10.3.100 (:80) ─┐ 5-tuple
        └─ (C)  mod_jk / AJP :8009 ───────────────────────────────┤
                                                                   ▼
 Tomcat1 10.10.3.4 : Tomcat2 10.10.3.5   snet-was 10.10.3.0/24 (HTTP :8080 / AJP :8009)
        │ (B만) 세션 R/W
        ▼
 Redis 10.10.3.50 (:6379)   ← B 전용
```
> Standard ILB 백엔드는 기본 아웃바운드가 비활성이라 `snet-was`에 NAT Gateway를 추가한다.

---

## 테스트 A — mod_proxy + L4 ILB, Redis 無

```
사용자 ─▶ AGW ─▶ Apache×2 (mod_proxy_http) ─▶ L4 ILB (5-tuple) ─▶ Tomcat×2
                                                                   세션 = Tomcat 로컬 메모리 (공유 ✗)
```

| # | 항목 | 결과 |
|---|------|------|
| A-1 | Client IP | ✅ 실 IP `121.128.200.167` 복원 (RemoteIpValve) |
| A-2 | 호출 추적 | ✅ Apache 부여 ID가 Tomcat까지 전달 |
| A-3 | 분산 | ⚠️ 기본 12:0 편중 → **`disablereuse=on`** 적용 후 25:15 |
| A-4 | **세션** | ❌ **백엔드 flip 시 유실**(40회 중 3회 리셋) |

IP·추적은 mod_proxy만으로 성립하고, 세션은 L4로 처리할 수 없다.

---

## 테스트 B — + Redis 공유 세션

```
사용자 ─▶ AGW ─▶ Apache×2 (mod_proxy_http) ─▶ L4 ILB ─▶ Tomcat×2 ═▶ Redis 공유 세션
                                                          jvmRoute 제거      (10.10.3.50)
```

- A의 유일한 실패(세션)를 Redis 외부 세션으로 해결.
- Tomcat `context.xml`에 `RedissonSessionManager`, `jvmRoute` 제거(스티키 불필요).
- 백엔드가 15회 바뀌어도 Redis에서 같은 세션을 읽음. **L4 세션 유실 완전 해소(실증)**.

| # | 항목 | 결과 |
|---|------|------|
| B-1 | 백엔드 분산 | flip 15회 (두 Tomcat 오감) |
| B-2 | 세션 동일성 | ✅ 30회 내내 sessionId 동일 |
| B-3 | 세션 연속성 | ✅ count 1→30 연속(유실 0) |
| B-4 | 저장 확인 | ✅ Redis 세션 키 115개, TTL 1855s |


---

## 테스트 C — mod_jk(AJP), 고객 현행

```
사용자 ─▶ AGW ══▶ Apache×2 (mod_jk, sticky_session=1) ══AJP:8009══▶ Tomcat×2 (jvmRoute=tomcatN)
          round-robin        JSESSIONID의 .tomcatN 접미사를 읽어 같은 Tomcat 고정   (ILB·Redis 없음)
```

- Tomcat `jvmRoute`로 세션ID에 `.tomcatN` 접미사 부여.

| # | 항목 | 결과 |
|---|------|------|
| C-1 | Client IP | ✅ 실 IP `211.226.86.198` AJP로 전달 |
| C-2 | 호출 추적 | ✅ Request-Id AJP 통과 |
| C-3 | 신규 세션 분산 | ✅ 25:5 (mod_jk lb) |
| C-4 | **세션(sticky)** | ✅ **20회 내내 tomcat2 고정, sessionId 동일, count 1→20** (Redis 없이) |

- AGW가 앞단 Apache를 round-robin 해도, mod_jk가 `.tomcat2` 접미사를 읽어 같은 Tomcat 고정.
- **N Apache × M Tomcat 에서도 외부 저장소 없이 sticky 성립** = 고객이 mod_jk를 고수하는 이유.
- 대가는 AJP(Ghostcat CVE-2020-1938, 폐기 권장) + Apache↔Tomcat 강결합.

<details><summary>재현 <code>workers.properties</code></summary>

```properties
worker.list=lb
worker.tomcat1.type=ajp13 ; worker.tomcat1.host=10.10.3.4 ; worker.tomcat1.port=8009
worker.tomcat2.type=ajp13 ; worker.tomcat2.host=10.10.3.5 ; worker.tomcat2.port=8009
worker.lb.type=lb ; worker.lb.balance_workers=tomcat1,tomcat2 ; worker.lb.sticky_session=1
```
Tomcat: `<Engine ... jvmRoute="tomcatN">` + `<Connector protocol="AJP/1.3" port="8009" secretRequired="false"/>`
</details>

---

## 결과 요약 (3-way)

| 항목 | A (ILB, Redis 無) | B (ILB, Redis 有) | C (**mod_jk**) |
|------|:---:|:---:|:---:|
| Client IP · 호출 추적 | ✅ | ✅ | ✅ |
| 트래픽 분산 | ✅ (disablereuse) | ✅ | ✅ |
| **세션 유지** | ❌ 유실 | ✅ Redis | ✅ **네이티브 sticky** |
| **장애 failover** | 라우팅 ✅ / 세션 ❌ | 라우팅 ✅ / **세션 ✅** | 라우팅 ✅ / 세션 ❌ |
| 추가 인프라 | ILB | **ILB + Redis** | **없음** |
| 대가 | 세션 미보장 | Redis 비용·운영 | AJP 폐기·강결합 |

> 세 방식의 차이는 세션 처리에 있다. 원리는 아래에서 정리한다.

---

## 결과 분석 

| 항목 | 원리 | 성립 조건 |
|------|------|-----------|
| **Client IP** | L4는 NAT만 하고 헤더를 안 바꾸므로 실 IP는 **XFF**로 흐른다. Tomcat **RemoteIpValve**가 XFF 최초 IP를 `remoteAddr`로 복원한다 (valve가 XFF를 소비하므로 JSP에서 XFF는 null이 정상) | mod_proxy + RemoteIpValve |
| **호출 추적** | Apache가 `mod_unique_id`/`mod_headers`로 Request-Id를 주입하고 L4가 안 지워 Tomcat까지 간다. **추적 원점=Apache**(AGW는 고유 ID를 안 넣으므로 AGW 로그+타임스탬프로 보완) | Apache에서 ID 생성 |
| **분산 편중** | L4 = 5-tuple 해시 + 출발지 IP가 Apache 2개뿐 + **mod_proxy keepalive가 flow를 한 Tomcat에 pin**해 12:0 | **`disablereuse=on`**으로 25:15 (연결비용↑, 부하 크면 ttl 튜닝) |
| **세션 유실 (A)** | L4는 쿠키(JSESSIONID)를 못 읽어 스티키가 안 되고, flip 시 다른 Tomcat엔 세션이 없다 | 해결=스티키 or 외부화 |
| **세션 유지 (B)** | **외부화**: Redis로 공유해 Tomcat이 무상태가 되고, 어느 노드든 동일 세션(flip 15회에도 count 연속) | Redis + Redisson |
| **세션 유지 (C)** | **스티키**: `jvmRoute` 접미사(`.tomcatN`)를 mod_jk가 읽어 같은 Tomcat으로 고정한다. 앞단 AGW가 Apache를 바꿔도 규칙이 같아 N×M sticky | jvmRoute + sticky_session=1 |

### 추적 — 상관 ID 전파와 APM 분산추적은 다른 층위

이 테스트에서 확인한 "호출 추적"은 **요청 상관 ID(Request-Id)** 가 Apache→(L4/AJP)→Tomcat까지 헤더로 전파되는 수준이다. 고객의 원래 관심인 **APM 분산추적(span 연속)** 은 이와 층위가 다르다.

| 층위 | 무엇 | LB(L4/mod_jk)의 역할 | 연속성 조건 |
|------|------|----------------------|-------------|
| 상관 ID | Request-Id / traceparent 헤더 전파 | 헤더를 안 지우고 통과(투명) | Apache에서 ID 생성, 이번 실증으로 확인 |
| APM 분산추적 | 각 노드 span 생성·연결 | LB는 무관 — 계측 대상 아님 | **각 노드(Apache·Tomcat)에 APM 에이전트 설치·계측** |

- L4 ILB든 mod_jk든 헤더를 그대로 넘기므로, 추적 헤더 연속성 자체는 LB 방식과 무관하다.
- 상용 APM(제니퍼·와탭·스카우터·Datadog 등)에서 span이 이어지려면 **중간 노드가 계측되어야** 한다. LB를 바꾼다고 끊기거나 이어지지 않는다.
- Apache 구간까지 span으로 보려면 Apache에 APM/OTel 모듈이 필요하고, 없으면 Tomcat부터 span이 시작된다(헤더는 이어짐).



| 축 | 내용 | 검증 |
|----|------|------|
| **스티키(고정)** | 같은 사용자를 같은 노드로 — L4는 불가 / **mod_jk는 `jvmRoute`로 가능** | ✗ ILB(A) / ✅ mod_jk(C) |
| **세션 외부화** | 세션을 Redis에 두고 공유하므로 어느 노드든 동일 | ✅ (B) |

**mod_jk가 Redis 없이 sticky 되는 원리**
```
Tomcat  jvmRoute="tomcat2"  →  JSESSIONID = …E9.tomcat2   (세션 위치를 ID에 각인)
Apache  mod_jk가 .tomcat2 접미사를 읽음  →  항상 tomcat2로 라우팅 (앱계층 파싱, L4는 불가)
```
"스티키만 필요하고 Redis는 쓰기 싫다"는 요구를 그대로 충족한다. 대가는 AJP(Ghostcat, 폐기 권장)와 Apache·Tomcat 강결합이다.

### 장애 failover — mod_jk 자동 failover는 다른 케이스에서 커버되나

mod_jk의 "자동 failover"는 **`worker.lb`가 죽은 Tomcat을 감지해 워커 목록에서 제외**하는 것 = **라우팅(가용성)** 차원이다.
A·B에서는 **L4 ILB 헬스프로브**가, AGW 직결에서는 **AGW 백엔드 프로브**가 **동일 역할**을 하므로, 라우팅 failover는 세 방식 모두 자동으로 커버된다.
단 **진행 중이던 세션의 생존은 별개** — mod_jk든 ILB든 죽은 노드의 로컬 세션은 사라진다. failover 시 세션까지 지키려면 **세션 외부화(Redis, B)** 가 필요.

| failover 관점 | ① mod_jk (C) | ② ILB, Redis 無 (A) | ③ ILB + Redis (B) |
|---|:---:|:---:|:---:|
| 장애 감지·차단 | `worker.lb` 자동 | ILB 헬스프로브 자동 | ILB 헬스프로브 자동 |
| 신규 요청 라우팅 | ✅ 생존 노드로 | ✅ 생존 노드로 | ✅ 생존 노드로 |
| **진행 세션 생존** | ❌ 재로그인 | ❌ 재로그인 | ✅ Redis로 유지 |
| 복구 노드 재투입 | ✅ 자동(worker recover) | ✅ 자동(프로브) | ✅ 자동(프로브) |

> **결론**: mod_jk 자동 failover(=라우팅 가용성)는 **ILB/AGW 프로브가 그대로 대체하므로** A·B가 커버한다.
> **세션까지 지키는 failover는 B(Redis)만** 성립하므로, 이 관점에선 **B가 mod_jk보다 오히려 우월**.
> (mod_jk에 세션복제 옵션이 있으나 강결합이 심해지고 이번엔 미검증.)
>
> ※ 위 표는 각 컴포넌트의 **표준 동작(헬스프로브·worker 상태) 기반 분석**이다. 이번 실측은 분산·세션 지속성 중심이라 **노드 강제 종료(kill) failover는 별도 측정하지 않았다.** 실측 증빙이 필요하면 재배포해 kill 테스트 수행 가능.

### 성립 레시피 · 선택지

**mod_proxy + L4 ILB로 성립하는 레시피**
```
① mod_proxy_http                    → IP·추적 헤더가 L4 통과
② disablereuse=on + RemoteIpValve   → 분산 균등 + 실 IP 복원   (①② 추가 인프라 0)
③ Redis 외부 세션                    → 세션 유실 해소            (③만 Redis 필요)
```

| 방식 | 세션 | 대가 | 언제 |
|------|------|------|------|
| ① mod_jk 유지 | 네이티브 sticky | AJP 폐기·강결합 | 현행 유지, 변경 최소화 |
| ② L4 ILB + Redis | Redis 공유 | Redis 비용·운영 | Azure 표준 강제 + 세션 필요 |
| ③ **AGW→WAS 직결** | L7 쿠키 어피니티 | 웹서버 기능 이관 | Redis도 mod_jk도 피할 때 ([문서 3](03-agw-direct-review.md)) |

### 잔여 리스크

| 항목 | 내용 |
|------|------|
| 홉 증가 | Apache→ILB→Tomcat 홉·장애지점 추가 |
| 분산 엔트로피 | 출발지 IP가 Apache 수만큼뿐이라 완전 균등은 어려움 |
| Redis 비용/운영 | Managed Redis 비용 or VM 최소 3노드(고객 지적) |
| keepalive 성능 | disablereuse는 분산↑·연결비용↑ |
