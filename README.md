# Security Log ETL

Linux 보안 로그를 Kafka, Logstash, Elasticsearch, Kibana로 수집/정제/저장/분석하는 로컬 ETL 학습 프로젝트입니다.

이 프로젝트의 목표는 단순히 로그를 저장하는 것이 아니라, 보안 관점에서 의미 있는 필드로 정규화하고 Kibana Dashboard에서 탐지 시나리오를 확인하는 것입니다.

## 1. 프로젝트 개요

### 목적

이 프로젝트는 정보보호 데이터 엔지니어링 직무에서 자주 다루는 로그 수집 파이프라인을 로컬 환경에서 직접 구현하기 위해 만들었습니다.

학습하는 핵심 내용은 다음과 같습니다.

```text
ETL 기본 흐름
Kafka topic 기반 로그 전달
Logstash 로그 파싱과 필드 정규화
Elasticsearch index 저장과 검색
Kibana Discover, Lens, Dashboard 활용
보안 탐지 시나리오 구현
```

### 현재 구현 범위

현재는 Linux `auth.log`를 중심으로 SSH 로그인 실패, root 로그인, sudo 명령어 사용 이벤트를 분석합니다.

```text
sample-logs/auth.log
sample-logs/brute-force-auth.log
-> Filebeat
-> Kafka security-auth-log topic
-> Logstash parsing
-> Elasticsearch security-auth-log-* index
-> Kibana Dashboard
```

초기 학습 단계에서는 Kafka console producer로 샘플 로그를 직접 넣어 데이터 흐름을 확인했고, 확장 단계에서는 Filebeat가 `auth.log` 파일을 읽어 Kafka로 자동 전송하도록 구성했습니다.

## 2. 아키텍처

전체 흐름은 다음과 같습니다.

```text
Sample Log File
  -> Filebeat
  -> Kafka Topic
  -> Logstash Pipeline
  -> Elasticsearch Index
  -> Kibana Discover / Dashboard
```

구성 요소별 역할:

| 구성 요소 | 역할 |
|---|---|
| Sample Logs | 테스트용 보안 로그 원본 |
| Filebeat | 로그 파일을 감시하고 Kafka로 전달하는 수집 agent |
| Kafka | 로그를 topic 단위로 전달하는 메시지 브로커 |
| Logstash | Kafka에서 로그를 읽고 필드를 파싱/정규화 |
| Elasticsearch | 정제된 로그를 index에 저장하고 검색/집계 제공 |
| Kibana | 로그 검색, 시각화, Dashboard 구성 |

Docker Compose 서비스:

| 서비스 | 컨테이너 이름 | 포트 | 설명 |
|---|---|---|---|
| Kafka | `etl-kafka` | `29092` | KRaft 기반 단일 노드 Kafka |
| kafka-init | `etl-kafka-init` | - | 학습용 topic 자동 생성 |
| Filebeat | `etl-filebeat` | - | `sample-logs/auth.log`를 읽어 Kafka로 전송 |
| Logstash | `etl-logstash` | `5044` | Kafka consume 후 Elasticsearch 적재 |
| Elasticsearch | `etl-elasticsearch` | `9200` | 로그 저장 및 검색 |
| Kibana | `etl-kibana` | `5601` | Dashboard 확인 |

관련 문서:

```text
docs/docker-compose-environment.md
docs/filebeat-integration.md
```

## 3. 실행 방법

### 3.1 Docker 환경 실행

프로젝트 루트에서 실행합니다.

```powershell
docker compose up -d
```

이 명령어의 역할:

```text
Kafka, Elasticsearch, Kibana, Logstash 컨테이너를 백그라운드에서 실행합니다.
```

상태 확인:

```powershell
docker compose ps
```

Elasticsearch 상태 확인:

```powershell
curl.exe http://localhost:9200/_cluster/health
```

Kibana 접속:

```text
http://localhost:5601
```

### 3.2 Kafka topic 확인

```powershell
docker exec etl-kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

예상 topic:

```text
security-auth-log
nginx-access-log
docker-container-log
```

### 3.3 샘플 로그를 Kafka에 전송

PowerShell:

```powershell
Get-Content sample-logs\auth.log | docker exec -i etl-kafka /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic security-auth-log
```

Brute Force 테스트 로그:

```powershell
Get-Content sample-logs\brute-force-auth.log | docker exec -i etl-kafka /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic security-auth-log
```

WSL/Linux에서는 `Get-Content` 대신 `cat`을 사용합니다.

```bash
cat sample-logs/brute-force-auth.log | docker exec -i etl-kafka /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic security-auth-log
```

### 3.4 Elasticsearch index 확인

```powershell
curl.exe "http://localhost:9200/_cat/indices?v"
```

예상 index:

```text
security-auth-log-YYYY.MM.DD
```

## 4. Kafka Topic

현재 설계한 topic은 다음과 같습니다.

| Topic | 역할 |
|---|---|
| `security-auth-log` | Linux 인증/보안 로그 수집 |
| `nginx-access-log` | Nginx access log 수집 예정 |
| `docker-container-log` | Docker container log 수집 예정 |

현재 Logstash는 세 topic을 모두 consume하도록 설정되어 있습니다.

```text
security-auth-log
nginx-access-log
docker-container-log
```

다만 현재 상세 파싱은 `security-auth-log` 중심으로 구현되어 있습니다. 다른 topic은 추후 확장 대상입니다.

Kafka topic 관련 학습 문서:

```text
docs/kafka-operations-concepts.md
```

## 5. Logstash 정제 방식

Logstash 설정 파일:

```text
logstash/pipeline/logstash.conf
```

Logstash는 Kafka topic에서 로그를 consume한 뒤, `security-auth-log` topic의 메시지를 조건별로 파싱합니다.

### 정규화 필드

주요 필드는 다음과 같습니다.

| 필드 | 의미 |
|---|---|
| `@timestamp` | Kibana 시간 필터 기준 |
| `auth_timestamp` | 원본 auth.log의 시간 문자열 |
| `source_ip` | 접근한 IP |
| `source_port` | 접근한 포트 |
| `username` | 로그인 또는 sudo 사용 계정 |
| `event_type` | 보안 이벤트 유형 |
| `status` | 성공/실패 상태 |
| `service` | `sshd`, `sudo` 등 서비스 |
| `log_level` | 로그 수준 |
| `raw_message` | 원본 로그 메시지 |

### event_type 분류

| 조건 | event_type |
|---|---|
| SSH 로그인 실패 | `ssh_failed_login` |
| SSH 로그인 성공 | `ssh_success_login` |
| root 로그인 성공 | `root_login` |
| sudo 명령어 사용 | `sudo_command` |

관련 문서:

```text
docs/auth-log-parsing.md
docs/log-field-analysis.md
```

## 6. Elasticsearch Index

정제된 로그는 Elasticsearch에 날짜별 index로 저장됩니다.

```text
security-auth-log-YYYY.MM.DD
```

Index template 파일:

```text
elasticsearch/mappings/security-auth-log-template.json
```

주요 mapping:

| 필드 | 타입 |
|---|---|
| `@timestamp` | `date` |
| `event_type` | `keyword` |
| `source_ip` | `ip` |
| `source_port` | `integer` |
| `username` | `keyword` |
| `status` | `keyword` |
| `sudo_command` | `keyword` |
| `raw_message` | `text` |

Elasticsearch query 파일:

```text
elasticsearch/queries/ssh-failed-login.json
elasticsearch/queries/root-login.json
elasticsearch/queries/sudo-command.json
elasticsearch/queries/failed-login-by-ip.json
elasticsearch/queries/brute-force-5m.json
```

query 실행 예시:

```powershell
curl.exe -s -X GET "http://localhost:9200/security-auth-log-*/_search?pretty" -H "Content-Type: application/json" --data-binary "@elasticsearch/queries/brute-force-5m.json"
```

관련 문서:

```text
docs/elasticsearch-index-mapping.md
docs/query-files-guide.md
```

## 7. 탐지 시나리오

현재 구현한 탐지 시나리오는 다음과 같습니다.

### 7.1 SSH 로그인 실패 탐지

KQL:

```text
event_type: ssh_failed_login
```

의미:

```text
SSH 로그인 실패 이벤트를 조회합니다.
```

### 7.2 Brute Force 의심 탐지

조건:

```text
5분 내 같은 source_ip에서 SSH 로그인 실패 20회 이상
```

Query 파일:

```text
elasticsearch/queries/brute-force-5m.json
```

의미:

```text
짧은 시간 안에 같은 IP에서 로그인 실패가 반복되면 무차별 대입 공격 가능성이 있습니다.
```

### 7.3 root 계정 로그인 탐지

KQL:

```text
event_type: root_login
```

의미:

```text
관리자 계정 직접 로그인 발생 여부를 확인합니다.
```

### 7.4 sudo 명령어 사용 탐지

KQL:

```text
event_type: sudo_command
```

의미:

```text
권한 상승 명령어 사용 기록을 확인합니다.
```

### 7.5 IP별 과도한 로그인 실패 탐지

Query 파일:

```text
elasticsearch/queries/failed-login-by-ip.json
```

의미:

```text
어떤 IP가 로그인 실패를 많이 발생시키는지 집계합니다.
```

## 8. Kibana Dashboard

Kibana Data View:

```text
security_logs2
```

대상 index pattern:

```text
security-auth-log-*
```

현재 Dashboard는 다음 6개 패널로 구성됩니다.

| 번호 | 패널 | 방식 | 문서 |
|---|---|---|---|
| 1 | 시간대별 SSH 로그인 실패 추이 | Lens chart | `docs/dashboards/01-ssh-failed-login-trend.md` |
| 2 | SSH 로그인 실패 TOP IP | Lens chart | `docs/dashboards/02-ssh-failed-login-top-ip.md` |
| 3 | 계정별 SSH 로그인 실패 횟수 | Lens chart | `docs/dashboards/03-ssh-failed-login-by-username.md` |
| 4 | root 로그인 발생 목록 | Discover Saved Search | `docs/dashboards/04-root-login-list.md` |
| 5 | sudo 명령어 사용 목록 | Discover Saved Search | `docs/dashboards/05-sudo-command-list.md` |
| 6 | 위험 이벤트 목록 | Discover Saved Search | `docs/dashboards/06-risk-events-list.md` |

### Lens와 Discover 사용 기준

```text
통계/순위/추이 = Lens
원본 로그 목록 = Discover Saved Search
```

관련 문서:

```text
docs/kibana-dashboard-guide.md
docs/kibana-filtering-and-detection-queries.md
docs/dashboards/
```

## 9. 트러블슈팅

### 9.1 WSL에서 Get-Content 오류

오류:

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

### 9.2 같은 샘플 로그를 여러 번 넣은 경우

현상:

```text
Kibana에서 같은 IP의 실패 횟수가 예상보다 크게 보입니다.
```

원인:

```text
Kafka producer 명령을 실행할 때마다 같은 로그가 새 메시지로 들어갑니다.
Elasticsearch에도 새 문서로 저장됩니다.
```

학습 포인트:

```text
실무에서는 중복 이벤트 처리, idempotency, event_id 설계가 중요합니다.
```

### 9.3 source_ip와 source_ip.keyword 차이

정리:

```text
source_ip = 실제 IP 값을 표시하는 필드
source_ip.keyword = 집계와 그룹화에 사용하는 하위 필드
```

사용 기준:

```text
Discover 목록 = source_ip 사용 가능
Lens TOP IP 집계 = source_ip.keyword 사용
```

### 9.4 Kibana Lens Table에서 Top values가 자동으로 생김

현상:

```text
Rows에 필드를 넣으면 Top 3 values of username 같은 설정이 생깁니다.
```

원인:

```text
Lens Table은 원본 로그 목록이 아니라 집계 기반 시각화입니다.
```

해결:

```text
원본 로그 목록 패널은 Discover에서 저장한 Table을 Dashboard에 추가합니다.
```

### 9.5 단일 노드 Elasticsearch에서 yellow 상태

현상:

```text
cluster health가 yellow로 표시될 수 있습니다.
```

원인:

```text
단일 노드 환경에서 replica shard를 할당할 다른 노드가 없기 때문입니다.
```

해결:

```text
학습용 단일 노드에서는 number_of_replicas를 0으로 설정합니다.
```

## 10. 다음 확장 방향

현재 프로젝트는 단일 노드 로컬 학습 환경입니다. 이후에는 다음 방향으로 확장할 수 있습니다.

```text
nginx-access-log 파싱 구현
docker-container-log 파싱 구현
Kafka broker와 replication factor 증가
Elasticsearch index lifecycle management 적용
Kibana Alerting 또는 Detection Rule 구성
README에 Dashboard 캡처 추가
```

## 11. 참고 문서

프로젝트 학습 기록:

```text
docs/daily-work-2026-06-22.md
docs/daily-work-2026-06-25.md
docs/daily-work-2026-06-30.md
```

다시 시작 체크리스트:

```text
docs/resume-environment-checklist.md
```
