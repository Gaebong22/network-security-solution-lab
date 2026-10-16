# Network Security Solution Lab

UTM의 격리망에서 Ubuntu VM 3대를 연결하고, Gateway의 라우팅·방화벽 정책을 적용한 뒤 **허용 통신, 차단 통신, 경로 장애와 재부팅 후 지속성**을 검증한 개인 실습이다.

> **현재 범위 완료 (2026-10-16).** 확인한 성과는 아래의 라우팅·UFW·SSH 허용·TCP/8080 차단·경로 장애·재부팅 검증이다.

## 구성

```text
Mac 관리 단말 --SSH--> Gateway 관리 NIC (UTM Shared Network)

Test Client                 Security Gateway                    Protected Server
10.10.10.10/24 -- lab-wan -- 10.10.10.1/24
                             [IPv4 forwarding + UFW]
                             10.20.20.1/24 -- lab-lan -- 10.20.20.10/24
```

`lab-wan`과 `lab-lan`은 UTM Host Network로 만든 별도 실습망이다. Client는 Server 대역을 `10.10.10.1`로, Server는 Client 대역을 `10.20.20.1`로 보내도록 정적 경로를 설정했다. Gateway의 관리 NIC는 실습망과 별도의 Shared Network에 연결했다. 실제 인터페이스와 주소 구성은 [실습 환경 설계](docs/02_lab_architecture.md)에 정리했다.

## 적용 정책과 실제 결과

| 흐름 | Gateway 정책 | 확인 결과 |
| --- | --- | --- |
| Mac 관리 단말 → Gateway TCP/22 | 관리 NIC 입력 허용 | UFW 활성화 및 Gateway 재부팅 후 SSH 재접속 성공 |
| Client → Server TCP/22 | `lab-wan`→`lab-lan` 전달 허용 | Client에서 Server SSH 로그인 성공. Gateway 재부팅 후 재확인 |
| Client → Server TCP/8080 | 기본 전달 차단 | Client 요청 시간 초과, 같은 시각 Gateway의 `[UFW BLOCK]` TCP/8080 SYN 기록 확인 |

Gateway의 UFW는 active, 기본 **입력 deny / 출력 allow / 전달 deny**로 설정했다. `net.ipv4.ip_forward=1`을 설정해 두 실습망 사이에서 라우팅이 가능하게 한 뒤, 필요한 SSH 흐름만 예외로 허용했다. 따라서 이 결과는 모든 프로토콜이 차단됐다는 주장이 아니라 **명시한 TCP/8080 흐름**에 대한 검증이다.

차단 실험 때 Server에서 임시 Python HTTP/8080 서비스를 실행했다. Gateway에서 같은 서비스에 `curl -I` 요청을 보내 `HTTP/1.0 200 OK`를 확인했으므로 서비스 부재와 단순 경로 부재를 배제했다. Client의 요청은 시간 초과됐고 Gateway 커널 로그에 해당 흐름의 `[UFW BLOCK]`가 남았다. 테스트 후 임시 서비스를 종료했으며 8080 포트의 LISTEN 항목이 없음을 확인했다. 재부팅 후에는 8080 차단 실험을 반복하지 않았고, 정책 유지 및 SSH 허용 경로를 확인했다. 세부 결과는 [최소 허용 정책 검증 기록](reports/minimal_policy_validation.md)에 있다.

## 장애 분석과 지속성

Client에서 Gateway에는 통신할 수 있었지만 처음에는 Server 대역으로 `ip route get`을 실행하면 `Network is unreachable`이 나왔다. Client 라우팅 테이블에 직접 연결된 `10.10.10.0/24`만 있었기 때문이다. `10.20.20.0/24 via 10.10.10.1` 경로를 추가한 뒤 `ip route get`과 Server SSH 로그인으로 복구를 확인했다. 증상·원인·조치·재검증은 [Client 경로 장애 분석](reports/client_route_troubleshooting.md)에 기록했다.

Gateway, Client, Server를 각각 재부팅한 뒤 주소·경로와 접속을 확인했다. Gateway에서는 forwarding=1, UFW active·기본 전달 deny·SSH 허용 규칙이 유지됐다. Client와 Server에서도 상대 대역으로 향하는 정적 경로가 유지됐다. 관찰 시점과 각 명령 결과는 [진행 상태](docs/PROJECT_STATUS.md)에 구분해 기록했다.

## 검증 범위

이 저장소의 결과는 격리 VM의 SSH/22 허용과 TCP/8080 차단, 정적 경로 장애 1건, 설정 지속성에 한정된다. Suricata 탐지, 패킷 캡처, nginx 운영, 기업망 적용이나 성능 검증은 수행하지 않았다.

## 문서 길잡이

- [프로젝트 핵심 요약](reports/submission_summary.md): 문제–조치–검증 요약
- [실습 환경 설계](docs/02_lab_architecture.md): VM, NIC, 주소, 경로와 정책
- [최소 허용 정책 검증 기록](reports/minimal_policy_validation.md): SSH 허용과 TCP/8080 차단 근거
- [Client 경로 장애 분석](reports/client_route_troubleshooting.md): 원인 분석과 재검증
- [변경 전 경로 출력](evidence/before/client_route_before_redacted.md) · [적용 후 정책 출력](evidence/after/network_policy_redacted.md): 실습 터미널 출력에서 식별정보를 제외한 발췌
- [진행 상태](docs/PROJECT_STATUS.md): 실제 수행 순서와 남은 제한 사항

실습용 사설 IP는 구성을 설명하는 데 필요한 범위에서만 사용한다. 관리 주소, MAC 주소, 계정명, SSH 키와 원본 로그는 공개 자료에서 제외하거나 placeholder로 대체한다. 외부 시스템이나 허가받지 않은 대상을 대상으로 테스트하지 않았다.
