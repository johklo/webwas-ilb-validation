# 4. 온프렘↔Azure TLS 강제 — 옵션 분석

배경: 기존 **온프렘↔Azure를 LB로 평문 HTTP** 하던 것을, 이제 **Azure 측에서 TLS 강제**(전송 중 암호화).
ExpressRoute 사설 피어링은 기본이 미암호화라 평문 스니핑 리스크가 실재한다.

## 결론

- 방향 자체는 맞다. 다만 **AGW·Gateway API 모두 HTTP(S) 전용**이라 "모든 워크로드"를 덮지는 못한다.
- **HTTP = AGW 또는 AKS Gateway API**, **비-HTTP(DB·파일·MQ 등) = IPsec 또는 워크로드 native TLS** 로 이원화한다.
- **TLS 종단 위치**(프런트까지 vs 파드까지 e2e)를 정책에 먼저 명시해야 한다. 종단 뒷구간은 다시 평문이 될 수 있다.

**한눈에 — 트래픽별 TLS 수단**

| 트래픽 | 수단 | WAF | 안 되는 것 |
|--------|------|:---:|-----------|
| HTTP (VM/PaaS/AKS) | ✅ **AGW**(외부 L7) | ✅ 관리형 | gRPC·AJP·비-HTTP |
| HTTP/gRPC (AKS 컨테이너) | ✅ **AKS Gateway API** | ❌ 직접 | AKS 밖·VM/PaaS·비-HTTP |
| 비-HTTP (DB·파일·MQ·LDAP) | ✅ **IPsec over ExpressRoute** | — | (AGW/Gateway로는 ❌) |
| 전 구간 물리 | ExpressRoute Direct + MACsec | — | ER Direct 전제·고가 |

```
기존:  온프렘 ─(ExpressRoute)→ Azure LB(L4, 평문 HTTP) → 워크로드
변경:  온프렘 ─(ExpressRoute)→ Azure [TLS 종단점] ─────→ 워크로드
```

## 4.1 옵션 A — AGW(외부 L7)에서 종단

```
온프렘 ──TLS──▶ ExpressRoute ──▶ AGW v2 (사설 프런트엔드)      ──HTTP/재암호화──▶ 워크로드
                                 TLS 종단 · WAF_v2 · mTLS                        (VM/PaaS/AKS)
```

- **보안**: ER 미암호화 해소, 관리형 **WAF_v2**로 검사, **mTLS**로 발신자 인증, 인증서 Key Vault+MI 중앙관리.
- **인프라**: AGW v2 **사설 전용 프런트엔드**(GA)로 ER 사설IP에 정석이다. 앱별 전용은 비용이 커서 **공유 AGW(멀티사이트 리스너)**로 다수를 수용한다(단 blast radius↑).
- **한계**: HTTP/HTTPS/WebSocket 전용. **gRPC·AJP·비-HTTP 불가.**

## 4.2 옵션 B — AKS Gateway API(인클러스터)에서 종단

```
온프렘 ──TLS──▶ ExpressRoute ──▶ AKS 내부 LB (L4 passthrough) ──▶ Gateway 파드 ──mTLS/HTTP──▶ Pod
                                 (TLS 안 건드림)                  (Envoy/NGINX/Istio, TLS 종단)
```

내부 LB는 L4 passthrough라 종단이 **게이트웨이 파드**이고, 온프렘~게이트웨이 구간 암호화로 요구를 충족한다(AGW 없이).

**구현체가 결론을 가른다**
| 구현체 | 관리 | WAF | 비고 |
|--------|:----:|:---:|------|
| App Gateway for Containers(AGC) | 관리형 | ❌ | gRPC/HTTP2 지원 |
| 관리형 Istio 애드온 | 관리형 | ❌(직접) | **메시 전체 mTLS** 강점 |
| App Routing(NGINX) | 관리형 | ❌ | 경량 |
| Envoy/NGINX 자체구축 | **자가운영** | 직접 | 서포트·거버넌스 경계 이슈 |

- **보안**: 종단이 앱 근접. **Istio면 메시 전체 mTLS**(AGW보다 강함). Envoy `x-request-id`로 추적 우수.
  단 **키가 클러스터 Secrets에 상주**하므로 KMS/KV 암호화·RBAC·네임스페이스 격리가 필수다. 뒷구간은 메시 mTLS로 덮어야 e2e.
- **인프라**: 별도 AGW 없이 **노드+내부 LB 1개**라 저렴하다. **gRPC/HTTP2 지원**.
- **한계**: **AKS 컨테이너 한정**(VM/PaaS/비-HTTP 못 덮음), K8s·cert-manager·Envoy 운영 성숙도 필요.

## 4.3 결정 매트릭스

| 항목 | AGW (외부 L7) | AKS Gateway API |
|------|---------------|-----------------|
| 적용 대상 | VM/PaaS/AKS 범용 | **AKS 컨테이너 한정** |
| **WAF** | ✅ 관리형 WAF_v2 | ❌ 직접(AGC도 미지원) |
| gRPC/백엔드 HTTP2 | ❌ (프런트 HTTP/2는 O) | ✅ (Envoy 계열) |
| mTLS/메시 | 프런트 mTLS | ✅ Istio면 메시 전체 |
| 비용 | 앱별 전용 시 높음 | 노드+ILB, 낮음 |
| 인증서 | Key Vault + MI | cert-manager / KV CSI |
| 운영 부담 | 낮음(관리형) | K8s 성숙도 필요 |
| 비-HTTP(DB 등) | ❌ | ❌ (TLSRoute L4 일부) |

## 4.4 "전부 TLS"는 앱 계층으론 불가

| 방식 | 커버 범위 | 비고 |
|------|-----------|------|
| AGW TLS | HTTP(S) | WAF/mTLS |
| AKS Gateway API TLS | AKS HTTP/gRPC | 저비용, 메시 mTLS |
| **IPsec S2S VPN over ExpressRoute** | **모든 프로토콜** | 투명 암호화, 비-HTTP 포함 |
| ExpressRoute Direct + MACsec | 물리 포트 L2 전체 | ER Direct 전제, 고가 |

## 4.5 권고

1. **유형별 이원화**: AKS HTTP/gRPC→AKS Gateway API / VM·PaaS HTTP→AGW(WAF 시 특히) / 비-HTTP→IPsec over ER 또는 native TLS.
2. **선결 확인**: WAF 필수 여부 · "Gateway API" 구현체 확정(AGC/Istio/App Routing) · TLS 종단 범위(프런트 vs 파드).
3. **공통 요건**: 온프렘 DNS→사설 프런트IP + 인증서 SAN 일치, Key Vault 인증서 수명주기 자동화.
