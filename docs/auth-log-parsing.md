# auth.log Logstash 파싱 단계

## 1. 이 단계의 목표

이 단계의 목표는 `security-auth-log` topic으로 들어온 Linux 인증 로그를 Logstash에서 구조화된 필드로 분리하는 것입니다.

이전 단계에서는 로그가 Elasticsearch에 다음처럼 원본 형태로만 저장되었습니다.

```json
{
  "message": "원본 로그",
  "raw_message": "원본 로그",
  "pipeline_stage": "raw_ingest"
}
```

이번 단계 이후에는 다음처럼 분석 가능한 필드가 추가됩니다.

```json
{
  "source_ip": "192.168.0.10",
  "username": "admin",
  "event_type": "ssh_failed_login",
  "status": "failed",
  "service": "sshd",
  "raw_message": "원본 로그"
}
```

## 2. 내가 하는 행동과 영향

### 2.1 Logstash 설정 수정

수정 파일:

```text
logstash/pipeline/logstash.conf
```

이 파일을 수정하면 Logstash가 Kafka에서 읽은 로그를 어떻게 해석하고 어디에 저장할지 바뀝니다.

이번 수정의 영향:

1. Kafka message의 topic 정보를 Logstash metadata로 가져옵니다.
2. `security-auth-log` topic에서 온 로그만 auth 로그 파싱 규칙을 적용합니다.
3. SSH 로그인 실패, SSH 로그인 성공, sudo 명령어 사용 로그를 구분합니다.
4. `source_ip`, `username`, `event_type`, `status`, `service` 필드를 생성합니다.
5. auth 로그는 `security-auth-log-YYYY.MM.dd` index에 저장합니다.

### 2.2 Logstash 재시작

실행 명령:

```powershell
docker compose restart logstash
```

이 명령은 Logstash 컨테이너를 재시작해 수정한 pipeline 설정을 다시 읽게 만듭니다.

영향:

1. Kafka, Elasticsearch, Kibana 컨테이너는 그대로 유지됩니다.
2. Elasticsearch에 저장된 기존 데이터는 삭제되지 않습니다.
3. Logstash만 새 설정으로 다시 시작합니다.

### 2.3 Kafka에 auth 로그 전송

실행 명령:

```powershell
docker exec etl-kafka /bin/bash -c "printf '%s\n' 'Jun 17 21:10:33 ubuntu sshd[1234]: Failed password for invalid user admin from 192.168.0.10 port 54321 ssh2' 'Jun 17 21:11:02 ubuntu sshd[1235]: Failed password for root from 203.0.113.10 port 49812 ssh2' 'Jun 17 21:12:15 ubuntu sshd[1236]: Accepted password for ubuntu from 192.168.0.20 port 50222 ssh2' 'Jun 17 21:13:44 ubuntu sudo:   ubuntu : TTY=pts/0 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/bin/apt update' 'Jun 17 21:14:09 ubuntu sshd[1237]: Accepted publickey for root from 198.51.100.7 port 60123 ssh2' | /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic security-auth-log"
```

이 명령은 Filebeat가 아직 없는 현재 단계에서 auth 로그를 수동으로 Kafka에 넣는 테스트입니다.

영향:

```text
Kafka topic security-auth-log
-> Logstash consume
-> auth 로그 파싱
-> Elasticsearch security-auth-log-* index 저장
```

## 3. 이번에 파싱한 로그 유형

### 3.1 SSH 로그인 실패

예시:

```text
Jun 17 21:10:33 ubuntu sshd[1234]: Failed password for invalid user admin from 192.168.0.10 port 54321 ssh2
```

생성 필드:

| 필드 | 값 |
|---|---|
| service | sshd |
| event_type | ssh_failed_login |
| status | failed |
| username | admin |
| source_ip | 192.168.0.10 |
| source_port | 54321 |

### 3.2 SSH 로그인 성공

예시:

```text
Jun 17 21:12:15 ubuntu sshd[1236]: Accepted password for ubuntu from 192.168.0.20 port 50222 ssh2
```

생성 필드:

| 필드 | 값 |
|---|---|
| service | sshd |
| event_type | ssh_success_login |
| status | success |
| username | ubuntu |
| source_ip | 192.168.0.20 |
| auth_method | password |

### 3.3 root 로그인 성공

예시:

```text
Jun 17 21:14:09 ubuntu sshd[1237]: Accepted publickey for root from 198.51.100.7 port 60123 ssh2
```

생성 필드:

| 필드 | 값 |
|---|---|
| service | sshd |
| event_type | root_login |
| status | success |
| username | root |
| source_ip | 198.51.100.7 |
| auth_method | publickey |

`username`이 `root`이고 `status`가 `success`이면 `event_type`을 `root_login`으로 바꿉니다.

### 3.4 sudo 명령어 사용

예시:

```text
Jun 17 21:13:44 ubuntu sudo:   ubuntu : TTY=pts/0 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/bin/apt update
```

생성 필드:

| 필드 | 값 |
|---|---|
| service | sudo |
| event_type | sudo_command |
| status | success |
| username | ubuntu |
| sudo_command | /usr/bin/apt update |

## 4. 확인 명령어

### 4.1 Logstash 상태 확인

```powershell
docker logs --tail 80 etl-logstash
```

정상적으로 Kafka topic을 읽을 준비가 되면 다음과 비슷한 로그가 보입니다.

```text
Successfully joined group
Adding newly assigned partitions
security-auth-log-0
```

### 4.2 Elasticsearch index 확인

```powershell
curl.exe -s "http://localhost:9200/_cat/indices/security-auth-log-*?v"
```

성공 예시:

```text
health status index                        docs.count
yellow open   security-auth-log-2026.06.17 5
```

현재 로컬 Elasticsearch는 단일 노드인데 replica 설정이 기본 1이기 때문에 `yellow`가 나올 수 있습니다.

단일 노드에서는 replica shard를 다른 노드에 배치할 수 없어서 발생하는 상태입니다.

학습 환경에서는 index가 열려 있고 검색이 되면 다음 단계 진행에는 문제 없습니다.

### 4.3 파싱 결과 확인

```powershell
curl.exe -s "http://localhost:9200/security-auth-log-*/_search?pretty&size=10"
```

확인할 필드:

```text
source_ip
username
event_type
status
service
raw_message
pipeline_stage
```

성공 예시:

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

## 5. Kibana에서 확인하는 방법

Kibana 접속:

```text
http://localhost:5601
```

Data View 생성:

```text
Stack Management
-> Data Views
-> Create data view
```

입력값:

```text
Index pattern: security-auth-log-*
Time field: @timestamp
```

Discover 이동:

```text
Analytics -> Discover
```

또는:

```text
http://localhost:5601/app/discover
```

Discover에서 `security-auth-log-*` Data View를 선택한 뒤 다음 필드를 확인합니다.

```text
source_ip
username
event_type
status
service
sudo_command
auth_method
```

## 6. 기억해야 할 개념

### 6.1 grok

`grok`은 문자열 로그에서 원하는 값을 뽑아내는 Logstash filter입니다.

예:

```text
Failed password for invalid user admin from 192.168.0.10
```

뽑는 값:

```text
username = admin
source_ip = 192.168.0.10
```

### 6.2 event_type

`event_type`은 원본 로그 문장을 분석하기 쉬운 이벤트 이름으로 바꾼 값입니다.

예:

```text
Failed password   -> ssh_failed_login
Accepted password -> ssh_success_login
sudo              -> sudo_command
root success      -> root_login
```

### 6.3 status

`status`는 이벤트의 성공/실패 상태입니다.

예:

```text
Failed password   -> failed
Accepted password -> success
sudo command      -> success
```

### 6.4 topic 기반 분기

Logstash는 Kafka에서 읽은 메시지가 어떤 topic에서 왔는지 확인할 수 있습니다.

이번 설정에서는 `security-auth-log`에서 온 로그만 auth 파싱을 적용합니다.

이 방식은 나중에 다음처럼 확장할 수 있습니다.

```text
security-auth-log       -> auth 로그 파싱
nginx-access-log        -> nginx 로그 파싱
docker-container-log    -> docker 로그 파싱
```

### 6.5 index 분리

기존에는 모든 로그가 하나의 raw index에 들어갔습니다.

```text
etl-raw-log-YYYY.MM.dd
```

이번 단계부터 auth 로그는 별도 index에 저장됩니다.

```text
security-auth-log-YYYY.MM.dd
```

이렇게 분리하면 Kibana에서 인증 로그만 따로 분석하고 대시보드를 만들기 쉬워집니다.

## 7. 다음 단계

다음 단계는 Elasticsearch index mapping을 작성하는 것입니다.

현재는 Elasticsearch가 필드 타입을 자동 추론합니다.

다음에는 명시적으로 타입을 지정합니다.

예:

```text
source_ip   -> ip
username    -> keyword
event_type  -> keyword
status      -> keyword
@timestamp  -> date
raw_message -> text
```

이 작업을 하면 검색, 집계, Kibana 시각화가 더 안정적으로 동작합니다.
