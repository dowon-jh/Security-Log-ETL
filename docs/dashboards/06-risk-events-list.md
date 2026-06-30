# Dashboard 06. 위험 이벤트 목록

## 1. 패널 목적

이 패널은 프로젝트에서 탐지한 주요 보안 이벤트를 한곳에 모아서 확인하기 위한 목록 패널입니다.

개별 패널이 특정 이벤트만 보는 용도라면, 위험 이벤트 목록은 보안 관제 화면처럼 여러 이벤트를 함께 확인하는 용도입니다.

## 2. 사용하는 데이터

Data View:

```text
security_logs2
```

대상 index:

```text
security-auth-log-*
```

포함하는 이벤트:

```text
ssh_failed_login
root_login
sudo_command
```

## 3. Kibana 설정

이 패널은 Discover에서 저장한 Table로 구성합니다.

KQL 필터:

```text
event_type: ssh_failed_login or event_type: root_login or event_type: sudo_command
```

시간 범위:

```text
Last 30 days
```

추가한 컬럼:

```text
@timestamp
raw_message
status
event_type
source_ip
username
service
```

저장 이름:

```text
위험 이벤트 목록
```

## 4. 이 KQL의 의미

KQL 조건은 다음 의미입니다.

```text
SSH 로그인 실패 이벤트이거나
root 로그인 이벤트이거나
sudo 명령어 사용 이벤트인 로그만 보여준다.
```

즉, 지금 프로젝트에서 보안적으로 의미 있다고 판단한 이벤트들을 하나의 목록으로 모아 보는 조건입니다.

## 5. 이 패널로 알 수 있는 것

이 패널은 다음 질문에 답합니다.

```text
현재 어떤 위험 이벤트가 발생했는가?
이벤트는 언제 발생했는가?
어떤 IP와 계정이 관련되어 있는가?
실패 이벤트인가 성공 이벤트인가?
원본 로그 메시지는 무엇인가?
```

## 6. 관련 탐지 시나리오

이 패널은 여러 탐지 시나리오를 함께 보여줍니다.

```text
5분 내 SSH 로그인 실패 20회 이상 -> Brute Force 의심
root 계정 로그인 발생 -> 관리자 계정 접근 탐지
sudo 명령어 사용 이벤트 탐지
```

## 7. source_ip와 source_ip.keyword

Discover 목록에서는 `source_ip`를 사용해도 됩니다.

```text
source_ip = 실제 IP 값을 화면에 표시하는 필드
source_ip.keyword = IP 값을 집계하거나 그룹화할 때 사용하는 하위 필드
```

따라서 목록 패널에서는 `source_ip`를 사용했고, TOP IP 차트처럼 집계가 필요한 패널에서는 `source_ip.keyword`를 사용했습니다.

## 8. 실무 관점

실무에서는 위험 이벤트 목록이 보안 관제의 출발점이 됩니다.

분석가는 이 목록에서 이벤트를 먼저 확인하고, 특정 IP나 계정이 수상하면 상세 로그, 원본 메시지, 시간대별 발생 추이를 추가로 분석합니다.

이번 프로젝트에서는 이 흐름을 로컬 단일 노드 환경에서 학습용으로 구현했습니다.

