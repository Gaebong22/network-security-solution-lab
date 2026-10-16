# 실습 환경과 적용 정책

## 구성 원칙

- 본인 소유의 UTM/QEMU Ubuntu VM 3대만 실습 대상으로 사용했다.
- Client와 Server를 서로 다른 UTM Host Network에 두어 구간 간 패킷이 Gateway를 지나도록 설계했다.
- 관리용 UTM Shared Network는 Gateway의 별도 NIC에 연결했다. Client·Server에 관리망 주소를 부여하지 않았다.
- 두 Host Network는 DHCP를 제공하지 않아 실습망 주소와 상대 대역 경로를 Netplan으로 설정했다.

## 실제 확인한 인터페이스와 경로

| VM | NIC | 네트워크 | IPv4 | 상대 대역 경로 |
| --- | --- | --- | --- | --- |
| Test Client | `enp0s1` | `lab-wan` | `10.10.10.10/24` | `10.20.20.0/24 via 10.10.10.1` |
| Security Gateway | `enp0s1` | `lab-wan` | `10.10.10.1/24` | 두 실습망 직결 |
| Security Gateway | `enp0s2` | `lab-lan` | `10.20.20.1/24` | 두 실습망 직결 |
| Security Gateway | `enp0s3` | UTM Shared Network | DHCP 관리 주소 | 관리망 기본 경로 |
| Protected Server | `enp0s1` | `lab-lan` | `10.20.20.10/24` | `10.10.10.0/24 via 10.20.20.1` |

```text
Mac 관리 단말 ---- SSH ----> Gateway enp0s3 (Shared Network)

Client enp0s1          Gateway enp0s1 / enp0s2          Server enp0s1
10.10.10.10/24 -- lab-wan -- 10.10.10.1/24
                                 | IPv4 forwarding + UFW
                              10.20.20.1/24 -- lab-lan -- 10.20.20.10/24
```

`lab-wan`은 인터넷 WAN이 아니라 Client 측 실습 구간 이름이다. Server에는 인터넷 기본 경로를 설정하지 않았고, Client도 이 실습에서 Server 대역으로 향하는 명시적 경로만 추가했다. Gateway의 관리망 기본 경로는 실습망 두 개 사이의 전달 경로와 구분한다.

## Gateway UFW 정책

| 조건 | 동작 | 목적 |
| --- | --- | --- |
| 관리 NIC로 들어오는 Mac 관리 주소의 TCP/22 | 입력 허용 | Gateway 관리 SSH 유지 |
| `enp0s1`로 들어온 Client의 Server TCP/22가 `enp0s2`로 나감 | 전달 허용 | 통제된 SSH 연결 검증 |
| 그 밖의 입력·전달 | 기본 deny | 명시하지 않은 흐름 제한 |
| Gateway 자체의 출력 | 기본 allow | VM의 자체 송신 허용; 엄격한 출력 정책을 구현한 것은 아님 |

IPv4 forwarding은 `net.ipv4.ip_forward=1`이며, UFW 상태는 active다. 실제 규칙·통신·로그 검증은 [정책 검증 기록](../reports/minimal_policy_validation.md)에 있다. TCP/8080 차단 결과를 다른 포트 전체의 실측 차단 결과로 일반화하지 않는다.

적용한 핵심 명령의 형태는 다음과 같다. 관리 주소는 공개용 placeholder이며 그대로 실행하는 값이 아니다.

```bash
sudo ufw allow in on enp0s3 proto tcp from '<MANAGEMENT_HOST_IP>' to any port 22
sudo ufw route allow in on enp0s1 out on enp0s2 proto tcp from 10.10.10.10 to 10.20.20.10 port 22
sudo ufw default deny routed
sudo ufw enable
```

`/etc/ufw/sysctl.conf`의 `net/ipv4/ip_forward=1`을 확인하고 `sudo sysctl -w net.ipv4.ip_forward=1` 및 `sudo ufw reload`로 적용했다. 관리 SSH 규칙을 먼저 추가하고 UFW 활성화 후 **새 SSH 세션**으로 접근 가능 여부를 확인했다. Client와 Server의 상대 대역 경로는 각 VM의 Netplan에 설정했으며, `netplan generate`와 `netplan try`로 검증했다.

## 운영상 주의

- NIC 순서를 추측하지 않고 `ip -br address`와 UTM 연결을 대조한다.
- SSH 관리 허용 규칙을 적용·확인한 다음 UFW를 활성화해 관리 접속을 유지한다.
- 원본 패킷·로그에는 주소·MAC·계정 등 식별값이 포함될 수 있으므로 그대로 공개하지 않는다.
- Suricata, nginx 및 패킷 캡처는 이 실습의 구성 요소가 아니다.
