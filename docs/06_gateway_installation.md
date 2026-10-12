# Security Gateway 설치 기록

## 1. 설치 대상

UTM에서 생성한 `netlab-gateway` VM에 Ubuntu Server ARM64를 설치한다.

| 항목 | 설정 |
| --- | --- |
| Architecture | ARM64/aarch64 |
| Virtualization | QEMU |
| CPU | 2 cores |
| Memory | 2 GB |
| Virtual disk | 20 GB |
| NIC 1 | Host Network `lab-wan` |
| NIC 2 | Host Network `lab-lan` |

## 2. 설치 화면별 선택과 이유

### Network configuration

두 NIC는 UTM Host Network에 연결되어 있어 DHCP 자동 설정이 실패했다. Host Network는 격리된 실습망을 제공하지만 IP address를 자동 배정하지 않는다.

설치기에서 잘못된 값을 저장하지 않고 `Continue without network`를 선택했다. 설치 완료 후 Netplan 설정 파일에 다음 고정 주소를 작성하고 실제 interface 상태를 검증할 예정이다.

```text
lab-wan: 10.10.10.1/24
lab-lan: 10.20.20.1/24
```

두 격리망에는 현재 상위 router나 DNS server가 없으므로 default gateway와 DNS는 임의로 입력하지 않는다.

### Proxy configuration

HTTP proxy를 사용하지 않으므로 공란을 유지했다. 실습에 없는 중계 server 정보를 임의로 추가하지 않는다.

### Ubuntu archive mirror

기본 Ubuntu archive mirror를 유지했다. 설치 시점에는 network가 없어 연결 검사를 수행하지 못했으며, package 설치용 관리 network를 구성한 뒤 별도로 검증할 예정이다.

### Storage configuration

새 VM 전용 20 GB virtual disk 전체와 기본 LVM 구성을 선택했다.

- EFI system partition: 약 953 MB
- `/boot`: 약 1.75 GB
- `/`: 10 GB logical volume
- LVM free space: 약 7.3 GB

LVM free space는 향후 root filesystem 공간이 부족할 때 확장할 수 있도록 남아 있다. 실제 Mac disk와 기존 VM disk는 설치 대상에 포함되지 않는다.

### Profile setup

Hostname은 `netlab-gateway`를 사용한다. Linux username과 password는 로컬 VM에만 입력하며 공개 문서와 evidence에는 기록하지 않는다.

### Ubuntu Pro

인터넷에 연결하지 않은 격리 실습 환경이며 현재 프로젝트의 필수 기능이 아니므로 `Skip Ubuntu Pro setup for now`를 선택했다. 기본 Ubuntu 보안 업데이트와 Ubuntu Pro의 추가 지원 범위는 구분해서 설명한다.

### SSH configuration

원격 관리와 접속 테스트를 위해 `Install OpenSSH server`를 선택했다. 설치 시점에는 SSH public key가 등록되지 않았으므로 password authentication을 임시로 허용했다.

설치 후에는 다음 순서로 전환한다.

1. 초기 SSH 접속 확인
2. SSH public key 등록
3. public key 접속 성공 확인
4. password authentication 비활성화
5. password 접속 거부 테스트

SSH key를 외부 서비스에서 가져오는 기능은 사용하지 않았다. 대체 접속 수단을 검증하기 전에 password authentication을 먼저 차단해 관리 경로를 잃지 않도록 순서를 정했다.

### Installation result

Ubuntu installer에서 `Installation complete!`를 확인했다. 설치 log에는 `installing openssh-server`와 `configuring cloud-init` 단계가 포함됐다.

이 시점의 확인 범위는 installation program 완료까지다. 새 operating system의 첫 부팅, 로그인, NIC 상태와 OpenSSH 동작은 재부팅 후 별도로 검증한다.

### Installation media removal

재부팅 과정에서 다음 안내가 표시됐다.

```text
Please remove the installation medium, then press ENTER:
```

이는 설치 실패가 아니라 가상 CD/DVD에 Ubuntu ISO가 여전히 연결돼 있다는 뜻이다. ISO를 분리하지 않으면 다음 부팅에서 installer로 다시 진입할 수 있으므로 UTM drive 메뉴에서 installation ISO를 eject한 뒤 Enter를 누른다.

ISO image file 자체를 삭제하는 것이 아니라 실행 중인 VM의 가상 CD/DVD 연결만 해제하는 작업이다. 첫 부팅 성공은 이 작업 후 login prompt가 나타났을 때 확인한다.

## 5. 첫 설치 시도 초기화

첫 설치 후 local account password를 확인할 수 없어 초기 구축 단계에서 VM을 삭제하고 다시 설치하기로 결정했다. 아직 firewall, routing, IDS 설정이나 수집한 evidence가 없어 password recovery 절차보다 깨끗한 재설치가 적절하다고 판단했다.

삭제 범위는 `netlab-gateway` VM과 연결된 virtual disk로 제한했다. 기존 `Kali`, `ubuntu-server`, UTM Host Network `lab-wan`과 `lab-lan`, Git repository는 유지했다.

재설치할 때는 다음 사항을 보완한다.

- VM 전용 password를 password manager에 저장
- 설치 완료 후 첫 로그인 즉시 확인
- 로그인 검증 전에는 다음 보안 설정으로 넘어가지 않음
- 성공이 확인된 결과만 완료 상태로 기록

## 3. 화면 기록 원칙

설치 단계의 화면은 선택하거나 입력하기 전에 확인한다. 공개 자료에는 다음 기준을 적용한다.

- network 방식, disk 구성과 package 선택 화면은 설명과 함께 기록
- disk identifier, MAC address 등 불필요한 장치 식별값은 가림 처리
- username, password 또는 password 입력 화면은 공개 evidence에서 제외
- 설치가 끝나지 않은 단계는 완료로 표시하지 않음

## 4. 설치 후 검증 예정 명령

```bash
hostnamectl --static
ip -brief link
ip -brief address
ip route
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
df -hT
```

- `hostnamectl --static`: 설치한 hostname 확인
- `ip -brief link`: NIC 이름과 UP/DOWN 상태 확인
- `ip -brief address`: interface별 IP address 확인
- `ip route`: routing table과 default route 확인
- `lsblk`: virtual disk, partition과 LVM 구조 확인
- `df -hT`: mount된 filesystem의 형식과 사용량 확인

검증 결과는 실제 첫 부팅 후 수집하며, 실행 전에는 성공으로 기록하지 않는다.
