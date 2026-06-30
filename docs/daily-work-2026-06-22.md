# 2026-06-22 작업 정리

## 1. 오늘의 목표

오늘의 목표는 기본 ETL 연결 검증 이후 다음 단계로 넘어가, Linux 인증 로그를 분석 가능한 구조화 데이터로 만드는 것이었습니다.

진행한 핵심 작업:

1. 다음 작업 전 환경 준비 체크리스트 작성
2. `auth.log` Logstash 파싱 구현
3. `security-auth-log-*` Elasticsearch index 분리
4. Elasticsearch index mapping template 작성
5. mapping template 적용 및 검증
6. Kibana에서 확인할 Data View 안내

## 2. 오늘 만든 문서

### 2.1 docs/resume-environment-checklist.md

PC를 다시 켠 뒤 프로젝트 환경을 복구하기 위한 체크리스트입니다.

포함 내용:

1. Docker Desktop 실행
2. 프로젝트 폴더 이동
3. Docker Compose 실행
4. Kafka topic 확인
5. Elasticsearch health 확인
6. Kibana 접속
7. Logstash 상태 확인

핵심 명령:

```powershell
cd Security-Log-ETL
docker compose up -d
docker compose ps
curl.exe http://localhost:9200/_cluster/health
```

### 2.2 docs/auth-log-parsing.md

`security-auth-log` topic으로 들어온 Linux 인증 로그를 Logstash에서 파싱하는 과정을 정리한 문서입니다.

포함 내용:

1. 내가 하는 행동과 영향
2. Logstash 설정 변경 내용
3. Logstash 재시작의 의미
4. Kafka에 auth 로그를 수동 전송하는 이유
5. SSH 로그인 실패 파싱
6. SSH 로그인 성공 파싱
7. root 로그인 탐지
8. sudo 명령어 사용 파싱
9. Elasticsearch 확인 명령
10. Kibana 확인 방법

### 2.3 docs/elasticsearch-index-mapping.md

Elasticsearch index mapping이 왜 필요한지, 어떤 타입을 지정했는지 정리한 문서입니다.

포함 내용:

1. 자동 mapping의 한계
2. `source_ip`를 `ip` 타입으로 지정해야 하는 이유
3. `event_type`, `status`, `username`을 `keyword`로 지정하는 이유
4. `raw_message`를 `text`로 두는 이유
5. index template 적용 명령
6. mapping 검증 명령
7. 단일 노드에서 `number_of_replicas: 0`을 둔 이유

### 2.4 docs/daily-work-2026-06-22.md

오늘 작업 내용을 날짜별로 다시 볼 수 있도록 정리한 문서입니다.

## 3. 오늘 수정한 설정 파일

### 3.1 logstash/pipeline/logstash.conf

기존에는 Kafka에서 읽은 로그를 원본 형태로만 Elasticsearch에 저장했습니다.

기존 흐름:

```text
Kafka topic
-> Logstash
-> etl-raw-log-YYYY.MM.dd
```

오늘 변경 후 흐름:

```text
security-auth-log topic
-> Logstash auth parsing
-> security-auth-log-YYYY.MM.dd
```

추가된 주요 기능:

1. Kafka topic metadata 사용
2. `security-auth-log` topic 분기 처리
3. SSH 로그인 실패 로그 grok 파싱
4. SSH 로그인 성공 로그 grok 파싱
5. sudo 명령어 로그 grok 파싱
6. root 로그인 성공 시 `event_type`을 `root_login`으로 변경
7. auth 로그를 `security-auth-log-*` index로 분리 저장

생성되는 주요 필드:

```text
source_ip
source_port
username
event_type
status
service
auth_method
sudo_command
raw_message
pipeline_stage
```

## 4. 오늘 만든 Elasticsearch 설정

### 4.1 elasticsearch/mappings/security-auth-log-template.json

`security-auth-log-*` index에 적용할 Elasticsearch index template입니다.

주요 mapping:

| 필드 | 타입 | 이유 |
|---|---|---|
| `@timestamp` | `date` | 시간 기반 검색 |
| `source_ip` | `ip` | IP 검색과 필터링 |
| `source_port` | `integer` | 포트 번호 |
| `username` | `keyword` | 계정별 집계 |
| `event_type` | `keyword` | 이벤트 유형별 집계 |
| `status` | `keyword` | 성공/실패 집계 |
| `service` | `keyword` | 서비스별 필터 |
| `raw_message` | `text` | 원본 로그 검색 |

로컬 단일 노드 환경이라 replica는 0으로 설정했습니다.

```json
"number_of_replicas": 0
```

## 5. 오늘 실행한 주요 명령어와 의미

### 5.1 Logstash 재시작

```powershell
docker compose restart logstash
```

역할:

```text
수정한 logstash.conf를 Logstash 컨테이너가 다시 읽게 한다.
```

영향:

```text
Kafka, Elasticsearch, Kibana는 그대로 유지된다.
Logstash만 재시작된다.
기존 Elasticsearch 데이터는 삭제되지 않는다.
```

### 5.2 auth 샘플 로그 Kafka 전송

```powershell
docker exec etl-kafka /bin/bash -c "printf '%s\n' 'Jun 17 21:10:33 ubuntu sshd[1234]: Failed password for invalid user admin from 192.168.0.10 port 54321 ssh2' 'Jun 17 21:11:02 ubuntu sshd[1235]: Failed password for root from 203.0.113.10 port 49812 ssh2' 'Jun 17 21:12:15 ubuntu sshd[1236]: Accepted password for ubuntu from 192.168.0.20 port 50222 ssh2' 'Jun 17 21:13:44 ubuntu sudo:   ubuntu : TTY=pts/0 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/bin/apt update' 'Jun 17 21:14:09 ubuntu sshd[1237]: Accepted publickey for root from 198.51.100.7 port 60123 ssh2' | /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic security-auth-log"
```

역할:

```text
Filebeat가 아직 없는 현재 단계에서, 사람이 직접 auth 로그를 Kafka topic에 넣어 테스트한다.
```

영향:

```text
security-auth-log topic에 로그 5줄 추가
Logstash가 consume
auth 파싱 수행
Elasticsearch security-auth-log-* index에 저장
```

### 5.3 Elasticsearch index 확인

```powershell
curl.exe -s "http://localhost:9200/_cat/indices/security-auth-log-*?v"
```

역할:

```text
security-auth-log-* index가 생성됐는지 확인한다.
```

검증 결과:

```text
security-auth-log-2026.06.17
security-auth-log-2026.06.22
```

### 5.4 파싱 결과 확인

```powershell
curl.exe -s "http://localhost:9200/security-auth-log-*/_search?pretty&size=10"
```

역할:

```text
Elasticsearch에 저장된 auth 로그 문서를 조회한다.
```

확인한 필드:

```text
source_ip
username
event_type
status
service
auth_method
sudo_command
raw_message
pipeline_stage
```

### 5.5 Elasticsearch index template 적용

```powershell
curl.exe -s -X PUT "http://localhost:9200/_index_template/security-auth-log-template" -H "Content-Type: application/json" --data-binary "@elasticsearch/mappings/security-auth-log-template.json"
```

역할:

```text
앞으로 생성되는 security-auth-log-* index의 field type 규칙을 Elasticsearch에 등록한다.
```

영향:

```text
새 index부터 source_ip는 ip 타입으로 생성된다.
event_type, username, status는 keyword 타입으로 생성된다.
기존 index의 mapping은 변경되지 않는다.
```

### 5.6 mapping template 확인

```powershell
curl.exe -s "http://localhost:9200/_index_template/security-auth-log-template?pretty"
```

역할:

```text
Elasticsearch에 등록된 security-auth-log-template 내용을 확인한다.
```

### 5.7 새 날짜 로그로 mapping 검증

```powershell
docker exec etl-kafka /bin/bash -c "echo 'Jun 22 21:10:33 ubuntu sshd[2001]: Failed password for invalid user test from 203.0.113.55 port 55123 ssh2' | /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic security-auth-log"
```

역할:

```text
새 날짜의 security-auth-log index를 생성해 template이 적용되는지 확인한다.
```

검증 결과:

```text
security-auth-log-2026.06.22
```

이 index에서 확인된 mapping:

```json
"source_ip": {
  "type": "ip"
}
```

## 6. 오늘 검증한 결과

### 6.1 auth 로그 파싱 성공

SSH 실패 로그:

```json
{
  "source_ip": "192.168.0.10",
  "username": "admin",
  "event_type": "ssh_failed_login",
  "status": "failed",
  "service": "sshd",
  "pipeline_stage": "parsed_auth"
}
```

root 로그인 성공 로그:

```json
{
  "source_ip": "198.51.100.7",
  "username": "root",
  "event_type": "root_login",
  "status": "success",
  "service": "sshd",
  "auth_method": "publickey"
}
```

sudo 명령어 사용 로그:

```json
{
  "username": "ubuntu",
  "event_type": "sudo_command",
  "status": "success",
  "service": "sudo",
  "sudo_command": "/usr/bin/apt update"
}
```

### 6.2 mapping template 적용 성공

새로 생성된 index:

```text
security-auth-log-2026.06.22
```

확인된 타입:

```json
"source_ip": {
  "type": "ip"
}
```

index health:

```text
green
```

## 7. 오늘 기억해야 할 개념

### 7.1 Logstash parsing

Logstash parsing은 원본 로그 문자열에서 분석에 필요한 필드를 뽑는 과정입니다.

예:

```text
Failed password for invalid user admin from 192.168.0.10
```

파싱 결과:

```text
username = admin
source_ip = 192.168.0.10
event_type = ssh_failed_login
status = failed
```

### 7.2 grok

`grok`은 Logstash에서 문자열 패턴을 이용해 필드를 추출하는 filter입니다.

이번 단계에서는 auth 로그에서 다음 값을 뽑는 데 사용했습니다.

```text
auth_timestamp
host_name
process_id
username
source_ip
source_port
auth_method
sudo_command
```

### 7.3 Elasticsearch mapping

mapping은 Elasticsearch field type 설계입니다.

예:

```text
source_ip   -> ip
event_type  -> keyword
raw_message -> text
@timestamp  -> date
```

### 7.4 Index template

index template은 특정 이름 패턴의 index가 생성될 때 자동으로 적용되는 설정입니다.

이번 template 대상:

```text
security-auth-log-*
```

### 7.5 기존 index mapping은 직접 바뀌지 않는다

중요한 점:

```text
이미 생성된 index의 field type은 template을 등록해도 자동 변경되지 않는다.
```

그래서 `security-auth-log-2026.06.17`은 기존 자동 mapping을 유지하고, template 등록 이후 새로 생성된 `security-auth-log-2026.06.22`부터 올바른 mapping이 적용되었습니다.

## 8. Kibana에서 확인할 것

Kibana Data View:

```text
security-auth-log-*
```

Time field:

```text
@timestamp
```

Discover에서 확인할 필드:

```text
source_ip
username
event_type
status
service
auth_method
sudo_command
raw_message
pipeline_stage
```

시간 범위는 넓게 설정합니다.

추천:

```text
Last 7 days
Last 30 days
```

## 9. 현재 진행 상태

완료:

```text
Kafka -> Logstash -> Elasticsearch -> Kibana 기본 흐름
auth.log Logstash 파싱
security-auth-log-* index 분리
Elasticsearch index template 작성
source_ip ip 타입 검증
```

다음 단계:

```text
Kibana에서 security-auth-log-* Data View 확인
event_type/source_ip/username/status 기준 검색 연습
보안 탐지 시나리오 쿼리 작성
```

가장 가까운 다음 작업:

```text
Kibana Discover에서 ssh_failed_login, root_login, sudo_command 이벤트를 필터링해보기
```
