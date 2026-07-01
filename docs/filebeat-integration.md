# Filebeat 실제 수집 연결

## 1. 이번 단계의 목표

이 단계의 목표는 수동으로 Kafka에 로그를 넣던 방식을 Filebeat 기반 자동 수집 방식으로 확장하는 것입니다.

기존 방식:

```text
sample-logs/auth.log
-> kafka-console-producer
-> Kafka security-auth-log topic
-> Logstash
-> Elasticsearch
-> Kibana
```

확장 방식:

```text
sample-logs/auth.log
-> Filebeat
-> Kafka security-auth-log topic
-> Logstash
-> Elasticsearch
-> Kibana
```

## 2. Filebeat의 역할

Filebeat는 로그 파일을 읽어서 Kafka, Logstash, Elasticsearch 같은 외부 시스템으로 전달하는 경량 수집 agent입니다.

이번 프로젝트에서는 Filebeat가 다음 파일을 읽습니다.

```text
sample-logs/auth.log
```

Docker 컨테이너 내부에서는 다음 경로로 마운트됩니다.

```text
/var/log/security-samples/auth.log
```

## 3. 추가한 파일

```text
filebeat/filebeat.yml
```

이 파일은 Filebeat가 어떤 로그 파일을 읽고, 어디로 보낼지 정의합니다.

## 4. filebeat.yml 설정

```yaml
filebeat.inputs:
  - type: filestream
    id: security-auth-log-sample
    enabled: true
    paths:
      - /var/log/security-samples/auth.log
```

의미:

```text
Filebeat가 컨테이너 내부의 /var/log/security-samples/auth.log 파일을 읽습니다.
```

Kafka output:

```yaml
output.kafka:
  hosts: ["kafka:9092"]
  topic: "security-auth-log"
  required_acks: 1
  codec.format:
    string: "%{[message]}"
```

의미:

```text
읽은 로그를 Kafka의 security-auth-log topic으로 전송합니다.
```

`codec.format.string`을 사용하는 이유:

```text
Filebeat는 기본적으로 Kafka에 JSON 형태의 이벤트를 보낼 수 있습니다.
하지만 기존 Logstash grok 설정은 auth.log 원문 한 줄을 기준으로 파싱합니다.
따라서 message 필드만 Kafka에 보내도록 설정했습니다.
```

## 5. Filebeat JSON 이벤트와 auth.log 원문 한 줄의 차이

이번 설정에서 가장 중요한 부분은 Filebeat가 Kafka에 어떤 형태의 데이터를 보내는지입니다.

현재 Logstash grok 패턴은 다음과 같은 auth.log 원문 한 줄을 기대합니다.

```text
Jun 17 21:10:33 ubuntu sshd[1234]: Failed password for invalid user admin from 192.168.0.10 port 54321 ssh2
```

Logstash는 이 한 줄을 보고 다음 필드를 분리합니다.

```text
Jun 17 21:10:33  -> auth_timestamp
ubuntu           -> host_name
sshd             -> service
admin            -> username
192.168.0.10     -> source_ip
54321            -> source_port
Failed password  -> event_type: ssh_failed_login
```

즉, 현재 Logstash 파이프라인은 "순수한 로그 원문 한 줄"을 기준으로 만들어져 있습니다.

하지만 Filebeat는 로그 파일을 읽을 때 로그 원문뿐만 아니라 여러 부가정보를 함께 가진 이벤트를 만들 수 있습니다.

예시:

```json
{
  "@timestamp": "2026-07-01T12:00:00Z",
  "message": "Jun 17 21:10:33 ubuntu sshd[1234]: Failed password for invalid user admin from 192.168.0.10 port 54321 ssh2",
  "log": {
    "file": {
      "path": "/var/log/security-samples/auth.log"
    }
  },
  "host": {
    "name": "filebeat-container"
  },
  "agent": {
    "type": "filebeat"
  }
}
```

이 JSON 전체가 Kafka로 들어가면 Logstash가 받는 메시지는 auth.log 한 줄이 아니라 JSON 문자열 전체가 될 수 있습니다.

그 경우 Logstash grok은 다음과 같은 형태를 받게 됩니다.

```text
{"@timestamp":"...","message":"Jun 17 21:10:33 ubuntu sshd[1234]: Failed password ..."}
```

하지만 grok 패턴은 다음 형태를 기대합니다.

```text
Jun 17 21:10:33 ubuntu sshd[1234]: Failed password ...
```

따라서 JSON 전체가 들어오면 기존 grok 파싱이 실패할 수 있습니다.

이를 피하기 위해 Filebeat 설정에서 다음 옵션을 사용했습니다.

```yaml
codec.format:
  string: "%{[message]}"
```

이 설정의 의미:

```text
Filebeat 이벤트 전체를 Kafka에 보내지 말고,
그 이벤트 안의 message 필드 값만 꺼내서 Kafka로 보내라.
```

즉, Kafka에는 다음 JSON 전체가 아니라:

```json
{
  "@timestamp": "...",
  "message": "Jun 17 21:10:33 ubuntu sshd[1234]: Failed password ...",
  "agent": "...",
  "log": "..."
}
```

다음 로그 원문 한 줄만 들어갑니다.

```text
Jun 17 21:10:33 ubuntu sshd[1234]: Failed password ...
```

이렇게 하면 기존 Logstash grok 설정을 크게 바꾸지 않고 Filebeat를 연결할 수 있습니다.

정리:

| 구분 | 의미 |
|---|---|
| Filebeat 기본 이벤트 | 로그 원문과 부가정보를 포함한 구조화 이벤트 |
| Logstash 기존 grok | auth.log 원문 한 줄을 기준으로 파싱 |
| `codec.format.string` | Filebeat 이벤트 중 `message` 필드만 Kafka로 전송 |

비유하면 다음과 같습니다.

```text
Logstash는 종이 한 장에 적힌 로그 한 줄을 읽는 방식으로 준비되어 있습니다.
그런데 Filebeat가 봉투 안에 여러 정보와 함께 로그를 넣어 보내면 Logstash가 바로 읽기 어렵습니다.
그래서 Filebeat에게 봉투 전체가 아니라 안에 적힌 로그 한 줄만 보내라고 설정한 것입니다.
```

## 6. docker-compose.yml에 추가한 서비스

```yaml
filebeat:
  image: docker.elastic.co/beats/filebeat:8.15.3
  container_name: etl-filebeat
  user: root
  command: ["filebeat", "-e", "--strict.perms=false"]
  volumes:
    - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
    - ./sample-logs:/var/log/security-samples:ro
    - filebeat-data:/usr/share/filebeat/data
```

주요 설정 의미:

| 설정 | 의미 |
|---|---|
| `filebeat/filebeat.yml` | Filebeat 설정 파일을 컨테이너에 연결 |
| `sample-logs` | Filebeat가 읽을 로그 파일 폴더 |
| `filebeat-data` | Filebeat가 어디까지 읽었는지 기록하는 registry 저장소 |
| `--strict.perms=false` | 로컬 학습용으로 설정 파일 권한 검사를 완화 |

## 7. 실행 방법

Docker Compose 실행:

```bash
docker compose up -d
```

Filebeat 로그 확인:

```bash
docker logs etl-filebeat --tail 50
```

Kafka topic 확인:

```bash
docker exec etl-kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

Elasticsearch index 확인:

```bash
curl.exe "http://localhost:9200/_cat/indices/security-auth-log-*?v"
```

Kibana 확인:

```text
http://localhost:5601
```

## 8. 주의할 점

Filebeat는 registry를 사용해 파일을 어디까지 읽었는지 기억합니다.

따라서 같은 로그를 반복해서 다시 넣고 싶다면 다음 중 하나를 선택해야 합니다.

```text
1. auth.log에 새 로그 라인을 추가한다.
2. filebeat-data volume을 삭제하고 다시 실행한다.
```

학습용으로 처음부터 다시 읽게 만들 때:

```bash
docker compose down
docker volume rm etl_streaming_filebeat-data
docker compose up -d
```

단, volume 이름은 Docker Compose 프로젝트명에 따라 달라질 수 있으므로 `docker volume ls`로 확인합니다.

## 9. 이번 단계로 바뀐 점

이전에는 사람이 직접 명령어로 로그를 Kafka에 넣었습니다.

```bash
cat sample-logs/auth.log | docker exec -i etl-kafka ...
```

이제는 Filebeat가 로그 파일을 감시하고 Kafka로 전송합니다.

```text
Filebeat가 로그 수집의 Extract 역할을 맡게 되었습니다.
```

## 10. 면접 설명 예시

```text
초기에는 Kafka console producer를 사용해 샘플 로그를 수동으로 전송했습니다.
이후 Filebeat를 추가해 auth.log 파일을 직접 읽고 Kafka topic으로 전달하도록 확장했습니다.
이 과정에서 Filebeat는 수집 agent, Kafka는 메시지 브로커, Logstash는 파싱 처리기 역할을 하도록 구성했습니다.
```
