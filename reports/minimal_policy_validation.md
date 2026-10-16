# 최소 허용 네트워크 정책 검증 기록

검증일: 2026-10-16 (VM 출력은 UTC)
범위: UTM 격리망의 Test Client, Security Gateway, Protected Server

## 정책과 사전 조건

| 흐름 | 정책 | 확인 상태 |
| --- | --- | --- |
| Mac 관리 주소 → Gateway 관리 NIC TCP/22 | 입력 허용 | UFW 활성화 후 새 SSH 접속 성공 |
| Client → Gateway → Server TCP/22 | 전달 허용 | Client에서 Server SSH 로그인 성공 |
| Client → Gateway → Server TCP/8080 | 전달 기본 차단 | Client 요청 시간 초과 및 Gateway의 `[UFW BLOCK]` 로그 확인 |

- Gateway의 `net.ipv4.ip_forward=1`, UFW 상태는 active이며 기본 입력·전달 정책은 deny다.
- Client에는 Server 대역으로 향하는 `via <GATEWAY_WAN_IP>` 경로, Server에는 Client 대역으로 향하는 `via <GATEWAY_LAN_IP>` 경로가 있다.
- Client에서 `ip route get <SERVER_LAN_IP>`는 Gateway WAN 주소를 다음 홉으로 반환했다.
- Server의 임시 HTTP 서비스는 빈 디렉터리를 대상으로 TCP/8080에서 실행했다. 상시 운영 서비스나 nginx 설치 결과가 아니다.

## 실제 결과

| 테스트 | 명령 또는 확인 | 관찰 결과 | 해석 |
| --- | --- | --- | --- |
| 관리 접속 유지 | Mac에서 Gateway로 새 SSH 연결 | 로그인 성공 | UFW 활성화 후 관리 경로 유지 |
| 허용 통신 | Client에서 Server로 SSH | 비밀번호 인증 후 로그인 성공 | TCP/22 전달 경로 작동 |
| 서비스 존재 확인 | Gateway에서 `curl -I --connect-timeout 3 --max-time 6 http://<SERVER_LAN_IP>:8080/` | `HTTP/1.0 200 OK`; Server에 Gateway LAN 주소의 `HEAD /` 기록 | Server의 HTTP/8080 서비스가 응답함 |
| 미허용 통신 | Client에서 동일한 `curl -I` | `curl: (28) Connection timed out after 3002 milliseconds` | 허용되지 않은 흐름의 연결 실패 |
| 차단 지점 | Gateway 커널 로그에서 같은 시각의 TCP/8080 확인 | `[UFW BLOCK] IN=enp0s1 OUT=enp0s2 SRC=<CLIENT_IP> DST=<SERVER_LAN_IP> PROTO=TCP DPT=8080 SYN` | Client의 최초 연결 패킷이 Gateway 전달 단계에서 차단됨 |

Gateway→Server `200 OK`와 Client→Server SSH 성공으로 서비스 부재·경로 부재를 배제했다. 같은 시각 Gateway에서 `IN=enp0s1 OUT=enp0s2 ... DPT=8080 SYN`인 `[UFW BLOCK]` 기록 3건을 확인했으므로, 이 연결 시도의 차단 지점은 Gateway UFW 전달 정책이다. 공개용 로그 발췌에서는 원본의 MAC 주소와 임시 포트를 제외했다.

## 남은 검증

- 재부팅 후 정적 경로, forwarding, UFW 규칙과 관리 SSH 유지 확인

테스트가 끝난 뒤 Server에서 임시 Python HTTP 서비스를 `Ctrl+C`로 종료했다. `ss -lnt '( sport = :8080 )'`에는 LISTEN 항목이 없었다.

이 문서는 사용자가 공유한 실제 명령 출력에 근거해 작성했다. 원본 터미널 출력 전체를 공개 저장소에 저장한 것은 아니며, 공개용 기록에서 MAC 주소·임시 포트·계정 정보는 제외했다.
