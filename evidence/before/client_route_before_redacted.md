# 변경 전 상태 — 실습 터미널 출력 발췌

2026-10-16 실습 중 확인한 명령과 출력을 옮겼다. VM에서 자동 수집한 원본 로그가 아니라 당시 기록한 출력의 발췌다.

## Gateway: 라우팅과 방화벽 적용 전

```text
$ sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 0

$ sudo ufw status verbose
Status: inactive
```

## Client: Server 대역 경로 추가 전

```text
$ ip route
10.10.10.0/24 dev enp0s1 proto kernel scope link src 10.10.10.10

$ ip route get 10.20.20.10
RTNETLINK answers: Network is unreachable
```

이 시점에는 Client의 연결망 경로만 있었고 `10.20.20.0/24`로 향하는 경로가 없었다. 이후 정적 경로를 추가하고 SSH 접속으로 재검증했다.
