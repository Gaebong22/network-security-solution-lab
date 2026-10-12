# 진행 계획

## 1단계: 환경 준비와 네트워크 기초

- [ ] UTM VM 3대 생성
- [ ] 각 VM의 hostname과 역할 구분
- [ ] `lab-wan`, `lab-lan` 가상 network 구성
- [ ] interface 이름과 MAC address 확인
- [ ] IP address와 route 설정
- [ ] gateway 적용 전 같은 구간 통신 확인

학습 항목:

- IP address와 subnet mask
- network와 host 부분
- default gateway
- routing table
- ARP와 ICMP

## 2단계: 변경 전 상태 수집

- [ ] `ip address`, `ip route`, `ip link` 결과
- [ ] listening port와 running service
- [ ] firewall 초기 상태
- [ ] IP forwarding 상태
- [ ] 정상 TCP handshake packet capture
- [ ] evidence 민감정보 검토

## 3단계: Security Gateway 구축

- [ ] 두 interface 역할 확정
- [ ] IPv4 forwarding 설정
- [ ] 기본 firewall policy 설계
- [ ] HTTP와 관리용 SSH만 허용
- [ ] 미허용 TCP port 차단
- [ ] 설정 백업과 rollback 절차 작성

## 4단계: Packet 분석

- [ ] ARP request·reply
- [ ] ICMP echo request·reply
- [ ] TCP SYN·SYN/ACK·ACK
- [ ] HTTP request·response
- [ ] firewall 차단 시 재전송 또는 timeout
- [ ] 열린 port에 service가 없을 때 RST 비교

## 5단계: Suricata IDS

- [ ] package와 service 상태 확인
- [ ] 감시 interface 설정
- [ ] HOME_NET 범위 설정
- [ ] 기본 rule update
- [ ] 설정 문법 검사
- [ ] 통제된 탐지 테스트
- [ ] `eve.json`과 `fast.log` field 분석

## 6단계: 장애 재현과 기술지원

- [ ] 잘못된 default gateway
- [ ] firewall rule 누락 또는 잘못된 순서
- [ ] nginx stopped 상태
- [ ] Suricata interface 불일치
- [ ] 장애별 증상·가설·확인·원인·조치·재검증 기록

## 7단계: 최종 정리

- [ ] before/after 비교
- [ ] 방화벽 정책표 확정
- [ ] 테스트 결과표 작성
- [ ] 장애 보고서 작성
- [ ] 설치·운영·복구 가이드 작성
- [ ] README 완성
- [ ] 전체 저장소 민감정보 검사
- [ ] 면접 질문과 답변 정리

## 작업 방식

각 변경은 다음 단위로 진행한다.

1. 명령의 목적과 옵션 확인
2. 현재 상태 확인
3. 설정 백업 또는 snapshot
4. 한 가지 변경 적용
5. 문법과 service 상태 확인
6. 정상·비정상 테스트
7. 실제 결과 저장
8. 다음 변경 진행

여러 설정을 한 번에 적용하지 않는다. 오류가 발생하면 추가 변경보다 현재 상태와 로그를 먼저 확인한다.
