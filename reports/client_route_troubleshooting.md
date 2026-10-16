# Client→Server 경로 장애 분석

발생·재검증일: 2026-10-16
대상: UTM 격리망 Client(`lab-wan`)에서 Server(`lab-lan`)로 가는 통신

## 증상

Client에 `10.10.10.10/24`를 설정하고 Gateway WAN 주소로 ping 3/3 성공했지만, Server 주소로 향하는 경로 조회는 다음과 같이 실패했다.

```text
$ ip route
10.10.10.0/24 dev enp0s1 proto kernel scope link src 10.10.10.10
$ ip route get 10.20.20.10
RTNETLINK answers: Network is unreachable
```

## 확인과 원인

`ip route`에는 Client가 직접 연결된 `10.10.10.0/24`만 있었다. 목적지 `10.20.20.10`은 다른 대역이므로 Gateway로 보내는 경로가 필요했다. `ip route get`은 패킷을 송신하지 않고 커널의 경로 선택을 확인하므로, 이 단계의 실패는 방화벽 차단이 아니라 **Client의 경로 부재**를 가리킨다.

## 조치

기존 Netplan 설정을 백업하고 Client의 `enp0s1` 항목에 다음 정적 경로를 추가했다.

```yaml
routes:
  - to: 10.20.20.0/24
    via: 10.10.10.1
```

`sudo netplan generate`로 문법을 확인한 뒤 설정을 적용했다. Server에는 반대 방향인 `10.10.10.0/24 via 10.20.20.1` 경로가 이미 설정돼 있다.

## 재검증

```text
$ ip route get 10.20.20.10
10.20.20.10 via 10.10.10.1 dev enp0s1 src 10.10.10.10 uid 1000
```

이후 Client에서 Server로 SSH 접속·로그인에 성공했고 Server의 SSH 세션 출발지에서도 Client 주소 `10.10.10.10`을 확인했다. 따라서 경로 복구와 허용된 TCP/22 전달 통신까지 확인했다. 이 SSH 성공만으로 다른 포트의 허용 여부나 재부팅 후 지속성을 증명하지는 않는다. [변경 전 출력 발췌](../evidence/before/client_route_before_redacted.md)도 별도로 보존했다.

## 원인과 조치 요약

같은 대역의 Gateway에는 ping이 됐지만 다른 대역의 Server에는 `Network is unreachable`이 나왔다. 서비스나 방화벽을 먼저 바꾸지 않고 Client 라우팅 테이블을 확인해 목적지 대역 경로가 없음을 찾았다. 정적 경로를 추가한 뒤 `ip route get`과 실제 SSH 로그인으로 재검증했다.
