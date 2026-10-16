# Network Security Solution Lab

Ubuntu 기반의 소규모 격리 네트워크를 설계하고 Gateway 방화벽의 최소 허용 정책, 정상 통신·차단 및 경로 장애를 검증하는 프로젝트다. IDS와 패킷 분석은 후속 확장 계획이다.

첫 번째 `Ubuntu Security Hardening` 프로젝트가 단일 서버 내부의 보안 통제를 다뤘다면, 이 프로젝트는 여러 시스템 사이의 트래픽 흐름과 네트워크 보안 솔루션의 구축·기술지원 과정을 다룬다.

> 현재 상태: 지원 전 최소 실습 범위의 구성·검증 완료. Client–Server 정적 경로, Gateway IPv4 forwarding과 UFW 최소 허용 정책을 적용했다. Client→Server SSH 허용과 HTTP/8080 차단을 확인했고, 세 VM의 재부팅 후 주소·경로·Gateway 방화벽 설정 및 SSH 접속을 재확인했다. IDS는 지원 후 확장 범위다.

## 지금 어디까지 했나

| 항목 | 상태 | 확인한 내용 |
| --- | --- | --- |
| UTM 격리망 `lab-wan`, `lab-lan` | 완료 | 두 Host Network 구성 |
| Security Gateway | 재부팅 검증 완료 | WAN·LAN·관리 주소, forwarding=1, UFW active·최소 허용 규칙, Mac 관리 SSH 유지 |
| Test Client | 재부팅 검증 완료 | `10.10.10.10/24`, Server행 경로 `via 10.10.10.1`, Mac에서 Gateway 경유 SSH 접속 |
| Protected Server | 재부팅 검증 완료 | `10.20.20.10/24`, Client행 경로 `via 10.20.20.1`, Mac에서 Gateway 경유 SSH 접속 |
| Gateway 자동 업데이트 | 이전 작업 종료 확인 | `dpkg --audit` 빈 출력, 실패한 서비스 0개. 적용 가능한 업데이트 22개는 별도 관리 필요 |
| 라우팅·방화벽 | 최소 범위 검증 완료 | Client→Server SSH 전달, HTTP/8080 차단 로그, Gateway 재부팅 후 SSH 전달 확인 |
| Suricata IDS | 지원 후 확장 | 설치·탐지 결과를 현재 포트폴리오 성과로 주장하지 않음 |

지원용 최소 범위의 실제 검증 결과는 [정책 검증 기록](reports/minimal_policy_validation.md)과 [경로 장애 분석](reports/client_route_troubleshooting.md)에 정리했다. 제출 전에는 공개 자료를 최종 검토하고 이력서·자기소개서에 연결할 핵심 성과를 선별한다. 자세한 기록은 [진행 상태](docs/PROJECT_STATUS.md)에 남긴다.

## 목표

- TCP/IP, subnet, routing과 gateway의 동작을 실제 트래픽으로 확인한다.
- Linux firewall을 이용해 최소 허용 정책을 설계한다.
- Suricata IDS를 설치하고 통제된 테스트 트래픽을 탐지한다.
- `tcpdump`와 Wireshark로 정상·차단·탐지 패킷을 비교한다.
- 연결 장애를 network, firewall, service 계층으로 나누어 진단한다.
- 구축 절차, 정책표, 테스트 결과와 장애 보고서를 기술지원 문서 형식으로 남긴다.

## 구성과 목표

```text
Test Client
    |
    | lab-wan
    v
Security Gateway / IDS
  - Linux routing
  - nftables or UFW
  - Suricata
  - tcpdump
    |
    | lab-lan
    v
Protected Server
  - SSH
  - 임시 HTTP 테스트 서비스 (nginx는 후속 확장)
  - system and application logs
```

세 시스템을 UTM의 격리된 가상 네트워크에서 실행한다. 지원 전 최소 범위는 정적 경로, Gateway 최소 허용 정책, 허용·차단 검증 및 장애 사례 1건이다. 패킷 분석과 Suricata는 지원 후 확장한다. 외부 시스템이나 허가받지 않은 대상을 스캔하거나 시험하지 않는다.

## 진행 계획

- [x] 요구사항, 주소 계획, UTM 격리망 구성
- [x] Gateway·Client 설치 및 같은 구간 통신 확인
- [x] Server 설치 및 첫 로그인
- [x] Server 고정 IP 설정과 Gateway LAN 연결 확인
- [x] Gateway 이전 자동 업데이트 작업 종료와 패키지 상태 점검
- [ ] 변경 전 네트워크·서비스·방화벽 상태 수집
- [x] Gateway 라우팅과 최소 허용 방화벽 정책 적용
- [x] SSH 허용 및 미허용 TCP/8080 차단 원인 검증
- [x] 세 VM 재부팅 후 주소·경로·Gateway 정책 및 관리·전달 SSH 재검증
- [ ] `tcpdump`·Wireshark로 정상·차단 패킷 비교
- [ ] Suricata 설치, 탐지 테스트와 로그 분석
- [ ] 장애 재현, 원인 분석과 복구
- [ ] 결과표·운영 가이드·면접 설명 정리 및 공개 전 검토

## 주요 결과물

- 네트워크 구성도와 주소 계획
- 방화벽 정책표
- 정상·차단·탐지 테스트 시나리오
- [최소 허용 정책 검증 기록](reports/minimal_policy_validation.md)
- [Client 경로 장애 분석](reports/client_route_troubleshooting.md)
- Wireshark·tcpdump 패킷 분석
- Suricata alert 분석
- 장애 분석 보고서
- 구축·운영·복구 가이드
- 변경 전후 evidence

## 저장소 구성

```text
config-examples/   공개 가능한 설정 예시
docs/              설계, 구축, 테스트와 트러블슈팅 문서
evidence/
  before/          보안 정책 적용 전 실제 결과
  after/           적용 후 실제 결과
reports/           최종 보고서와 장애 보고서
scripts/           반복 점검 및 로그 요약 스크립트
```

## 공개 정보 처리

실습용 격리망 주소는 설계를 설명할 때만 사용한다. 실제 관리 주소, MAC 주소, 계정명, SSH key 등 환경 식별값과 원본 로그는 공개 전에 검토하고 필요하면 다음 placeholder로 치환한다.

- `<CLIENT_IP>`
- `<GATEWAY_WAN_IP>`
- `<GATEWAY_LAN_IP>`
- `<SERVER_LAN_IP>`
- `<LAB_WAN_SUBNET>`
- `<LAB_LAN_SUBNET>`
- `<REDACTED_MAC>`

테스트 결과는 실제 명령 출력으로만 기록하며, 수행하지 않은 항목은 PASS로 표시하지 않는다.
