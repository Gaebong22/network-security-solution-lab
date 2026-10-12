# Network Security Solution Lab 진행 상태

## 현재 위치

격리망 2개와 Gateway·Client VM의 기본 연결을 확인했다. `netlab-server`는 Ubuntu 설치와 콘솔 로그인을 마쳤고, 고정 IP 설정은 아직이다. Gateway에서는 자동 업데이트 중 `soft lockup` 경고가 기록되었으나, 이후 SSH 접속이 가능했고 `systemctl --failed` 결과는 0개였다. 자동 업데이트가 완전히 종료됐는지는 아직 확인하지 못했다. 이 문제를 해결했다고 기록하지 않는다.

## 확인한 구성

| VM | 인터페이스 | 연결 | 주소 | 확인 결과 |
| --- | --- | --- | --- | --- |
| `netlab-gateway` | `enp0s1` | `lab-wan` | `10.10.10.1/24` | 주소·직결 경로 확인 |
| `netlab-gateway` | `enp0s2` | `lab-lan` | `10.20.20.1/24` | 주소·직결 경로 확인 |
| `netlab-gateway` | `enp0s3` | UTM Shared Network | DHCP(관리용) | Mac에서 SSH 접속 확인 |
| `netlab-client` | `enp0s1` | `lab-wan` | `10.10.10.10/24` | Gateway WAN 주소로 ping 3/3 성공 |
| `netlab-server` | 미확인 | `lab-lan` | 고정 IP 설정 전 | Ubuntu 설치·콘솔 로그인 확인 |

- Gateway와 Client의 Netplan 설정은 `netplan generate`로 문법을 확인하고 `netplan try`로 적용했다.
- Mac에서 `ssh -J <USER>@<GATEWAY_MGMT_IP> <USER>@<CLIENT_IP>` 방식으로 Client에 접속했다.
- Gateway의 `net.ipv4.ip_forward`는 마지막 확인 당시 `0`이었다. 아직 두 격리망 사이 라우팅·방화벽을 구성하지 않았다.
- `netlab-server`의 인터페이스 이름, 주소 및 Gateway LAN 연결은 아직 확인하지 않았다.

## Gateway 자동 업데이트 점검 기록

- `journalctl -b -k -g 'soft lockup'`에서 `unattended-upgr` 관련 CPU 정체 경고를 확인했다.
- 점검 당시 Mac에서 Gateway로 ping과 SSH가 성공했고, `systemctl --failed --no-pager`는 실패한 유닛 0개를 표시했다.
- 자동 업데이트 로그에는 패키지 설치와 `dracut`의 initrd 생성 단계가 기록되었다. 해당 `dpkg`·`dracut` 프로세스는 이후 종료됐지만 `unattended-upgr`의 하위 프로세스는 남아 있었다.
- 따라서 업데이트 완료, 패키지 상태 정상, 재부팅 후 정상 동작은 **미검증**이다. 장시간 경고의 원인을 UTM이나 macOS 절전으로 단정하지 않는다.

## 다음 작업

1. `netlab-server`의 인터페이스를 확인하고 `10.20.20.10/24`를 설정한다.
2. Gateway에서 패키지 작업이 종료됐는지만 확인한다. 진행 중이면 중단·강제 재부팅·별도 `apt` 실행을 피한다.
3. 작업 종료 후 `sudo dpkg --audit`로 불완전한 패키지가 없는지 확인한다.
4. 정상 종료가 확인되면 Gateway를 일반 재부팅하고 SSH, IP 주소, 경로, Client→Gateway ping을 재검증한다.
5. 변경 전 네트워크·서비스·방화벽 상태를 수집한 다음, 라우팅과 방화벽 정책 실습으로 넘어간다.

실제 명령 출력과 스크린샷을 공개 저장소에 추가할 때에는 주소·계정·키·토큰 등 민감정보를 먼저 검토한다.
