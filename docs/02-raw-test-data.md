# 2. 실측 원자료 (Raw Test Data)

Azure에 실제 배포해 측정한 원자료. 분석과 결론은 [문서 1](01-scenarios-and-results.md)에 있고, 이 문서는 근거가 된 측정값·환경·명령을 그대로 남긴다. 리소스는 측정 후 모두 삭제했다(과금 없음). cloud-init(YAML)으로 재현 가능하다.

## 공통 사항

- 리전 / SKU: Korea Central / Standard_D2s_v5
- OS: Debian, Tomcat 9, Apache 2
- 서브넷: `snet-agw` 10.10.1.0/24 · `snet-web` 10.10.2.0/24 · `snet-was` 10.10.3.0/24
- 노드 IP: apache1 10.10.2.4 · apache2 10.10.2.5 · tomcat1 10.10.3.4 · tomcat2 10.10.3.5
- 포트: Apache 80 · Tomcat HTTP 8080 / AJP 8009 · Redis 6379 · ILB VIP 10.10.3.100
- 테스트 앱: 프레임워크 없는 진단용 `index.jsp` — hostname · remoteAddr · 요청헤더 · sessionId · 세션 카운터(count) 출력
- 검증 방식: `az vm run-command invoke -g <rg> -n <vm> --command-id RunShellScript --scripts "curl ..."` 를 반복해 응답의 hostname / sessionId / count를 집계 (VM에 public IP·SSH 없음)
- `snet-was` 아웃바운드: Standard ILB 백엔드는 기본 아웃바운드가 비활성이라 NAT Gateway를 추가

---

## 테스트 A — mod_proxy + L4 ILB, Redis 없음

리소스 그룹 `rg-webwas-ilb-test` (삭제됨).

| 항목 | 값 |
|------|-----|
| AGW 공인 IP | 20.214.147.221 |
| 측정 클라이언트 IP | 121.128.200.167 |
| 세션 저장 | Tomcat 로컬 메모리 (공유 없음), jvmRoute 있음 |

측정값:

| 항목 | 측정 결과 |
|------|-----------|
| Client IP 복원 | `remoteAddr` = 121.128.200.167 (실제 공인 IP) — Tomcat RemoteIpValve |
| Request-Id / Apache-Id 전파 | Apache 부여 ID가 Tomcat까지 전달됨 |
| AGW → Apache 분산 | 6 : 6 |
| ILB → Tomcat 분산 (기본 keepalive) | **12 : 0** (한 노드 편중) |
| ILB → Tomcat 분산 (`disablereuse=on`, 40건) | **25 : 15** |
| 세션 유지 | ❌ 백엔드 flip 시 sessionId 리셋 — **40건 중 3회 리셋** |

편중 원인: L4 = 5-tuple 해시 + 출발지 IP가 Apache 2개뿐 + mod_proxy keepalive가 flow를 한 Tomcat에 pin. `disablereuse=on`(요청마다 새 연결)으로 완화.

---

## 테스트 B — mod_proxy + L4 ILB + Redis 공유 세션

리소스 그룹 `rg-webwas-redis-test` (삭제됨). 구성은 A와 동일 + Redis VM(10.10.3.50) + Tomcat에 Redisson 세션매니저, jvmRoute 제거.

| 항목 | 값 |
|------|-----|
| AGW 공인 IP | 20.214.147.210 |
| Redis | 10.10.3.50:6379 (bind 0.0.0.0, protected-mode no) |
| 세션매니저 | Redisson 3.27.2 (`redisson-all` + `redisson-tomcat-9`) |

측정값 (쿠키 유지 30요청):

| 항목 | 측정 결과 |
|------|-----------|
| 백엔드 flip | 30요청 중 **15회** (두 Tomcat 오감) |
| sessionId 동일성 | ✅ 30회 내내 동일 (`6440E3AD3F91…`) |
| count 연속성 | ✅ 1 → 30 연속 (유실 0) |
| Redis 세션 키 | 115개, TTL 1855s (`redis-cli keys '*'`) |

flip이 15회 일어나도 sessionId·count가 끊기지 않았고, Redis 공유 세션으로 L4 ILB 세션 유실이 해소됨을 실측했다.

### 재현 설정 (요지)

Tomcat `context.xml`:
```xml
<Manager className="org.redisson.tomcat.RedissonSessionManager"
         configPath="${catalina.base}/conf/redisson.yaml"
         readMode="REDIS" updateMode="AFTER_REQUEST"/>
```
`redisson.yaml`은 `/etc/tomcat9/redisson.yaml`(= `${catalina.base}/conf/redisson.yaml`, Debian 심볼릭 링크). jar은 `/usr/share/tomcat9/lib`에 배치.

---

## 테스트 C — mod_jk(AJP), 고객 현행

리소스 그룹 `rg-webwas-modjk-test` (삭제됨). Apache `libapache2-mod-jk`, Tomcat `jvmRoute=tomcatN` + AJP 커넥터. ILB·Redis 없음.

| 항목 | 값 |
|------|-----|
| AGW 공인 IP | 20.194.25.57 |
| 측정 클라이언트 IP | 211.226.86.198 |
| 세션 | jvmRoute 접미사 기반 네이티브 sticky |

측정값:

| 항목 | 측정 결과 |
|------|-----------|
| Client IP | `remoteAddr` = 211.226.86.198 (AJP로 전달) |
| X-Request-Id 전파 | AJP 통과, Tomcat까지 전달 |
| 신규 세션 분산 (mod_jk lb) | **25 : 5** |
| 세션 sticky (20요청) | ✅ **20 / 20 tomcat2 고정, sessionId 동일, count 1 → 20** |

AGW가 앞단 Apache를 번갈아 보내도, mod_jk가 JSESSIONID의 `.tomcat2` 접미사를 읽어 같은 Tomcat으로 고정. 외부 세션 저장소 없이 sticky 성립.

### 재현 설정 (요지)

`workers.properties`:
```properties
worker.list=lb
worker.tomcat1.type=ajp13
worker.tomcat1.host=10.10.3.4
worker.tomcat1.port=8009
worker.tomcat2.type=ajp13
worker.tomcat2.host=10.10.3.5
worker.tomcat2.port=8009
worker.lb.type=lb
worker.lb.balance_workers=tomcat1,tomcat2
worker.lb.sticky_session=1
```
Tomcat: `<Engine ... jvmRoute="tomcatN">` + `<Connector protocol="AJP/1.3" port="8009" secretRequired="false"/>`

---

## 측정 환경 메모

- RemoteIpValve 기본 `internalProxies`가 10.x 대역을 신뢰하므로 AGW(10.10.1.x)·Apache(10.10.2.x)를 strip한 뒤 실제 클라이언트 IP를 복원한다. XFF는 valve가 소비하므로 JSP에서는 null로 표시(정상).
- AGW는 XFF를 `IP:port` 형태로 삽입.
- VM 접근은 public IP·SSH 없이 `az vm run-command` 로만 수행. 아웃바운드가 필요한 curl은 NAT Gateway를 통해 동작.
