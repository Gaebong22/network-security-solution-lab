# Network Security Solution Lab 진행 상태

## 현재 위치

격리망 2개와 VM 3대의 기본 연결을 확인했다. Client와 Server에는 상대 대역으로 향하는 정적 경로를 설정했고 Gateway의 IPv4 forwarding과 UFW 기본 차단·SSH 전달 허용 정책을 적용했다. Client→Server SSH 로그인에 성공했다. Gateway→Server 임시 HTTP/8080 요청은 `200 OK`, Client→Server 동일 요청은 시간 초과였으며 Gateway의 `[UFW BLOCK]` 로그에서 해당 TCP/8080 SYN 차단을 확인했다. Gateway의 이전 자동 업데이트 작업은 종료됐으나 로그인 시점에 적용 가능한 업데이트 22개가 남아 있다.

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

## Gateway 자동 업데이트 점검 기록

- 이전 부팅의 `journalctl -b -k -g 'soft lockup'`에서 `unattended-upgr` 관련 CPU 정체 경고를 확인했다. 원인을 UTM이나 macOS 절전으로 단정하지 않는다.
- 2026-10-16 점검에서 Mac→Gateway SSH 접속에 성공했고, 실행 중인 패키지 작업 목록에는 `dpkg`·`dracut`이 없었다. 자동 업데이트의 dpkg 로그는 `Log ended: 2026-10-16 06:43:55`로 끝났다.
- `sudo dpkg --audit`는 빈 출력, `systemctl --failed --no-pager`는 실패한 유닛 0개였다. 현재 실행 커널은 앞서 기록한 `7.0.0-30`에서 `7.0.0-38`로 바뀌었으나, 재부팅 과정 전체를 별도로 검증했다고 기록하지 않는다.
- 로그인 메시지에는 적용 가능한 업데이트 22개가 표시됐다. 따라서 패키지 작업 종료와 패키지 데이터베이스 정상 확인을 모든 업데이트 적용 완료와 혼동하지 않는다.

## 다음 작업

1. 재부팅 후 설정 유지와 관리 SSH 접속을 검증한다.
2. 공개용 증거에서 MAC 주소·임시 포트·계정 정보가 제거됐는지 점검한다.
3. 최소 범위 완료 여부를 판단한다.

실제 명령 출력과 스크린샷을 공개 저장소에 추가할 때에는 주소·계정·키·토큰 등 민감정보를 먼저 검토한다.
