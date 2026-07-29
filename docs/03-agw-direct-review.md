# 3. AGW→WAS 직결 구조 검토

고객 문의: L4 ILB의 세션·비용 부담을 피하려 **웹서버를 제거하고 AGW(L7)를 WAS에 직결**할 수 있는가.

## 결론

**타당하며 표준의 "추가 옵션"으로 채택 가능.** 단 정적 콘텐츠·백엔드 TLS·WAF 갭이 선결.

```
기존 :  사용자 → AGW → Apache×2 → [L4 ILB] → Tomcat×2 -.-> Redis
제안 :  사용자 → AGW(L7, 쿠키 어피니티, WAF·TLS) ───────→ Tomcat×2
                 └ 웹서버 · L4 ILB · Redis 모두 제거
```

**한눈에 — AGW 직결 되는 것 / 안 되는 것**

| | 항목 |
|--|------|
| ✅ **된다** | TLS 종단 · WAF_v2 · 세션 스티키(쿠키 어피니티) · failover(헬스프로브) · Client IP(XFF) · WebSocket · mTLS(리스너 단위) |
| ⚠️ **대응 필요** | 정적(Blob/CDN 이관) · 추적 ID(WAS 생성) · 복잡한 rewrite(앱 이관) · 백엔드 TLS(재암호화) |
| ❌ **안 된다 (웹서버 유지)** | PHP·CGI·스크립트 · 커스텀 mod_security · **auth/SSO 모듈** · 국내 보안 모듈 · gRPC(백엔드 HTTP/2)·AJP |

## 치환 참조 아키텍처 (일반 패턴)

웹서버를 AGW로 걷어낼 때 실무에서 쓰이는 대표 4패턴. 앱 특성에 맞게 조합한다.

**① 기본형 — AGW → WAS 직결** (동적 위주, 정적 적음)
```
사용자 ─▶ AGW (WAF·TLS 종단·쿠키 어피니티) ─▶ Tomcat×2 (backend pool, :8080)
```
가장 단순. 정적은 WAS(war)가 서빙. 정적 비중 낮고 스크립트 의존 없을 때.

**② 정적 분리형 — AGW + Blob** (정적 비중 높음, 고객 언급 패턴)
```
                              ┌─(path /static/*, /img/*)─▶ Blob Storage (정적 웹)
사용자 ─▶ AGW (WAF·TLS) ──────┤                              └ Private Endpoint + Private DNS
   경로 기반 라우팅            └─(그 외 동적)──────────────▶ Tomcat×2 (:8080)
```
정적은 Blob이 처리(Tomcat 부하↓·캐싱↑). 백엔드 풀에 Blob FQDN 등록 + **호스트헤더 override** 필요.
보안: **Private Endpoint + public/익명 차단**. 전역 캐싱까지 원하면 앞단에 Front Door/CDN.

**③ End-to-End TLS형 — 백엔드 재암호화** (전송 중 암호화 정책 강제)
```
사용자 ─▶ AGW (TLS 종단·WAF) ══HTTPS:8443 재암호화══▶ Tomcat×2 (server 인증서, Key Vault+MI)
```
AGW→WAS 구간도 TLS. Tomcat에 서버 인증서(PKCS12) 탑재. 백엔드 HTTP 설정=HTTPS + 신뢰 CA.

**④ 하이브리드 — 부분 웹서버 유지** (auth 모듈·PHP/CGI 등 하드 블로커 존재)
```
                              ┌─(정상 경로)─────────────▶ Tomcat×2 직결
사용자 ─▶ AGW (WAF·TLS) ──────┤
   경로 기반 라우팅            └─(/legacy, /auth, *.php)─▶ Apache(잔존) ─▶ Tomcat/CGI
```
인증 모듈·복잡한 rewrite·PHP/CGI만 Apache에 남기고, 나머지는 직결로 점진 전환. 마이그레이션 과도기.

| 패턴 | 언제 | 선결 |
|------|------|------|
| ① 기본형 | 정적 적음·순수 동적 | 없음(가장 단순) |
| ② AGW+Blob | 정적 비중 높음 | Blob Private Endpoint·호스트헤더·URL 구조 |
| ③ e2e TLS | 백엔드 암호화 정책 | WAS 서버 인증서(Key Vault) |
| ④ 하이브리드 | auth/PHP/CGI 잔존 | Apache 일부 유지·경로 설계 |

## 3.1 고객 딜레마 해소

L4는 쿠키를 몰라 스티키가 안 돼 Redis를 강제했던 것. AGW는 L7이라 **자체 쿠키 어피니티**로 스티키 처리.

| 고객 요청 | 해소 |
|-----------|------|
| Redis 불필요 | ✅ AGW 쿠키 어피니티 |
| WAS 세션복제 불필요 | ✅ 불필요 |
| Redis 비용·3노드 부담 | ✅ 제거 |

## 3.2 장점

- 웹서버 VM을 제거해 **EOS/패치 대상·공격 표면을 축소**하고 운영 부담을 낮춘다
- Redis/세션복제 없이 스티키 단순화
- TLS·인증서 AGW 일원화(Key Vault), 프런트 **WAF_v2(OWASP CRS)**

## 3.3 제약 — Apache가 하던 일 중 AGW가 못 대체하는 것

Apache가 하던 일을 6가지로 나눠 AGW 대체 가능 여부를 정리했다.

| Apache 기능 | AGW 대체 | 안 되면 | 판정 |
|---|---|---|:---:|
| ① TLS 종단 | ✅ (인증서가 WAS로 분산) | WAS마다 서버 인증서(Key Vault+MI) | ⚠️ |
| ② 정적 파일 서빙 | ❌ 프록시라 못 뿌림 | Blob/CDN 또는 WAS(war) 이관 | ⚠️ |
| ③ 스크립트/rewrite/커스텀 WAF | ❌ 실행 엔진 없음 | 의존 시 **웹서버 유지** | ❌ |
| ④ 백엔드 프로토콜 | 프런트 HTTP/2 ✅, **백엔드 HTTP/1.1** | gRPC·AJP 불가 (WebSocket ✅) | 제약 |
| ⑤ 세션 스티키 | ✅ 쿠키 어피니티 | 장애·배포·스케일 순간 유실 | 리스크 |
| ⑥ 요청 고유 ID(추적) | ❌ 미주입 | WAS Valve/Filter로 생성 | 대응개발 |

①④⑤는 수용 가능하고 **②③⑥이 실제 판단 포인트**다. 세션 유실은 '중요 서비스 아님'으로 넘길 수 있으나 문서에 명시해야 한다.

## 3.4 미들웨어별

| 미들웨어 | 결과 | 비고 |
|----------|:----:|------|
| Tomcat 9/10 | ✅ | HTTP 커넥터 + `RemoteIpValve`(XFF 실 IP 복원) |
| WildFly/JBoss | ✅ 조건부 | Undertow HTTP 지원, mod_cluster 동적등록 상실로 정적 풀 |
| AJP 의존 앱 | ❌ | AGW가 AJP 미지원이라 HTTP 전환 |
| PHP/Perl/CGI | ❌ | 웹서버 유지 |
| gRPC(백엔드 HTTP/2) | ❌ | 프런트 HTTP/2는 지원, 백엔드는 HTTP/1.1 |

## 3.5 개발·운영 관점 걸림돌

**하드 블로커** — 있으면 웹서버 유지: gRPC/백엔드 스트리밍, Apache 종속 코드(`SSL_CLIENT_CERT`·AJP 속성), + 3.3의 ②③.

**관리 가능하나 비용 발생 (Top)**
| 항목 | 내용 |
|------|------|
| 관측성 재구축 | Apache 로그/mod_status → AGW 진단로그·WAF로그·KQL 기반 재작성 |
| 셸 없는 RCA | 관리형이라 tcpdump가 불가해 로그·메트릭만으로 원인분석 |
| WAF 튜닝 | CRS 오탐으로 앱별 예외 상시 운영 |
| 헬스프로브/타임아웃 | 프로브 경로·200 매칭 직접 정의, backend 20s·idle 4min 조정 |
| 배포 드레이닝 | connection draining 없으면 Redis 없는 노드 세션 유실 |
| 변경관리 | VM conf 편집 → Azure 리소스(IaC/RBAC) |

## 3.6 보안실 검토 포인트

1. AGW→WAS 백엔드 평문 허용 여부(불가 시 e2e TLS)
2. Blob 정적: Private Endpoint + 익명·public 차단 + private DNS
3. WAF 갭: 커스텀 mod_security를 AGW WAF가 어디까지 커버하는지 분석
4. 긍정: 웹서버 VM(EOS/패치) 제거 = 명백한 보안 이점

## 3.7 권고

- ✅ **적용 가능**: Tomcat/WildFly(HTTP), 정적·스크립트 의존 없음, 세션 유실 수용, 트레이스ID WAS 생성
- ⚠️ **선결**: 백엔드 TLS 정책, 정적 이관 방식, WAF 갭
- ❌ **기존 표준 유지**: PHP/Perl/CGI · 복잡한 rewrite · 커스텀 mod_security · AJP 의존

## 3.8 판정 체크리스트 — Apache 제거가 어렵거나 불가능한 케이스

> **최우선 진단:** `httpd.conf`+`conf.d/*.conf`의 **`LoadModule` 목록**을 뽑아 표준 모듈 외 **서드파티 모듈이 하나라도 있으면 검토 필요**.

| # | 케이스 | 대안 / 정정 | 판정 |
|---|--------|-------------|:----:|
| 1 | **인증/인가를 Apache 모듈이 담당**(SiteMinder, mod_auth_openidc/kerb, 국내 SSO 에이전트) | 앱 이관 또는 APIM/Entra App Proxy | ❌ **가장 흔한 중단 사유** |
| 2 | **복잡한 mod_rewrite**(RewriteMap, 다중 Cond, `%{HTTP:}`) | AGW rewrite는 **AND·RE2**만 지원해 OR분기·RewriteMap은 불가. 복잡 로직 앱 이관 | ⚠️ 규모 비례 |
| 3 | **mTLS**(`SSLVerifyClient`+DN 헤더) | **AGW v2 mTLS 지원** + 인증서 속성 rewrite 헤더 주입으로 대체 **가능**. 단 리스너 단위·CRL/OCSP 제한 | ⚠️ 되나 제약 |
| 4 | **국내 보안 솔루션이 Apache 모듈**(전자서명·암복호화·마스킹·웹방화벽 에이전트) | 벤더 Tomcat/Java 지원 확인 先 | ❌ 벤더 종속 |
| 5 | **다수 앱 단일 진입점**(PHP·CGI·타 WAS 공동) | AGW 경로 라우팅으로 전개하며 리스너/풀 상한 확인 | ⚠️ 복잡화 |
| 6 | **정적 비중 매우 높음** | Blob+CDN 분리(대개 URL 구조 변경) | ⚠️ 선행작업 |
| 7 | **규제·감사 WAF/로그 고정**(ModSecurity 커스텀·ISMS-P 로그포맷) | AGW WAF 재작성 + 로그 대체 감사 승인 | ⚠️ 거버넌스 |
| 8 | **세밀한 압축·헤더 조작** | Tomcat 커넥터 압축+서블릿 필터로 상당 커버(config parity는 아님) | ⚠️ 앱 필터 |

**하드 블로커**: #1·#4와 3.3의 ②③. 해당 시 웹서버를 유지한다.

## 3.9 참고 자료

- **rewrite 판정**: [Rewrite headers/URL](https://learn.microsoft.com/en-us/azure/application-gateway/rewrite-http-headers-url) — server variable·조건식(**AND, RE2**). v2 SKU 전용
- **mTLS**: [Mutual auth 개요](https://learn.microsoft.com/en-us/azure/application-gateway/mutual-authentication-overview) (Strict/Passthrough, Passthrough는 API 2025-03-01+ ARM 전용) · [PowerShell](https://learn.microsoft.com/en-us/azure/application-gateway/mutual-authentication-powershell) (CA 체인 1파일, 25KB)
- **제한값**: [FAQ](https://learn.microsoft.com/en-us/azure/application-gateway/application-gateway-faq) · [서비스 제한](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits) — 리스너·풀·룰셋 상한(#5)
- **Tomcat**: [RemoteIpValve](https://tomcat.apache.org/tomcat-9.0-doc/config/valve.html) · [HTTP Connector(compression·maxThreads·timeout)](https://tomcat.apache.org/tomcat-9.0-doc/config/http.html) · [Reverse Proxy How-To](https://tomcat.apache.org/connectors-doc/common_howto/proxy.html)
- **WAF**: [WAF v2 개요](https://learn.microsoft.com/en-us/azure/web-application-firewall/ag/ag-overview) · [커스텀 룰](https://learn.microsoft.com/en-us/azure/web-application-firewall/ag/custom-waf-rules-overview)
