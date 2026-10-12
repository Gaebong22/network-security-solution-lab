# Network Security Solution Lab

Ubuntu 기반의 소규모 네트워크를 설계하고 방화벽과 IDS를 구축한 뒤, 패킷과 로그를 이용해 정상 통신·차단·탐지·장애 상황을 검증하는 프로젝트다.

첫 번째 `Ubuntu Security Hardening` 프로젝트가 단일 서버 내부의 보안 통제를 다뤘다면, 이 프로젝트는 여러 시스템 사이의 트래픽 흐름과 네트워크 보안 솔루션의 구축·기술지원 과정을 다룬다.

> 현재 상태: 실습망 구축 중. Gateway·Client 연결은 확인했고, Server는 설치·로그인까지 완료했다. 라우팅·방화벽·IDS는 아직 적용하지 않았다.

## 지금 어디까지 했나

| 항목 | 상태 | 확인한 내용 |
| --- | --- | --- |
| UTM 격리망 `lab-wan`, `lab-lan` | 완료 | 두 Host Network 구성 |
| Security Gateway | 기본 구성 완료 | WAN·LAN 고정 IP, Mac에서 관리용 SSH 접속 |
| Test Client | 기본 구성 완료 | 고정 IP 설정, Gateway로 ping, Mac에서 Gateway 경유 SSH 접속 |
| Protected Server | 설치 완료 | Ubuntu 설치와 콘솔 로그인. 고정 IP·Gateway 연결은 아직 미확인 |
| Gateway 자동 업데이트 | 점검 필요 | `soft lockup` 경고 기록. 이후 SSH 접속은 됐지만 업데이트 완료·재부팅 검증 전 |
| 라우팅·방화벽·Suricata | 미착수 | 정책 적용이나 탐지 결과 없음 |

바로 다음 작업은 Server의 인터페이스와 Netplan 파일을 확인한 뒤 `lab-lan` 고정 IP를 설정하는 것이다. Gateway 자동 업데이트가 끝났는지 확인하기 전에는 강제 재부팅이나 별도 패키지 작업을 하지 않는다. 자세한 기록은 [진행 상태](docs/PROJECT_STATUS.md)에 남긴다.

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
  - nginx
  - SSH
  - system and application logs
```

세 시스템을 UTM의 격리된 가상 네트워크에서 실행한다. 현재는 Server의 네트워크 설정과 VM 간 라우팅이 남아 있다. 외부 시스템이나 허가받지 않은 대상을 스캔하거나 시험하지 않는다.

## 진행 계획

- [x] 요구사항, 주소 계획, UTM 격리망 구성
- [x] Gateway·Client 설치 및 같은 구간 통신 확인
- [x] Server 설치 및 첫 로그인
- [ ] Server 고정 IP 설정과 Gateway LAN 연결 확인
- [ ] Gateway 자동 업데이트 상태 점검 및 재부팅 후 연결 재검증
- [ ] 변경 전 네트워크·서비스·방화벽 상태 수집
- [ ] Gateway 라우팅과 최소 허용 방화벽 정책 적용
- [ ] HTTP·SSH 허용 및 미허용 포트 차단 테스트
- [ ] `tcpdump`·Wireshark로 정상·차단 패킷 비교
- [ ] Suricata 설치, 탐지 테스트와 로그 분석
- [ ] 장애 재현, 원인 분석과 복구
- [ ] 결과표·운영 가이드·면접 설명 정리 및 공개 전 검토

## 주요 결과물

- 네트워크 구성도와 주소 계획
- 방화벽 정책표
- 정상·차단·탐지 테스트 시나리오
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
