# Web-WAS 아키텍처 검토


1. **Web-WAS L4 ILB 검증** — Apache–Tomcat 사이 L4 ILB 강제 시 IP/추적/세션 유지 여부 (Azure 실증) + 고객 현행 **mod_jk(AJP) 3-way 비교**
2. **AGW→WAS 직결 검토** — 웹서버 제거 대안의 장단점/제약
3. **온프렘↔Azure TLS 강제** — AGW vs AKS Gateway API 비교


## 목차

| 문서 | 주제 |
|------|------|
| [1. 테스트 시나리오·결과·분석](docs/01-scenarios-and-results.md) | L4 ILB 실증 — 환경·시나리오 A/B/C·3-way·원리 분석 |
| [2. 실측 원자료](docs/02-raw-test-data.md) | 측정값·환경·재현 설정 원본 |
| [3. AGW→WAS 직결 검토](docs/03-agw-direct-review.md) | 웹서버 제거 대안 |
| [4. 온프렘 TLS 옵션](docs/04-onprem-tls-options.md) | AGW vs AKS Gateway API |

---

## 한눈에 — 무엇이 되고 안 되나

Web-WAS 구성 4가지 방식의 최종 비교. (✅ 됨 · ⚠️ 조건부/대응 필요 · ❌ 안 됨)

| 항목 | ① mod_jk<br/>(고객 현행) | ② ILB+Redis | ③ ILB<br/>(Redis 無) | ④ AGW→WAS<br/>직결 |
|------|:---:|:---:|:---:|:---:|
| Client IP 보존 | ✅ | ✅ | ✅ | ✅ |
| 호출 추적(Request-Id) | ✅ | ✅ | ✅ | ⚠️ WAS서 생성 |
| 트래픽 분산 | ✅ | ✅ | ⚠️ `disablereuse` | ✅ |
| **세션 유지** | ✅ sticky | ✅ Redis | ❌ 유실 | ✅ 쿠키 어피니티 |
| 장애 failover(라우팅) | ✅ worker.lb | ✅ 프로브 | ✅ 프로브 | ✅ 프로브 |
| **failover 시 세션 생존** | ❌ | ✅ | ❌ | ❌ |
| 웹서버(Apache) | 필요 | 필요 | 필요 | ❌ 제거 |
| 추가 인프라 | **없음** | ILB+Redis | ILB | 없음(AGW만) |
| 백엔드 프로토콜 | AJP | HTTP | HTTP | HTTP(gRPC·AJP ✗) |
| 정적·PHP·스크립트 | Apache 처리 | Apache | Apache | ❌ 이관 필요 |
| 주요 대가 | AJP 폐기·강결합 | Redis 비용·운영 | 세션 유실 | 웹서버 기능 이관 |

**언제 무엇을**
- 현행 최소 변경: **①**
- Azure 표준 + 세션·failover 보장: **②**
- 웹서버까지 제거(정적·스크립트·auth 모듈 없을 때): **④**
- ③은 세션이 불필요한 무상태 서비스에 한한다.

### ④ AGW 직결이 안 되는 / 어려운 경우

아래에 하나라도 해당하면 ④는 부적합하다. 웹서버를 유지(①②)하거나 부분 하이브리드로 간다. (불가 또는 조건부, 전체 8케이스는 [문서 3 §3.8](docs/03-agw-direct-review.md))

| 제약 | 이유 | 판정 |
|------|------|:---:|
| 인증/인가 Apache 모듈 (SiteMinder·mod_auth_openidc/kerb·국내 SSO 에이전트) | AGW에 대체 모듈 없음 → 앱 이관 or APIM/Entra App Proxy | ❌ 가장 흔함 |
| 국내 보안 솔루션 모듈 (전자서명·암복호화·마스킹·웹방화벽 에이전트) | Tomcat 단독 불가 → 벤더 Java 지원 확인 先 | ❌ |
| PHP/Perl/CGI · 복잡한 mod_rewrite(RewriteMap·다중 Cond) | AGW에 스크립트/rewrite 엔진 없음 | ❌ |
| gRPC(백엔드 HTTP/2) · AJP 의존 | AGW 백엔드는 HTTP/1.1 | ❌ |
| 커스텀 mod_security · 감사 로그포맷 고정(ISMS-P) | WAF/로그 1:1 대체 아님 → 재작성·감사 승인 | ⚠️ |
| 정적 콘텐츠 다량 | 처리량·캐싱 저하 → Blob+CDN 분리(URL 구조 변경) | ⚠️ |
| mTLS 경로별 검증 | AGW mTLS는 리스너 단위·CRL/OCSP 제한 | ⚠️ |

> 진단: `httpd.conf`+`conf.d/*.conf`의 **`LoadModule` 목록**에서 표준 외 서드파티 모듈이 하나라도 있으면 검토 필요.

---

## 주제 1 — 한 장 결론

Apache–Tomcat 사이에 **L4 ILB**를 강제해도, 세 방식 중 **세션 처리만 다름**(IP/추적은 공통).

```
A  사용자 → AGW → Apache×2(mod_proxy) → L4 ILB → Tomcat×2            : 세션 유실 ❌
B  A + Redis 공유 세션                                               : 세션 유지 ✅ (Redis 비용)
C  사용자 → AGW → Apache×2(mod_jk/AJP) → Tomcat×2  [고객 현행]        : 세션 유지 ✅ (AJP·강결합)
```

- **A/B/C 모두** Client IP(XFF+RemoteIpValve)·호출 추적(Request-Id) 성립.
- **L4 ILB는 쿠키를 몰라 스티키가 안 되므로** Redis 외부화가 필요하다(B).
- **mod_jk는 `jvmRoute` 네이티브 스티키라서** Redis 없이 세션을 유지한다(C). 대가는 AJP(폐기 권장)와 강결합이다.

환경: Korea Central / Standard_D2s_v5 / Tomcat 9 / 진단용 `index.jsp`. 검증 후 리소스 삭제, cloud-init으로 재현 가능.
