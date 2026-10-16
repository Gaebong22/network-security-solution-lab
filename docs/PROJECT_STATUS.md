# Network Security Solution Lab 진행 상태

## 현재 위치

격리망 2개와 VM 3대의 연결, Client·Server의 상대 대역 정적 경로, Gateway의 IPv4 forwarding과 UFW 기본 차단·SSH 전달 허용 정책을 검증했다. Client→Server SSH 로그인에 성공했다. Gateway→Server 임시 HTTP/8080 요청은 `200 OK`, Client→Server 동일 요청은 시간 초과였으며 Gateway의 `[UFW BLOCK]` 로그에서 해당 TCP/8080 SYN 차단을 확인했다. Gateway 재부팅 후에는 관리 SSH와 Client→Server SSH가 다시 성공했다. Client·Server도 각각 재부팅한 뒤 주소·상대 대역 경로와 Mac에서 Gateway 경유 SSH 접속을 확인했다.

## 확인한 구성

| VM | 인터페이스 | 연결 | 주소 | 확인 결과 |
| --- | --- | --- | --- | --- |
| `netlab-gateway` | `enp0s1` | `lab-wan` | `10.10.10.1/24` | 주소·직결 경로 확인 |
| `netlab-gateway` | `enp0s2` | `lab-lan` | `10.20.20.1/24` | 주소·직결 경로 확인 |
| `netlab-gateway` | `enp0s3` | UTM Shared Network | DHCP(관리용) | Mac에서 SSH 접속 확인 |
| `netlab-client` | `enp0s1` | `lab-wan` | `10.10.10.10/24` | Gateway WAN 주소로 ping 3/3 성공 |
| `netlab-server` | `enp0s1` | `lab-lan` | `10.20.20.10/24` | Gateway LAN ping 3/3, Mac→Gateway→Server SSH 확인 |

- Gateway와 Client의 Netplan 설정은 `netplan generate`로 문법을 확인하고 `netplan try`로 적용했다.
- Mac에서 `ssh -J <USER>@<GATEWAY_MGMT_IP> <USER>@<CLIENT_IP>` 방식으로 Client에 접속했다.
- Server의 Netplan은 `netplan generate`로 문법을 확인하고 `netplan try`로 적용했다. `10.10.10.0/24` 경로의 다음 홉은 Gateway LAN 주소다.
- Server의 `ssh.socket`은 active다. Mac에서 Gateway를 jump host로 사용해 Server에 SSH 접속했다.
- Client에서 변경 전 `ip route get 10.20.20.10`은 `Network is unreachable`이었다. `10.20.20.0/24 via 10.10.10.1` 경로를 추가한 뒤 `ip route get` 결과가 Gateway를 가리켰고, Client→Server SSH 로그인이 성공했다.
- Gateway의 `net.ipv4.ip_forward`는 `1`이며 `/etc/ufw/sysctl.conf`에 IPv4 forwarding을 설정했다. UFW는 active, 기본 입력·전달 deny, 출력 allow다.
- UFW에는 Mac 관리 주소에서 `enp0s3`의 SSH를 허용하는 입력 규칙과 Client→Server SSH/22만 허용하는 전달 규칙이 있다. UFW 활성화 후 Mac에서 Gateway로 새 SSH 접속에 성공했다.
- Server에서 임시 Python HTTP/8080 서비스를 실행한 상태로 Gateway→Server `curl -I`는 `HTTP/1.0 200 OK`, Client→Server `curl -I`는 `curl: (28) Connection timed out after 3002 milliseconds`였다. Gateway 커널 로그에는 같은 시각 `IN=enp0s1 OUT=enp0s2 SRC=10.10.10.10 DST=10.20.20.10 PROTO=TCP DPT=8080 SYN`인 `[UFW BLOCK]`가 3건 기록됐다. 공개 자료에서는 MAC 주소와 임시 포트를 제거한다.
- 테스트 후 Server에서 임시 Python HTTP 서버를 `Ctrl+C`로 종료했고, `ss -lnt '( sport = :8080 )'` 출력에 LISTEN 항목이 없음을 확인했다.

## 재부팅 후 지속성 검증

- Gateway 재부팅 후 `ip -br address`에 `enp0s1=10.10.10.1/24`, `enp0s2=10.20.20.1/24`, 관리 NIC의 DHCP 주소가 표시됐다. `ip route`에는 두 실습망 직결 경로와 관리망 기본 경로가 있었다.
- Gateway의 `net.ipv4.ip_forward=1`, UFW는 active이며 기본 입력·전달 deny와 Mac 관리 SSH 입력 허용·Client→Server SSH/22 전달 허용 규칙이 유지됐다. Mac→Gateway 새 SSH 접속에 성공했다.
- Gateway 재부팅 후 Client에서 `ip route get 10.20.20.10`은 `via 10.10.10.1`을 반환했고, Client→Server SSH 로그인에 성공했다. Server에서 `ip route get 10.10.10.10`은 `via 10.20.20.1`을 반환했다.
- Client를 별도로 재부팅한 뒤 Mac→Gateway→Client SSH 접속에 성공했고 Server행 경로 `via 10.10.10.1`이 유지됐다.
- Server를 별도로 재부팅한 뒤 Mac→Gateway→Server SSH 접속에 성공했다. `ip -br address`에 `10.20.20.10/24`, `ip route get 10.10.10.10`에 `via 10.20.20.1`이 표시됐다.

## 기록 범위

이 문서는 수행한 네트워크 구성·정책 검증 결과만 기록한다. Suricata 탐지나 패킷 분석은 수행하지 않았다. 공개용 출력 발췌에서는 MAC 주소·계정·임시 포트를 제외했다.
