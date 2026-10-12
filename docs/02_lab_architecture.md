# 실습 환경 설계

## 1. 설계 원칙

- 기존 `Ubuntu Security Hardening` 서버와 분리한다.
- 모든 테스트는 UTM 내부의 격리된 가상 네트워크에서 수행한다.
- 여러 VM이 같은 격리망을 공유할 수 있도록 UTM의 QEMU backend와 Host Network를 사용한다.
- 인터넷 연결은 package 설치가 필요한 구간에서만 사용한다.
- 변경 전 VM snapshot 또는 설정 백업을 만든다.
- gateway를 우회해 protected server에 직접 접근하지 못하도록 구성한다.

## 2. VM 구성안

| 시스템 | 역할 | 권장 자원 | Network interface |
| --- | --- | --- | --- |
| Test Client | 정상·차단·탐지 트래픽 생성 | 1 CPU, 1 GB RAM | `lab-wan` |
| Security Gateway | routing, firewall, Suricata, packet capture | 2 CPU, 2 GB RAM | `lab-wan`, `lab-lan` |
| Protected Server | nginx, SSH, system/application log | 1~2 CPU, 1.5 GB RAM | `lab-lan` |

호스트 메모리는 16 GB이므로 세 VM을 동시에 실행할 수 있도록 합계 약 4.5 GB로 시작한다. 실제 부하를 확인한 뒤 필요한 경우 조정한다.

UTM의 Host Network는 여러 QEMU VM이 같은 격리망을 공유하게 만들 수 있지만 DHCP를 제공하지 않는다. 따라서 이 프로젝트에서는 각 VM에 주소를 수동으로 설정한다. `Emulated VLAN`은 VM마다 별도의 VLAN을 생성하므로 이 구성의 VM 간 공유망 용도로 사용하지 않는다.

## 3. 논리 구성

```text
                 lab-wan                       lab-lan

+-------------+          +--------------------------------+          +------------------+
| Test Client |----------| Security Gateway / IDS         |----------| Protected Server |
| Ubuntu      |          | NIC 1: WAN / NIC 2: LAN        |          | nginx, SSH       |
+-------------+          | routing, firewall, Suricata    |          +------------------+
                         +--------------------------------+
```

## 4. 주소 계획

초기 실습 주소는 다음과 같이 시작한다. 실제 evidence를 공개할 때는 아래 placeholder로 치환한다.

| 구간 | 항목 | 초기 실습 주소 | 공개 표기 |
| --- | --- | --- | --- |
| WAN | Test Client | `10.10.10.10/24` | `<CLIENT_IP>` |
| WAN | Gateway WAN | `10.10.10.1/24` | `<GATEWAY_WAN_IP>` |
| LAN | Gateway LAN | `10.20.20.1/24` | `<GATEWAY_LAN_IP>` |
| LAN | Protected Server | `10.20.20.10/24` | `<SERVER_LAN_IP>` |
| WAN | Subnet | `10.10.10.0/24` | `<LAB_WAN_SUBNET>` |
| LAN | Subnet | `10.20.20.0/24` | `<LAB_LAN_SUBNET>` |

초기 gateway는 Test Client에서 `10.10.10.1`, Protected Server에서 `10.20.20.1`이 된다. 외부 인터넷 연결은 별도의 임시 Shared Network interface를 사용할 때만 허용하고, 구축 완료 후 제거 여부를 검토한다.

## 5. 예정 통신 정책

| Source | Destination | Service | 초기 정책 | 목적 |
| --- | --- | --- | --- | --- |
| Test Client | Protected Server | HTTP/80 | Allow | 정상 웹 연결 검증 |
| Test Client | Protected Server | SSH/22 | Allow 또는 관리 source 제한 | 관리 연결 검증 |
| Test Client | Protected Server | TCP/8080 | Deny | 미허용 포트 차단 검증 |
| Test Client | Protected Server | ICMP | 단계별 결정 | network 기본 진단 |
| Protected Server | Internet | package repository | 설치 단계에서만 허용 | package 설치·업데이트 |

이 표는 설계 초안이며 실제 적용 전 운영 목적과 테스트 범위를 다시 검토한다.

## 6. 확인해야 할 위험

- 두 network interface의 순서를 잘못 지정하면 WAN과 LAN이 뒤바뀔 수 있다.
- IP forwarding만 활성화하고 firewall 정책을 적용하지 않으면 불필요한 통신이 허용될 수 있다.
- Suricata가 잘못된 interface를 감시하면 패킷이 지나가도 alert가 생성되지 않는다.
- VM 복제 시 hostname, machine-id, SSH host key와 고정 IP가 충돌할 수 있다.
- packet capture에는 IP, hostname, cookie 또는 request payload가 포함될 수 있으므로 원본을 바로 공개하지 않는다.
