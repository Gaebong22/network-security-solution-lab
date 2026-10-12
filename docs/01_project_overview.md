# 프로젝트 개요

## 1. 배경

보안솔루션 구축·기술지원과 네트워크 보안 엔지니어는 제품 설치만 수행하지 않는다. 고객 환경의 네트워크 구조를 이해하고, 정책을 적용하며, 패킷과 로그를 이용해 장애와 보안 이벤트를 구분해야 한다.

이 프로젝트는 소규모 격리 네트워크를 직접 구성하고 다음 과정을 반복 검증하기 위해 진행한다.

```text
요구사항 분석 → 설계 → 구축 → 정책 적용 → 테스트 → 장애 분석 → 복구 → 문서화
```

## 2. 프로젝트에서 증명할 역량

- IP address, subnet, route와 gateway의 관계 설명
- 정상 TCP 연결 과정을 packet 단위로 확인
- firewall rule의 source, destination, protocol과 port 해석
- IDS signature와 alert field 분석
- packet, firewall log, IDS alert와 server log의 상관관계 확인
- 장애 원인을 physical/interface, route, firewall, service 순서로 좁히는 방법
- 작업 전 백업, 변경, 검증과 rollback 절차 작성
- 비전문가에게도 이해 가능한 기술지원 문서 작성

## 3. 범위

### 포함

- UTM 내부에 구성한 테스트 전용 network
- Ubuntu Server 기반 gateway와 protected server
- Linux routing 및 firewall
- Suricata IDS
- nginx와 SSH를 이용한 연결 테스트
- `ip`, `ss`, `ping`, `curl`, `tcpdump`, Wireshark 등 진단 도구
- 통제된 port 접근 및 탐지 테스트

### 제외

- 인터넷상의 시스템 또는 타인 소유 시스템 스캔
- 실제 악성코드 실행
- 서비스 가용성에 영향을 주는 대량 트래픽
- 탐지 우회를 목적으로 한 공격 기법 개발
- 검증하지 않은 결과의 PASS 처리

## 4. 완료 조건

- 구성도와 실제 interface·route가 일치한다.
- gateway를 통과하는 정상 HTTP 및 SSH 흐름을 설명할 수 있다.
- 최소 두 개 이상의 미허용 통신이 firewall에서 차단된다.
- 최소 두 개 이상의 통제된 테스트가 Suricata에서 탐지된다.
- 정상, 차단, 탐지 사례의 packet과 log가 각각 연결된다.
- 최소 세 건의 장애를 재현하고 원인·조치·재검증을 기록한다.
- 모든 공개 evidence에서 식별정보와 불필요한 payload를 검토한다.
- README, 구축 문서, 테스트 문서와 최종 보고서를 완성한다.

## 5. 취업 직무 연결

이 프로젝트는 다음 직무를 우선 대상으로 한다.

- 네트워크 보안 엔지니어
- 보안솔루션 구축·기술지원 엔지니어
- Linux·서버 보안 엔지니어
- 인프라 보안 운영 엔지니어
- 주니어 취약점 관리 및 보안 운영 직무
