# UTM 초기 구성 절차

## 1. 왜 QEMU와 Host Network를 사용하는가

이 프로젝트에는 서로 다른 두 개의 격리망을 여러 VM이 공유해야 한다.

- `lab-wan`: Test Client와 Security Gateway 연결
- `lab-lan`: Security Gateway와 Protected Server 연결

UTM의 Host Network는 여러 QEMU VM을 같은 격리망에 연결할 수 있다. DHCP는 제공하지 않으므로 Ubuntu에서 IP address를 직접 설정한다. Apple Virtualization backend에서는 이 기능을 사용할 수 없으므로 새 VM은 QEMU backend로 만든다.

## 2. Host Network 생성

UTM에서 다음 순서로 두 개의 Host Network를 만든다.

1. UTM을 실행한다.
2. 메뉴에서 `UTM → Settings…`를 연다.
3. `Network`의 `Host Networks`로 이동한다.
4. 첫 번째 격리망을 추가하고 구분 가능한 이름을 `lab-wan`으로 정한다.
5. 두 번째 격리망을 추가하고 이름을 `lab-lan`으로 정한다.
6. 두 network의 UUID가 서로 다른지 확인한다.

UTM 버전에 따라 화면에 이름 대신 UUID 중심으로 표시될 수 있다. 이 경우 어느 UUID가 WAN과 LAN인지 로컬 메모에 기록한다. UUID나 MAC address는 공개 evidence에 포함하지 않는다.

## 3. VM 생성 순서

### Security Gateway

- Name: `netlab-gateway`
- Architecture: ARM64/aarch64
- Backend: QEMU virtualization
- CPU: 2 cores
- Memory: 2048 MB
- Disk: 20 GB
- Network 1: Host Network `lab-wan`
- Network 2: Host Network `lab-lan`
- 설치용 임시 Network: package 설치가 필요하면 Shared Network를 추가하고 이후 제거 여부를 검토

Gateway에는 interface가 두 개 이상 표시되므로 MAC address 또는 UTM의 device 순서를 이용해 WAN과 LAN을 구분한다. 추측으로 설정하지 않는다.

### Test Client

- Name: `netlab-client`
- CPU: 1 core
- Memory: 1024 MB
- Disk: 12 GB
- Network: Host Network `lab-wan`

### Protected Server

- Name: `netlab-server`
- CPU: 1~2 cores
- Memory: 1536 MB
- Disk: 16 GB
- Network: Host Network `lab-lan`

## 4. Ubuntu 설치 시 기준

- Ubuntu Server ARM64를 사용한다.
- 서로 다른 hostname을 지정한다.
- 동일한 IP를 여러 VM에 입력하지 않는다.
- SSH server 설치 여부는 기록한다.
- 실제 사용자 이름과 비밀번호는 evidence에 저장하지 않는다.
- 설치가 끝나기 전에는 firewall 또는 Suricata를 설정하지 않는다.

## 5. 첫 부팅 후 수집할 명령

각 VM에서 다음 명령을 실행한다.

```bash
hostnamectl --static
ip -brief link
ip -brief address
ip route
```

명령의 의미:

- `hostnamectl --static`: 현재 시스템의 고정 hostname을 출력한다.
- `ip`: Linux network 설정과 상태를 조회하는 명령이다.
- `-brief`: 긴 정보를 한 줄씩 간단히 표시한다.
- `link`: network interface의 존재와 UP/DOWN 상태를 확인한다.
- `address`: interface에 할당된 IPv4·IPv6 address를 확인한다.
- `ip route`: 목적지 network로 packet을 보낼 경로와 default gateway를 확인한다.

이 단계에서는 설정을 변경하지 않는다. 먼저 각 VM의 interface 이름과 UTM network 연결이 설계와 일치하는지 확인한다.
