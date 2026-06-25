# 2026-06-25 작업 정리

## 1. 오늘의 목표

오늘의 목표는 `security-auth-log-*`에 저장된 인증 로그를 Kibana에서 직접 필터링하고, 탐지 시나리오를 Elasticsearch query와 Kibana Dashboard로 연결하는 것이었습니다.

이전 단계까지는 다음을 완료했습니다.

```text
auth.log
-> Kafka security-auth-log topic
-> Logstash parsing
-> Elasticsearch security-auth-log-* index
```

오늘은 이 데이터를 이용해 다음을 진행했습니다.

```text
Kibana Discover 필터링
Elasticsearch 탐지 query 작성
Brute Force 테스트 로그 생성
탐지 query 검증
Kibana Dashboard 패널 3개 생성
Kafka 운영 개념 학습 문서화
```

## 2. 오늘 만든 파일

### 2.1 Kibana 필터링 및 탐지 쿼리 문서

```text
docs/kibana-filtering-and-detection-queries.md
```

역할:

```text
Kibana Discover에서 사용할 KQL 필터와 Elasticsearch Query DSL 파일 실행 방법을 정리했습니다.
```

포함 내용:

1. `event_type`별 필터링 방법
2. `ssh_failed_login`, `root_login`, `sudo_command` 조회 방법
3. TOP 실패 IP 집계 방법
4. 5분 내 SSH 실패 20회 이상 탐지 방법
5. 테스트 로그를 Kafka에 넣는 명령

### 2.2 Query 파일 확인 가이드

```text
docs/query-files-guide.md
```

역할:

```text
이번 단계에서 만든 Elasticsearch query 파일을 어디서 보고, 어떻게 실행하는지 정리했습니다.
```

핵심 요약:

```text
파일 보기 = Get-Content
쿼리 실행 = curl.exe --data-binary "@파일경로"
```

PowerShell과 WSL/Linux의 차이도 학습했습니다.

```text
PowerShell: Get-Content
WSL/Linux: cat
```

### 2.3 Kafka 운영 개념 문서

```text
docs/kafka-operations-concepts.md
```

역할:

```text
Kafka가 실무에서 어려워지는 지점을 초보자용으로 정리했습니다.
```

정리한 개념:

```text
leader partition
follower replica
ISR
acks
unclean leader election
consumer lag
broker disk usage
under replicated partitions
request latency
message in/out rate
controller 상태
consumer 장애
배포
스케일 아웃
네트워크 지연
```

면접 대비 핵심 문장:

```text
Kafka가 어려운 이유는 단순 메시지 큐가 아니라 분산 시스템이기 때문입니다.
topic, partition, offset, consumer group뿐 아니라 replication, ISR, lag, rebalance, 장애 복구까지 이해해야 실무 운영이 가능합니다.
```

### 2.4 Kibana Dashboard 구성 가이드

```text
docs/kibana-dashboard-guide.md
```

역할:

```text
Kibana에서 보안 인증 로그 Dashboard를 만드는 순서를 정리했습니다.
```

포함한 Dashboard 항목:

1. 시간대별 SSH 로그인 실패 추이
2. TOP 공격 IP
3. 계정별 로그인 실패 횟수
4. root 로그인 발생 목록
5. sudo 명령어 사용 목록
6. 위험 이벤트 목록

### 2.5 Dashboard 패널별 문서

```text
docs/dashboards/01-ssh-failed-login-trend.md
docs/dashboards/02-ssh-failed-login-top-ip.md
docs/dashboards/03-ssh-failed-login-by-username.md
```

역할:

```text
오늘 실제로 만든 Dashboard 패널 3개를 각각 문서화했습니다.
```

각 문서에는 다음을 정리했습니다.

```text
패널 목적
Data View
KQL 필터
차트 타입
축/집계 설정
확인 결과
보안적으로 알 수 있는 점
관련 탐지 시나리오
```

### 2.6 Elasticsearch Query DSL 파일

```text
elasticsearch/queries/ssh-failed-login.json
elasticsearch/queries/root-login.json
elasticsearch/queries/sudo-command.json
elasticsearch/queries/failed-login-by-ip.json
elasticsearch/queries/brute-force-5m.json
```

역할:

```text
Kibana에서 손으로 검색하는 조건을 Elasticsearch API query 파일로 저장했습니다.
```

각 파일의 목적:

| 파일 | 목적 |
|---|---|
| `ssh-failed-login.json` | SSH 로그인 실패 조회 |
| `root-login.json` | root 로그인 성공 이벤트 조회 |
| `sudo-command.json` | sudo 명령어 사용 이벤트 조회 |
| `failed-login-by-ip.json` | SSH 실패를 IP별로 집계 |
| `brute-force-5m.json` | 5분 내 SSH 실패 20회 이상 탐지 |

### 2.7 Brute Force 테스트 로그

```text
sample-logs/brute-force-auth.log
```

역할:

```text
같은 IP에서 5분 안에 SSH 로그인 실패가 20회 발생하는 테스트 로그입니다.
```

공격 IP:

```text
203.0.113.200
```

공격 계정:

```text
attacker
```

## 3. 오늘 수행한 주요 실습

### 3.1 Kibana에서 event_type 필터링

Kibana Discover 또는 Lens에서 다음 KQL을 사용했습니다.

```text
event_type: ssh_failed_login
event_type: root_login
event_type: sudo_command
status: failed
username: root
source_ip: "203.0.113.200"
```

의미:

```text
Logstash가 만든 구조화 필드를 이용해 보안 이벤트를 조건별로 조회했습니다.
```

중요한 학습 포인트:

```text
Kibana는 Elasticsearch에 저장된 필드를 기준으로 검색한다.
event_type, source_ip, username, status 같은 필드가 있어야 탐지와 대시보드가 가능하다.
```

### 3.2 Elasticsearch query 파일 실행

예시:

```powershell
curl.exe -s -X GET "http://localhost:9200/security-auth-log-*/_search?pretty" -H "Content-Type: application/json" --data-binary "@elasticsearch/queries/brute-force-5m.json"
```

명령어 역할:

```text
brute-force-5m.json 파일에 정의된 탐지 조건을 Elasticsearch에 전달해 검색합니다.
```

영향:

```text
데이터를 수정하지 않습니다.
Elasticsearch에 저장된 로그를 조회만 합니다.
```

### 3.3 Brute Force 테스트 로그 Kafka 전송

PowerShell:

```powershell
Get-Content sample-logs\brute-force-auth.log | docker exec -i etl-kafka /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic security-auth-log
```

WSL/Linux:

```bash
cat sample-logs/brute-force-auth.log | docker exec -i etl-kafka /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic security-auth-log
```

명령어 역할:

```text
brute-force-auth.log 파일의 각 줄을 Kafka security-auth-log topic에 메시지로 넣습니다.
```

데이터 흐름:

```text
sample-logs/brute-force-auth.log
-> Kafka security-auth-log topic
-> Logstash parsing
-> Elasticsearch security-auth-log-* index
-> Kibana Dashboard
```

주의:

```text
이 명령을 여러 번 실행하면 같은 테스트 로그가 반복 저장됩니다.
오늘은 중복 실행으로 203.0.113.200 IP의 실패 횟수가 더 크게 표시되었습니다.
```

## 4. 오늘 검증한 탐지 시나리오

### 4.1 SSH 로그인 실패 조회

조건:

```text
event_type = ssh_failed_login
```

의미:

```text
SSH 로그인에 실패한 이벤트를 모두 조회합니다.
```

### 4.2 root 로그인 탐지

조건:

```text
event_type = root_login
```

의미:

```text
root 계정으로 로그인에 성공한 이벤트를 탐지합니다.
```

보안 의미:

```text
root 계정은 권한이 매우 높으므로 직접 로그인 성공은 확인이 필요한 이벤트입니다.
```

### 4.3 sudo 명령어 사용 탐지

조건:

```text
event_type = sudo_command
```

의미:

```text
sudo 명령어 사용 이벤트를 탐지합니다.
```

보안 의미:

```text
sudo는 권한 상승 행위이므로 누가 어떤 명령어를 실행했는지 추적해야 합니다.
```

### 4.4 TOP 실패 IP 집계

조건:

```text
event_type = ssh_failed_login
source_ip별 count 집계
```

검증 결과:

```text
203.0.113.200 IP가 가장 많은 실패 로그를 발생시켰습니다.
```

### 4.5 Brute Force 의심 탐지

조건:

```text
5분 내 같은 source_ip에서 SSH 로그인 실패 20회 이상
```

검증 결과:

```json
{
  "key_as_string": "2026-06-25T12:10:00.000Z",
  "doc_count": 20,
  "by_source_ip": {
    "buckets": [
      {
        "key": "203.0.113.200",
        "doc_count": 20
      }
    ]
  }
}
```

의미:

```text
203.0.113.200 IP가 5분 안에 SSH 로그인 실패를 20회 발생시켜 Brute Force 의심 조건에 걸렸습니다.
```

## 5. 오늘 만든 Kibana Dashboard 패널

### 5.1 시간대별 SSH 로그인 실패 추이

문서:

```text
docs/dashboards/01-ssh-failed-login-trend.md
```

Kibana 설정:

```text
KQL: event_type: ssh_failed_login
X축: @timestamp
Y축: Records
Chart: Bar vertical 또는 Line chart
```

이 패널로 알 수 있는 것:

```text
SSH 로그인 실패가 언제 많이 발생했는지 확인할 수 있습니다.
특정 시간대에 실패가 몰리면 Brute Force 가능성을 의심할 수 있습니다.
```

### 5.2 SSH 로그인 실패 TOP IP

문서:

```text
docs/dashboards/02-ssh-failed-login-top-ip.md
```

Kibana 설정:

```text
KQL: event_type: ssh_failed_login
Metric: Records
Group by: source_ip.keyword
Chart: Horizontal bar
Sort: Count descending
```

이 패널로 알 수 있는 것:

```text
어떤 IP가 SSH 로그인 실패를 가장 많이 발생시켰는지 확인할 수 있습니다.
```

오늘 확인한 주요 IP:

```text
203.0.113.200
```

### 5.3 계정별 SSH 로그인 실패 횟수

문서:

```text
docs/dashboards/03-ssh-failed-login-by-username.md
```

Kibana 설정:

```text
KQL: event_type: ssh_failed_login
Metric: Records
Group by: username 또는 username.keyword
Chart: Horizontal bar
Sort: Count descending
```

이 패널로 알 수 있는 것:

```text
어떤 계정명이 공격 대상이 되었는지 확인할 수 있습니다.
```

예상 상위 계정:

```text
attacker
root
admin
test
```

## 6. 오늘 겪은 문제와 해결

### 6.1 WSL에서 Get-Content 오류

문제:

```text
Get-Content: command not found
```

원인:

```text
Get-Content는 PowerShell 명령어입니다.
WSL/Linux에서는 사용할 수 없습니다.
```

해결:

```bash
cat sample-logs/brute-force-auth.log | docker exec -i etl-kafka /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic security-auth-log
```

정리:

```text
PowerShell = Get-Content
WSL/Linux = cat
```

### 6.2 같은 테스트 로그를 여러 번 넣음

상황:

```text
brute-force-auth.log를 여러 번 Kafka에 전송했습니다.
```

영향:

```text
Elasticsearch에 같은 유형의 실패 로그가 중복 저장되었습니다.
TOP IP와 Brute Force 탐지 결과에서 203.0.113.200의 count가 더 크게 표시되었습니다.
```

학습 포인트:

```text
Kafka producer 명령은 실행할 때마다 메시지를 새로 넣습니다.
Elasticsearch도 새 문서로 저장합니다.
로그 파이프라인에서는 중복 이벤트 처리 전략이 중요합니다.
```

### 6.3 Kibana에서 Count of records가 안 보임

상황:

```text
Lens Vertical axis에서 Count of records를 찾을 수 없었습니다.
```

해결:

```text
Kibana Lens에서 Records가 Count of records 역할을 합니다.
```

정리:

```text
Records = 로그 문서 개수
```

### 6.4 source_ip 대신 source_ip.keyword만 보임

상황:

```text
Kibana Lens에서 source_ip는 안 보이고 source_ip.keyword만 보였습니다.
```

원인:

```text
Data View가 예전 mapping과 새 mapping이 섞인 index를 보고 있어 Kibana가 집계 가능한 keyword 하위 필드를 보여준 것입니다.
```

해결:

```text
TOP 공격 IP 패널에서는 source_ip.keyword를 사용했습니다.
```

정리:

```text
source_ip.keyword는 IP 값을 문자열 그대로 묶어 집계하는 데 사용할 수 있습니다.
```

## 7. 오늘 배운 핵심 개념

### 7.1 Kibana KQL

Kibana에서 필터링할 때 사용하는 검색 문법입니다.

예:

```text
event_type: ssh_failed_login
source_ip: "203.0.113.200"
username: root
```

### 7.2 Elasticsearch Query DSL

Elasticsearch API로 검색 조건을 표현하는 JSON 문법입니다.

Kibana에서 손으로 검색하는 조건을 파일로 관리할 수 있습니다.

### 7.3 Aggregation

집계입니다.

예:

```text
source_ip별 SSH 실패 횟수
username별 SSH 실패 횟수
5분 단위 SSH 실패 횟수
```

### 7.4 Dashboard

여러 시각화를 모아 한 화면에서 관찰하는 공간입니다.

이번 프로젝트에서는 보안 관제 화면처럼 사용합니다.

### 7.5 Kafka의 실무 난이도

Kafka는 단순 메시지 큐가 아니라 분산 시스템입니다.

오늘 정리한 주요 운영 개념:

```text
leader partition
follower replica
ISR
acks
consumer lag
under replicated partitions
controller 상태
rebalance
```

## 8. 면접 대비 답변 예시

### 질문 1. Kafka는 이 프로젝트에서 어떤 역할인가요?

답변:

```text
Kafka는 Filebeat나 테스트 producer가 보낸 로그를 topic별로 받아두는 중간 버퍼 역할을 합니다.
Logstash는 Kafka topic에서 로그를 consume해 파싱하고 Elasticsearch에 저장합니다.
Kafka를 중간에 둔 이유는 로그 생산자와 소비자를 분리하고, Logstash나 Elasticsearch가 잠시 느려져도 로그를 안정적으로 쌓아둘 수 있기 때문입니다.
```

### 질문 2. Logstash와 Elasticsearch의 역할 차이는 무엇인가요?

답변:

```text
Logstash는 Kafka에서 읽은 원본 로그를 파싱하고 event_type, source_ip, username 같은 구조화 필드를 생성합니다.
Elasticsearch는 Logstash가 정제한 문서를 저장하고 검색/집계를 제공합니다.
즉 정제는 Logstash가 하고, 저장과 검색은 Elasticsearch가 담당합니다.
```

### 질문 3. Brute Force 탐지는 어떻게 구현했나요?

답변:

```text
Logstash에서 SSH 로그인 실패 로그를 ssh_failed_login 이벤트로 파싱했습니다.
이후 Elasticsearch Query DSL에서 5분 단위 date_histogram을 사용하고, source_ip별 terms aggregation을 적용했습니다.
같은 IP에서 5분 안에 실패 로그가 20회 이상 발생하면 Brute Force 의심 이벤트로 볼 수 있도록 쿼리를 작성했습니다.
```

### 질문 4. Kibana Dashboard에는 무엇을 만들었나요?

답변:

```text
현재는 인증 로그 기준으로 시간대별 SSH 로그인 실패 추이, SSH 로그인 실패 TOP IP, 계정별 로그인 실패 횟수 패널을 만들었습니다.
이를 통해 실패 이벤트가 언제 몰렸는지, 어떤 IP가 많이 실패했는지, 어떤 계정명이 공격 대상이 되었는지 확인할 수 있습니다.
```

### 질문 5. source_ip와 source_ip.keyword 차이는 무엇인가요?

답변:

```text
source_ip는 원본 IP 필드이고, source_ip.keyword는 문자열 값을 그대로 집계하기 위한 하위 필드입니다.
이번 프로젝트에서는 예전 index와 새 index의 mapping이 섞여 Kibana Lens에서 source_ip.keyword가 집계용으로 보였고, TOP IP 패널에서는 이를 사용해 IP별 실패 횟수를 집계했습니다.
```

## 9. 현재 진행 상태

전체 프로젝트 기준 현재 진행률:

```text
약 60% 이상
```

완료:

```text
Docker Compose 실행 환경
Kafka topic
Logstash auth.log 파싱
Elasticsearch mapping
Kibana Discover 필터링
Elasticsearch 탐지 쿼리
Brute Force 탐지 검증
Kibana Dashboard 패널 3개
학습 문서화
```

남은 작업:

```text
root 로그인 목록 패널
sudo 명령어 사용 목록 패널
위험 이벤트 목록 패널
Dashboard 캡처 저장
탐지 시나리오 문서 완성
Filebeat 연결
nginx-access-log 파싱
docker-container-log 파싱
README.md 작성
최종 포트폴리오 정리
```

## 10. 다음 단계

다음 작업은 Dashboard를 계속 완성하는 것입니다.

다음에 만들 패널:

```text
root 로그인 발생 목록
sudo 명령어 사용 목록
위험 이벤트 목록
```

그 다음 작업:

```text
Dashboard 캡처 저장
탐지 시나리오 문서 정리
GitHub README 작성
```
