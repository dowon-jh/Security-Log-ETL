# Docker Compose 실행 환경

## 1. 이 단계의 목표

이번 단계의 목표는 ETL 파이프라인에 필요한 기본 도구를 Docker Compose로 실행하는 것입니다.

실행되는 서비스는 다음과 같습니다.

| 서비스 | 역할 | 접속 정보 |
|---|---|---|
| Kafka | 로그 메시지를 topic에 저장하고 전달 | localhost:29092 |
| Elasticsearch | 정제된 로그를 저장하고 검색 | http://localhost:9200 |
| Kibana | Elasticsearch 데이터를 시각화 | http://localhost:5601 |
| Logstash | Kafka에서 로그를 읽어 Elasticsearch로 전달 | 내부 연결 |

## 2. 실행 명령

프로젝트 루트에서 실행합니다.

```powershell
docker compose up -d
```

처음 실행할 때는 Docker 이미지 다운로드 때문에 시간이 걸릴 수 있습니다.

## 3. 종료 명령

컨테이너를 중지하려면 다음 명령을 실행합니다.

```powershell
docker compose down
```

Elasticsearch 데이터까지 완전히 삭제하려면 다음 명령을 사용합니다.

```powershell
docker compose down -v
```

`-v` 옵션은 Docker volume을 삭제하므로, 저장된 Elasticsearch 데이터도 사라집니다.

## 4. 서비스 상태 확인

```powershell
docker compose ps
```

정상 상태 예시:

```text
etl-kafka           Up   healthy
etl-elasticsearch   Up   healthy
etl-kibana          Up
etl-logstash        Up
```

## 5. Kafka Topic 확인

```powershell
docker exec etl-kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

확인되어야 하는 topic:

```text
security-auth-log
nginx-access-log
docker-container-log
```

## 6. Elasticsearch 확인

```powershell
curl.exe http://localhost:9200
```

cluster health 확인:

```powershell
curl.exe http://localhost:9200/_cluster/health
```

`status`가 `green`이면 정상입니다.

## 7. Kibana 접속

브라우저에서 다음 주소로 접속합니다.

```text
http://localhost:5601
```

## 8. 현재 Logstash Pipeline

현재 Logstash는 Kafka의 3개 topic을 읽습니다.

```text
security-auth-log
nginx-access-log
docker-container-log
```

그리고 Elasticsearch에 다음 index 형식으로 저장합니다.

```text
etl-raw-log-YYYY.MM.DD
```

아직 로그 파싱은 하지 않습니다.

현재 단계에서는 먼저 아래 흐름이 되는지만 확인합니다.

```text
Kafka topic -> Logstash -> Elasticsearch
```

## 9. 샘플 로그 전송 테스트

다음 명령으로 auth 로그 한 줄을 Kafka에 넣을 수 있습니다.

```powershell
docker exec etl-kafka /bin/bash -c "echo 'Jun 17 21:10:33 ubuntu sshd[1234]: Failed password for invalid user admin from 192.168.0.10 port 54321 ssh2' | /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic security-auth-log"
```

Elasticsearch에서 들어간 로그를 확인합니다.

```powershell
curl.exe "http://localhost:9200/etl-raw-log-*/_search?pretty"
```

검색 결과에 `raw_message`가 보이면 성공입니다.

## 10. 다음 단계

다음 단계는 Logstash에서 로그를 파싱하는 것입니다.

현재는 로그 전체가 `raw_message`에 그대로 들어갑니다.

다음에는 auth 로그에서 아래 필드를 분리합니다.

```text
timestamp
source_ip
username
event_type
status
service
raw_message
```
