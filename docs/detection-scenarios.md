# 탐지 시나리오 정리

이 문서는 `Security-Log-ETL` 프로젝트에서 구현한 보안 탐지 시나리오를 한곳에 모아 정리한 문서입니다.

기존에는 탐지 조건이 README, Kibana 문서, Dashboard 문서, Elasticsearch query 파일에 나누어져 있었습니다. 이 문서는 면접과 포트폴리오 설명을 위해 각 탐지 시나리오의 목적, 조건, 사용하는 필드, KQL, Elasticsearch query, Dashboard 연결 관계를 한 번에 볼 수 있도록 구성했습니다.

## 1. 탐지 시나리오 개요

현재 구현한 탐지 시나리오는 다음과 같습니다.

| 번호 | 시나리오 | 목적 | 상태 |
|---|---|---|---|
| 1 | SSH 로그인 실패 탐지 | 인증 실패 이벤트 확인 | 구현 완료 |
| 2 | Brute Force 의심 탐지 | 짧은 시간 내 반복 로그인 실패 탐지 | 구현 완료 |
| 3 | root 계정 로그인 탐지 | 관리자 계정 직접 접근 확인 | 구현 완료 |
| 4 | sudo 명령어 사용 탐지 | 권한 상승 행위 확인 | 구현 완료 |
| 5 | IP별 과도한 로그인 실패 탐지 | 공격 가능성이 높은 IP 식별 | 구현 완료 |
| 6 | 위험 이벤트 통합 조회 | 주요 보안 이벤트를 한 화면에서 확인 | 구현 완료 |

## 2. 탐지에 사용하는 주요 필드

Logstash가 `auth.log`를 파싱해 Elasticsearch에 저장하는 주요 필드는 다음과 같습니다.

| 필드 | 의미 | 예시 |
|---|---|---|
| `@timestamp` | Kibana 시간 기준 필드 | `2026-06-25T21:10:20.000+09:00` |
| `event_type` | 보안 이벤트 유형 | `ssh_failed_login` |
| `source_ip` | 접근한 IP | `203.0.113.200` |
| `source_port` | 접근한 포트 | `42822` |
| `username` | 로그인 또는 명령 실행 계정 | `root`, `attacker` |
| `status` | 성공/실패 상태 | `success`, `failed` |
| `service` | 로그를 발생시킨 서비스 | `sshd`, `sudo` |
| `auth_method` | SSH 인증 방식 | `password`, `publickey` |
| `sudo_command` | sudo로 실행한 명령어 | `/usr/bin/apt update` |
| `raw_message` | 원본 로그 메시지 | `Failed password ...` |

관련 문서:

```text
docs/auth-log-parsing.md
docs/log-field-analysis.md
docs/elasticsearch-index-mapping.md
```

## 3. 시나리오 1. SSH 로그인 실패 탐지

### 3.1 목적

SSH 로그인 실패 이벤트를 탐지합니다.

로그인 실패는 단순한 사용자 실수일 수도 있지만, 같은 IP 또는 같은 계정에서 반복되면 공격 시도로 볼 수 있습니다.

### 3.2 탐지 조건

```text
event_type = ssh_failed_login
```

### 3.3 Kibana KQL

```text
event_type: ssh_failed_login
```

### 3.4 사용하는 필드

```text
@timestamp
event_type
source_ip
username
status
service
raw_message
```

### 3.5 Elasticsearch query 파일

```text
elasticsearch/queries/ssh-failed-login.json
```

실행 예시:

```powershell
curl.exe -s -X GET "http://localhost:9200/security-auth-log-*/_search?pretty" -H "Content-Type: application/json" --data-binary "@elasticsearch/queries/ssh-failed-login.json"
```

### 3.6 Dashboard 연결

```text
docs/dashboards/01-ssh-failed-login-trend.md
docs/dashboards/02-ssh-failed-login-top-ip.md
docs/dashboards/03-ssh-failed-login-by-username.md
docs/dashboards/06-risk-events-list.md
```

## 4. 시나리오 2. Brute Force 의심 탐지

### 4.1 목적

짧은 시간 안에 같은 IP에서 SSH 로그인 실패가 반복되는지 탐지합니다.

Brute Force 공격은 여러 비밀번호를 빠르게 시도하는 방식이므로, 단순 실패 횟수보다 시간 범위와 IP 기준 집계가 중요합니다.

### 4.2 탐지 조건

```text
5분 내 같은 source_ip에서 SSH 로그인 실패 20회 이상
```

### 4.3 Kibana KQL

Kibana Discover에서는 우선 SSH 로그인 실패만 필터링합니다.

```text
event_type: ssh_failed_login
```

5분 내 20회 이상 조건은 Elasticsearch aggregation query로 확인합니다.

### 4.4 사용하는 필드

```text
@timestamp
event_type
source_ip
status
username
```

### 4.5 Elasticsearch query 파일

```text
elasticsearch/queries/brute-force-5m.json
```

실행 예시:

```powershell
curl.exe -s -X GET "http://localhost:9200/security-auth-log-*/_search?pretty" -H "Content-Type: application/json" --data-binary "@elasticsearch/queries/brute-force-5m.json"
```

### 4.6 query 동작 방식

이 query는 다음 순서로 동작합니다.

```text
1. event_type이 ssh_failed_login인 로그만 조회
2. @timestamp를 5분 단위로 나눔
3. 각 시간 구간 안에서 source_ip별로 그룹화
4. 같은 source_ip에서 20건 이상 발생한 경우만 결과로 표시
```

### 4.7 테스트 로그

테스트용 로그 파일:

```text
sample-logs/brute-force-auth.log
```

Kafka 전송 명령:

```powershell
Get-Content sample-logs\brute-force-auth.log | docker exec -i etl-kafka /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic security-auth-log
```

WSL/Linux:

```bash
cat sample-logs/brute-force-auth.log | docker exec -i etl-kafka /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic security-auth-log
```

### 4.8 Dashboard 연결

```text
docs/dashboards/01-ssh-failed-login-trend.md
docs/dashboards/02-ssh-failed-login-top-ip.md
docs/dashboards/06-risk-events-list.md
```

## 5. 시나리오 3. root 계정 로그인 탐지

### 5.1 목적

`root` 계정으로 로그인한 이벤트를 탐지합니다.

`root`는 Linux에서 가장 높은 권한을 가진 관리자 계정입니다. 따라서 root 직접 로그인은 정상 작업인지 비정상 접근인지 확인해야 하는 이벤트입니다.

### 5.2 탐지 조건

```text
username = root
status = success
event_type = root_login
```

현재 Logstash pipeline에서는 SSH 로그인 성공 이벤트 중 `username`이 `root`이고 `status`가 `success`이면 `event_type`을 `root_login`으로 변경합니다.

### 5.3 Kibana KQL

```text
event_type: root_login
```

### 5.4 사용하는 필드

```text
@timestamp
event_type
source_ip
username
status
auth_method
raw_message
```

### 5.5 Elasticsearch query 파일

```text
elasticsearch/queries/root-login.json
```

실행 예시:

```powershell
curl.exe -s -X GET "http://localhost:9200/security-auth-log-*/_search?pretty" -H "Content-Type: application/json" --data-binary "@elasticsearch/queries/root-login.json"
```

### 5.6 Dashboard 연결

```text
docs/dashboards/04-root-login-list.md
docs/dashboards/06-risk-events-list.md
```

## 6. 시나리오 4. sudo 명령어 사용 탐지

### 6.1 목적

`sudo` 명령어 사용 이벤트를 탐지합니다.

`sudo`는 일반 사용자가 관리자 권한으로 명령을 실행할 때 사용합니다. 따라서 sudo 사용 로그는 권한 상승 행위 추적에 중요합니다.

### 6.2 탐지 조건

```text
event_type = sudo_command
```

### 6.3 Kibana KQL

```text
event_type: sudo_command
```

### 6.4 사용하는 필드

```text
@timestamp
event_type
username
sudo_command
status
service
raw_message
```

### 6.5 Elasticsearch query 파일

```text
elasticsearch/queries/sudo-command.json
```

실행 예시:

```powershell
curl.exe -s -X GET "http://localhost:9200/security-auth-log-*/_search?pretty" -H "Content-Type: application/json" --data-binary "@elasticsearch/queries/sudo-command.json"
```

### 6.6 Dashboard 연결

```text
docs/dashboards/05-sudo-command-list.md
docs/dashboards/06-risk-events-list.md
```

## 7. 시나리오 5. IP별 과도한 로그인 실패 탐지

### 7.1 목적

SSH 로그인 실패가 많이 발생한 IP를 집계합니다.

이 시나리오는 어떤 IP가 공격 가능성이 높은지 빠르게 파악하기 위한 것입니다.

### 7.2 탐지 조건

```text
event_type = ssh_failed_login
source_ip별 count 집계
count 기준 내림차순 정렬
```

### 7.3 Kibana KQL

```text
event_type: ssh_failed_login
```

### 7.4 사용하는 필드

```text
source_ip
event_type
status
@timestamp
```

### 7.5 Elasticsearch query 파일

```text
elasticsearch/queries/failed-login-by-ip.json
```

실행 예시:

```powershell
curl.exe -s -X GET "http://localhost:9200/security-auth-log-*/_search?pretty" -H "Content-Type: application/json" --data-binary "@elasticsearch/queries/failed-login-by-ip.json"
```

### 7.6 query 동작 방식

```text
1. event_type이 ssh_failed_login인 로그만 조회
2. source_ip 기준으로 그룹화
3. 실패 횟수가 많은 IP부터 정렬
```

현재 query는 `runtime_mappings`의 `source_ip_group`을 사용합니다. 이는 초기 index와 이후 index의 `source_ip` mapping 차이를 피하기 위해 임시 집계 필드를 만드는 방식입니다.

### 7.7 Dashboard 연결

```text
docs/dashboards/02-ssh-failed-login-top-ip.md
docs/dashboards/06-risk-events-list.md
```

## 8. 시나리오 6. 위험 이벤트 통합 조회

### 8.1 목적

주요 보안 이벤트를 한 화면에서 목록으로 확인합니다.

개별 탐지 시나리오가 특정 이벤트를 분석하는 용도라면, 위험 이벤트 목록은 보안 관제 화면처럼 여러 이벤트를 통합해서 보는 용도입니다.

### 8.2 탐지 조건

```text
event_type = ssh_failed_login
or event_type = root_login
or event_type = sudo_command
```

### 8.3 Kibana KQL

```text
event_type: ssh_failed_login or event_type: root_login or event_type: sudo_command
```

### 8.4 사용하는 필드

```text
@timestamp
raw_message
status
event_type
source_ip
username
service
```

### 8.5 Dashboard 연결

```text
docs/dashboards/06-risk-events-list.md
```

## 9. 탐지 시나리오와 Dashboard 연결 요약

| 탐지 시나리오 | Dashboard 패널 | 시각화 방식 |
|---|---|---|
| SSH 로그인 실패 탐지 | 시간대별 SSH 로그인 실패 추이 | Lens chart |
| IP별 과도한 로그인 실패 탐지 | SSH 로그인 실패 TOP IP | Lens chart |
| 계정별 로그인 실패 분석 | 계정별 SSH 로그인 실패 횟수 | Lens chart |
| root 계정 로그인 탐지 | root 로그인 발생 목록 | Discover Saved Search |
| sudo 명령어 사용 탐지 | sudo 명령어 사용 목록 | Discover Saved Search |
| 위험 이벤트 통합 조회 | 위험 이벤트 목록 | Discover Saved Search |

## 10. 탐지 시나리오와 파일 연결 요약

| 목적 | 파일 |
|---|---|
| Kibana KQL과 탐지 query 학습 | `docs/kibana-filtering-and-detection-queries.md` |
| query 파일 실행 방법 | `docs/query-files-guide.md` |
| Logstash 파싱 규칙 | `logstash/pipeline/logstash.conf` |
| Elasticsearch mapping | `elasticsearch/mappings/security-auth-log-template.json` |
| SSH 실패 조회 query | `elasticsearch/queries/ssh-failed-login.json` |
| root 로그인 조회 query | `elasticsearch/queries/root-login.json` |
| sudo 사용 조회 query | `elasticsearch/queries/sudo-command.json` |
| IP별 실패 집계 query | `elasticsearch/queries/failed-login-by-ip.json` |
| Brute Force 탐지 query | `elasticsearch/queries/brute-force-5m.json` |

## 11. 면접 설명 예시

이 프로젝트의 탐지 시나리오는 다음처럼 설명할 수 있습니다.

```text
Linux auth.log를 Logstash에서 파싱해 event_type, source_ip, username, status 같은 구조화 필드를 만들었습니다.
이후 Elasticsearch와 Kibana에서 event_type 기반으로 SSH 로그인 실패, root 로그인, sudo 명령어 사용 이벤트를 탐지했습니다.
또한 5분 단위 date_histogram과 source_ip terms aggregation을 사용해 같은 IP에서 SSH 로그인 실패가 20회 이상 발생하는 Brute Force 의심 조건을 구현했습니다.
마지막으로 Kibana Dashboard에서 시간대별 추이, TOP 공격 IP, 계정별 실패 횟수, root/sudo/위험 이벤트 목록을 시각화했습니다.
```

## 12. 다음 확장 방향

Filebeat 실제 수집 연결 이후에는 다음 탐지 시나리오를 확장할 수 있습니다.

```text
nginx access log 기반 과도한 요청 IP 탐지
관리자 URL 접근 시도 탐지
HTTP 404/500 에러 급증 탐지
docker container error 로그 급증 탐지
sudo 위험 명령어 risk_score 부여
Kibana Alerting 또는 Detection Rule 연결
```

