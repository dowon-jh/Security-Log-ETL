# 2026-07-02 Filebeat 실제 수집 연결 작업 정리

## 1. 오늘 작업의 목표

오늘 작업의 목표는 기존에 사람이 직접 Kafka에 로그를 넣던 방식을 Filebeat 기반 자동 수집 방식으로 확장하는 것이었습니다.

기존 흐름:

```text
sample-logs/auth.log
-> kafka-console-producer
-> Kafka security-auth-log topic
-> Logstash
-> Elasticsearch
-> Kibana
```

확장 후 흐름:

```text
sample-logs/auth.log
-> Filebeat
-> Kafka security-auth-log topic
-> Logstash
-> Elasticsearch
-> Kibana
```

## 2. 추가한 구성

### filebeat/filebeat.yml

Filebeat가 어떤 로그 파일을 읽고, 어느 Kafka topic으로 보낼지 정의하는 설정 파일입니다.

주요 설정:

```yaml
filebeat.inputs:
  - type: filestream
    paths:
      - /var/log/security-samples/auth.log

output.kafka:
  hosts: ["kafka:9092"]
  topic: "security-auth-log"
  codec.format:
    string: "%{[message]}"
```

이 설정은 Filebeat가 `auth.log` 파일을 읽고, 로그 원문 한 줄만 Kafka의 `security-auth-log` topic으로 보내도록 합니다.

### docker-compose.yml

`filebeat` 서비스를 추가했습니다.

Filebeat 컨테이너는 로컬의 `sample-logs` 폴더를 컨테이너 내부의 `/var/log/security-samples` 경로로 마운트합니다.

```text
로컬: sample-logs/auth.log
컨테이너 내부: /var/log/security-samples/auth.log
```

## 3. 중요한 학습 개념

### Filebeat

Filebeat는 서버의 로그 파일을 감시하고, 새로 추가된 로그를 Kafka, Logstash, Elasticsearch 같은 외부 시스템으로 전달하는 경량 수집 agent입니다.

이번 프로젝트에서는 Filebeat가 Extract 단계에 가까운 역할을 합니다.

### Kafka를 중간에 둔 이유

Kafka는 단순 메시지 큐라기보다 이벤트 스트리밍 플랫폼입니다.

이 프로젝트에서 Kafka를 사용한 이유는 다음과 같습니다.

```text
로그 수집과 처리 단계를 분리
Logstash 장애 시 로그 유실 가능성 완화
Kafka topic에 남은 데이터를 재처리 가능
topic 단위로 로그 종류를 분리하고 확장 가능
```

### Filebeat JSON 이벤트와 Logstash Grok

Filebeat는 기본적으로 로그 원문뿐 아니라 파일 경로, agent 정보, timestamp 같은 부가 정보를 포함한 구조화된 이벤트를 만들 수 있습니다.

하지만 현재 Logstash grok 설정은 다음과 같은 auth.log 원문 한 줄을 기준으로 파싱합니다.

```text
Jun 17 21:10:33 ubuntu sshd[1234]: Failed password for invalid user admin from 192.168.0.10 port 54321 ssh2
```

그래서 Filebeat 설정에서 다음 옵션을 사용했습니다.

```yaml
codec.format:
  string: "%{[message]}"
```

이 설정은 Filebeat 이벤트 전체가 아니라 `message` 필드에 들어 있는 로그 원문만 Kafka로 보내게 합니다.

## 4. 확인한 실행 흐름

Docker Compose 실행:

```bash
docker compose up -d
```

Filebeat 상태 확인:

```bash
docker logs etl-filebeat --tail 50
```

Logstash 상태 확인:

```bash
docker logs etl-logstash --tail 80
```

Elasticsearch index 확인:

```bash
curl "http://localhost:9200/_cat/indices/security-auth-log-*?v"
```

Kibana Discover 확인:

```text
Data View: security_logs2
KQL: username: filebeattest
```

## 5. 오늘 작업의 의미

이번 작업으로 프로젝트는 단순한 샘플 로그 수동 주입 방식에서 실제 로그 수집 구조에 가까운 형태로 확장되었습니다.

면접에서는 다음처럼 설명할 수 있습니다.

```text
초기에는 Kafka console producer를 사용해 샘플 로그를 수동으로 주입했습니다.
이후 Filebeat를 추가하여 auth.log 파일 변경을 자동 감지하고 Kafka topic으로 전송하도록 확장했습니다.
이를 통해 로그 수집 agent, Kafka topic, Logstash 정제, Elasticsearch 저장, Kibana 분석으로 이어지는 실제 보안 로그 ETL 흐름을 구현했습니다.
```

## 6. 다음 단계

다음 확장 단계는 다음 중 하나로 진행할 수 있습니다.

```text
nginx-access-log 파싱 구현
docker-container-log 파싱 구현
Filebeat input을 여러 로그 파일로 확장
Kafka topic별 Logstash pipeline 분리
Kibana Dashboard에 Filebeat 기반 수집 흐름 반영
```
