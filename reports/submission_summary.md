# 프로젝트 핵심 요약 — Network Security Solution Lab

| 항목 | 내용 |
| --- | --- |
| 형태 | 개인 실습, UTM/QEMU의 Ubuntu VM 3대와 격리망 2개 |
| 목적 | 네트워크 보안 정책의 구축·검증과 연결 장애 원인 분석 |
| 담당 | 네트워크 주소·경로 설계, Gateway UFW 정책 적용, 통신·로그 검증, 장애 기록 |
| 완료 시점 | 2026-10-16, 현재 검증 범위 |

## 문제와 조치

Client와 Server를 다른 서브넷으로 분리하고 Gateway를 경유하도록 구성했다. Client에서 Server로 가는 경로가 없을 때 `ip route get`은 `Network is unreachable`을 반환했다. Client의 정적 경로 `10.20.20.0/24 via 10.10.10.1`을 추가하고 Server의 반환 경로를 확인한 뒤, 실제 SSH 로그인으로 연결을 재검증했다.

Gateway에서는 IPv4 forwarding을 활성화하고 UFW의 기본 입력·전달을 deny로 설정했다. Mac의 Gateway 관리 SSH와 Client→Server SSH/22만 명시적으로 허용했다. 허용 흐름은 실제 로그인으로 확인했다.

## 검증 결과

| 검증 항목 | 관찰 결과 | 의미 |
| --- | --- | --- |
| Client→Server SSH/22 | 로그인 성공 | 허용된 전달 경로 작동 |
| Gateway→Server 임시 HTTP/8080 | `HTTP/1.0 200 OK` | 테스트 서비스 응답 확인 |
| Client→Server HTTP/8080 | 연결 시간 초과 | 미허용 흐름의 접속 실패 |
| Gateway 로그 | 같은 시각 Client→Server TCP/8080 SYN의 `[UFW BLOCK]` 3건 | 방화벽 전달 단계가 차단 지점임을 확인 |
| 재부팅 후 | Gateway의 forwarding·UFW 규칙과 Client→Server SSH 유지; Client·Server 개별 재부팅 후 각 주소·경로와 Mac 경유 SSH 확인 | 확인한 범위의 설정 지속성 검증 |

임시 HTTP 서버는 검증 후 종료했고 TCP/8080 LISTEN 항목이 없음을 확인했다. 8080 차단 실험은 재부팅 전 결과이며, 재부팅 후에는 정책 유지와 SSH 경로를 확인했다. [실제 정책 검증](minimal_policy_validation.md)과 [경로 장애 분석](client_route_troubleshooting.md)에 상세 근거를 분리했다.

## 핵심 설명

> UTM 격리망에 Client–Gateway–Server 3-VM 구조를 구축하고 양방향 정적 경로와 UFW 최소 허용 정책을 적용했습니다. SSH 허용과 HTTP/8080 차단을 실제 접속·서버 응답·Gateway UFW 로그로 교차 검증했으며, Client의 경로 부재 장애를 `ip route get`으로 진단해 복구했습니다. 세 VM의 재부팅 후 주소·경로와 Gateway 정책 유지도 확인했습니다.

## 주장하지 않는 범위

Suricata 탐지, 패킷 캡처 분석, nginx 운영, 상용 보안 제품 직접 사용 또는 기업 환경 구축은 수행하지 않았다. 이 실습은 격리 VM에서의 구축·검증·문서화 사례이며 실제 운영 환경의 보안 수준이나 성능을 증명하지 않는다.
