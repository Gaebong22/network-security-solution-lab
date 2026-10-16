# 적용 후 상태 — 실습 터미널 출력 발췌

출처: 2026-10-16 실습에서 확인한 VM 터미널 출력. 명령·결과를 당시 기록에서 옮긴 것으로, 자동 수집한 원본 로그 전체는 아니다. 공개에 불필요한 MAC 주소, 임시 송신 포트, 관리 단말 주소는 `<REDACTED>` 또는 `<MANAGEMENT_HOST_IP>`로 바꿨고 긴 출력의 일부는 생략했다. 실습망 IP와 테스트 포트는 검증을 위해 남겼다.

## 1. 양방향 경로와 허용된 SSH

Client에서 정적 경로를 추가한 후:

```text
$ ip route get 10.20.20.10
10.20.20.10 via 10.10.10.1 dev enp0s1 src 10.10.10.10 uid 1000
    cache
```

Server에서 역방향 경로 확인:

```text
$ ip route get 10.10.10.10
10.10.10.10 via 10.20.20.1 dev enp0s1 src 10.20.20.10 uid 1000
    cache
```

Client에서 Server에 SSH 로그인했다. Server에서 확인한 세션 주소는 다음과 같다. 임시 송신 포트는 공개하지 않았다.

```text
$ echo "$SSH_CONNECTION"
10.10.10.10 <REDACTED> 10.20.20.10 22
```

## 2. TCP/8080 차단 검증

Server에서 임시 HTTP 서비스를 실행하는 동안 Gateway에서는 응답이 확인됐다.

```text
Server $ python3 -m http.server 8080 --bind 10.20.20.10 --directory /tmp/netlab-http-test
Serving HTTP on 10.20.20.10 port 8080 (http://10.20.20.10:8080/) ...
10.20.20.1 - - [16/Oct/2026 08:11:06] "HEAD / HTTP/1.1" 200 -
```

```text
Gateway $ curl -I --connect-timeout 3 --max-time 6 http://10.20.20.10:8080/
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.14.4
Date: Fri, 16 Oct 2026 08:11:06 GMT
Content-type: text/html; charset=utf-8
Content-Length: 256
```

같은 서비스에 Client에서 접근하면 시간 초과가 났다.

```text
Client $ curl -I --connect-timeout 3 --max-time 6 http://10.20.20.10:8080/
curl: (28) Connection timed out after 3002 milliseconds
```

Gateway의 같은 시각 커널 로그에는 Client→Server TCP/8080 SYN 차단이 3회 남았다. 아래 로그에서 `MAC`과 `SPT`만 가렸다.

```text
Oct 16 08:11:18 netlab-gateway kernel: [UFW BLOCK] IN=enp0s1 OUT=enp0s2 MAC=<REDACTED> SRC=10.10.10.10 DST=10.20.20.10 LEN=60 TOS=0x00 PREC=0x00 TTL=63 ID=43279 DF PROTO=TCP SPT=<REDACTED> DPT=8080 WINDOW=64240 RES=0x00 SYN URGP=0
Oct 16 08:11:19 netlab-gateway kernel: [UFW BLOCK] IN=enp0s1 OUT=enp0s2 MAC=<REDACTED> SRC=10.10.10.10 DST=10.20.20.10 LEN=60 TOS=0x00 PREC=0x00 TTL=63 ID=43280 DF PROTO=TCP SPT=<REDACTED> DPT=8080 WINDOW=64240 RES=0x00 SYN URGP=0
Oct 16 08:11:20 netlab-gateway kernel: [UFW BLOCK] IN=enp0s1 OUT=enp0s2 MAC=<REDACTED> SRC=10.10.10.10 DST=10.20.20.10 LEN=60 TOS=0x00 PREC=0x00 TTL=63 ID=43281 DF PROTO=TCP SPT=<REDACTED> DPT=8080 WINDOW=64240 RES=0x00 SYN URGP=0
```

검증 뒤 Server에서 `Ctrl+C`로 임시 HTTP 서비스를 종료했다. 다음 출력에는 LISTEN 항목이 없다.

```text
Server $ ss -lnt '( sport = :8080 )'
State       Recv-Q       Send-Q             Local Address:Port             Peer Address:Port
```

## 3. 재부팅 후 설정 유지

Gateway를 재부팅한 뒤 확인한 출력. 관리 주소와 IPv6 주소는 가렸다.

```text
Gateway $ ip -br address
lo               UNKNOWN        127.0.0.1/8 ::1/128
enp0s1           UP             10.10.10.1/24 [IPv6 생략]
enp0s2           UP             10.20.20.1/24 [IPv6 생략]
enp0s3           UP             <MANAGEMENT_IP>/24 [IPv6 생략]

Gateway $ ip route
default via <MANAGEMENT_GATEWAY_IP> dev enp0s3 proto dhcp src <MANAGEMENT_IP> metric 1024
10.10.10.0/24 dev enp0s1 proto kernel scope link src 10.10.10.1
10.20.20.0/24 dev enp0s2 proto kernel scope link src 10.20.20.1
[관리망의 연결 경로 생략]

Gateway $ sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 1

Gateway $ sudo ufw status verbose
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip

To                             Action      From
--                             ------      ----
22/tcp on enp0s3               ALLOW IN    <MANAGEMENT_HOST_IP>  # Mac management SSH
10.20.20.10 22/tcp on enp0s2   ALLOW FWD   10.10.10.10 on enp0s1 # Client to Server SSH
```

Gateway 재부팅 후 Mac→Gateway 및 Client→Server SSH 로그인에 성공했다. Client와 Server를 각각 재부팅한 후에는 아래 경로가 유지됐고, Mac에서 Gateway를 경유해 각 VM에 다시 접속했다.

```text
Client $ ip route get 10.20.20.10
10.20.20.10 via 10.10.10.1 dev enp0s1 src 10.10.10.10 uid 1000
    cache

Server $ ip -br address
lo               UNKNOWN        127.0.0.1/8 ::1/128
enp0s1           UP             10.20.20.10/24 [IPv6 생략]

Server $ ip route get 10.10.10.10
10.10.10.10 via 10.20.20.1 dev enp0s1 src 10.20.20.10 uid 1000
    cache
```

TCP/8080 차단 요청은 재부팅 후 다시 실행하지 않았다. 위 지속성 결과는 주소·경로·방화벽 상태 및 명시한 SSH 접속에 대한 것이다.
