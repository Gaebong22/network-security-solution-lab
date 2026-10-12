# UTM 격리 네트워크 구성 학습 기록

## 1. 이번 단계에서 한 일

UTM에 서로 분리된 가상 네트워크 두 개를 만들고, Security Gateway VM의 두 network interface에 각각 연결했다.

```text
Test Client
    |
    | lab-wan
    v
Security Gateway
    |
    | lab-lan
    v
Protected Server
```

- `lab-wan`: 테스트 트래픽이 Gateway로 들어오는 외부 역할의 구간
- `lab-lan`: Gateway가 보호하는 내부 서버 구간

여기서 WAN과 LAN은 실제 인터넷과 사내망이 아니라, 역할을 구분하기 위해 붙인 실습용 이름이다.

## 2. UTM에서 Host Network를 만든 순서

1. UTM 상단 메뉴에서 `UTM → 설정…`을 열었다.
2. 왼쪽에서 `네트워크`를 선택했다.
3. `Host Networks`의 `추가` 버튼을 눌렀다.
4. 첫 번째 network 이름을 `lab-wan`으로 변경했다.
5. 다시 `추가`를 눌러 두 번째 network를 만들었다.
6. 두 번째 이름을 `lab-lan`으로 변경하고 Enter를 눌러 저장했다.
7. 두 항목에 서로 다른 UUID가 자동 생성됐는지 확인했다.

UUID는 network를 내부적으로 구분하는 고유 식별자다. 프로젝트 공개 문서에는 실제 UUID를 기록하지 않는다.

## 3. 왜 Shared Network가 아닌 Host Network인가

### Shared Network

- Mac의 network 연결을 통해 VM이 외부 network에 접근할 수 있다.
- UTM이 NAT와 DHCP를 제공한다.
- package 설치에는 편리하지만 실습 구간을 명확히 분리하기 어렵다.

### Host Network

- 여러 QEMU VM을 같은 격리된 network에 연결할 수 있다.
- 외부 network와 분리된 실습망을 만들 수 있다.
- DHCP가 없으므로 각 VM에서 IP address를 직접 설정해야 한다.

이번 프로젝트는 routing과 firewall을 직접 구성해야 하므로 Host Network를 선택했다.

## 4. 왜 network를 두 개 만드는가

Gateway가 packet을 한 구간에서 받아 다른 구간으로 전달하려면 서로 다른 network에 연결된 interface가 두 개 필요하다.

```text
NIC 1 → lab-wan → 10.10.10.1/24
NIC 2 → lab-lan → 10.20.20.1/24
```

두 구간이 같은 subnet이면 Gateway를 거치지 않고 같은 network 안에서 직접 통신할 수 있다. 서로 다른 subnet으로 분리해야 packet이 Gateway의 routing과 firewall 정책을 통과한다.

## 5. Security Gateway VM 생성 과정

UTM에서 다음과 같이 생성했다.

| 항목 | 설정 | 이유 |
| --- | --- | --- |
| Name | `netlab-gateway` | VM의 역할을 이름으로 구분 |
| Architecture | ARM64 | Apple M4와 Ubuntu ARM64 image 사용 |
| Engine | QEMU | UTM Host Network 사용 |
| CPU | 2 cores | Suricata와 packet 분석 실행 여유 확보 |
| Memory | 2 GB | 기능 수행과 호스트 자원 사이의 균형 |
| Disk | 20 GB | Ubuntu, log와 Suricata rule 저장 |
| NIC 1 | `lab-wan` | Test Client 측 연결 |
| NIC 2 | `lab-lan` | Protected Server 측 연결 |

Ubuntu Server 26.04.1 ARM64 ISO를 설치 image로 연결했으며, 아직 운영체제 설치는 완료되지 않았다.

## 6. 알아둘 용어

### NIC

Network Interface Card의 약자다. 물리 장비에서는 LAN card이고 VM에서는 가상 network 장치다. Linux에서는 `enp0s1`, `enp0s2` 같은 interface 이름으로 표시될 수 있다.

### IP address

Network에서 host 또는 interface를 구분하는 논리 주소다. Gateway에는 NIC가 두 개이므로 IP address도 구간별로 하나씩 필요하다.

### Subnet

하나의 network 범위를 의미한다. `/24`는 IPv4 32bit 중 앞 24bit가 network 부분이라는 뜻이다.

### Gateway

현재 subnet 밖의 목적지로 packet을 전달하는 장치다. 이 프로젝트의 Security Gateway는 routing뿐 아니라 firewall과 IDS 역할도 수행한다.

### Routing

목적지 IP address를 확인해 packet을 어느 interface로 보낼지 결정하는 과정이다.

### Firewall

source, destination, protocol과 port 등의 조건으로 통신을 허용하거나 차단한다.

### IDS

Intrusion Detection System의 약자다. packet이나 traffic pattern을 분석해 의심스러운 행위를 탐지하고 alert를 남긴다. 이 프로젝트에서는 Suricata를 사용한다.

## 7. 설치 후 직접 확인할 명령

```bash
ip -brief link
ip -brief address
ip route
```

- `ip`: Linux network 상태와 설정을 다루는 명령
- `-brief`: 결과를 interface별 한 줄로 간단히 표시
- `link`: NIC 이름과 UP/DOWN 상태 확인
- `address`: 각 NIC에 할당된 IP address 확인
- `route`: 목적지별 packet 전달 경로 확인

Gateway 설치 후에는 명령 결과를 보면서 어떤 interface가 `lab-wan`이고 어떤 interface가 `lab-lan`인지 직접 구분한다. interface 순서를 추측해서 IP를 입력하지 않는다.

## 8. 면접에서 설명하는 방식

> 외부 테스트 구간과 보호 서버 구간을 분리하기 위해 UTM Host Network를 두 개 만들었습니다. Gateway에 NIC 두 개를 연결하고 서로 다른 subnet을 배정해 모든 구간 간 통신이 Gateway를 통과하도록 설계했습니다. 이를 통해 routing, firewall 정책과 Suricata 탐지를 같은 packet 흐름에서 검증할 수 있도록 구성했습니다.
