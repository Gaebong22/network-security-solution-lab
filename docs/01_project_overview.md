# 프로젝트 개요

## 실습 환경

UTM/QEMU에서 Ubuntu VM 3대(Test Client, Security Gateway, Protected Server)를 실행했다. Client와 Server를 `lab-wan`, `lab-lan`이라는 서로 다른 Host Network에 연결하고 Gateway가 두 구간 사이의 패킷을 전달하도록 구성했다. Gateway에는 Mac 관리용 Shared Network NIC를 별도로 연결했다.

주소·인터페이스·상대 대역 경로는 [실습 환경과 적용 정책](02_lab_architecture.md)에 기록했다.

## 적용한 구성

- Client와 Server에 고정 IPv4 주소와 상대 대역으로 향하는 정적 경로를 설정했다.
- Gateway에서 IPv4 forwarding을 활성화했다.
- Gateway UFW에 기본 입력·전달 deny를 적용하고 관리 SSH 입력과 Client→Server SSH/22 전달만 허용했다.
- Server에서 임시 HTTP/8080 서비스를 실행해 차단 테스트를 수행한 뒤 종료했다.

## 확인한 결과

| 항목 | 결과 |
| --- | --- |
| 허용 통신 | Client→Server SSH 로그인 성공 |
| 서비스 응답 | Gateway→Server 임시 HTTP/8080 요청에서 `HTTP/1.0 200 OK` |
| 차단 통신 | Client→Server TCP/8080 요청 시간 초과, Gateway의 동일 흐름 `[UFW BLOCK]` 기록 |
| 장애 분석 | Client의 Server행 경로 부재를 `ip route get`으로 확인하고 정적 경로 추가 후 SSH로 재검증 |
| 지속성 | Gateway·Client·Server를 각각 재부팅한 뒤 주소·경로와 해당 관리·SSH 정책 유지 확인 |

명령 결과와 차단 원인은 [정책 검증 기록](../reports/minimal_policy_validation.md), 경로 장애 조치는 [장애 분석](../reports/client_route_troubleshooting.md), 재부팅 확인 순서는 [진행 상태](PROJECT_STATUS.md)에 구분해 기록했다.

## 결과의 한계

이 결과는 본인 소유의 격리 VM에서 관찰한 SSH/22 허용과 TCP/8080 차단에 한정된다. Suricata, 패킷 캡처, nginx 운영, 실제 기업망·상용 보안 제품 적용은 수행하지 않았다. 미수행 항목을 완료 결과로 표시하지 않는다.
