# 다음 작업 전 프로젝트 환경 준비 체크리스트

## 1. 이 문서의 목적

PC를 다시 켠 뒤 Security-Log-ETL 프로젝트를 이어서 진행하기 위해 필요한 실행 순서를 정리한 문서입니다.

다음 단계인 `auth.log` Logstash 파싱 구현을 시작하기 전에 이 문서를 보고 환경을 먼저 준비합니다.

## 2. Docker Desktop 실행

Windows에서 Docker Desktop을 먼저 실행합니다.

Docker Desktop이 완전히 켜질 때까지 기다립니다.

PowerShell에서 Docker가 준비됐는지 확인합니다.

```powershell
docker version
```

또는:

```powershell
docker compose version
```

정상 출력이 나오면 Docker가 준비된 상태입니다.

## 3. 프로젝트 폴더로 이동

PowerShell에서 프로젝트 폴더로 이동합니다.

```powershell
cd C:\Users\dwjam\OneDrive\문서\ETL_streaming
```

## 4. Docker Compose 환경 실행

전체 ETL 환경을 실행합니다.

```powershell
docker compose up -d
```

이 명령으로 아래 서비스가 실행됩니다.

```text
Kafka
Kafka-init
Elasticsearch
Kibana
Logstash
```

## 5. 컨테이너 상태 확인

```powershell
docker compose ps
```

정상적으로 보여야 하는 컨테이너:

```text
etl-kafka
etl-elasticsearch
etl-kibana
etl-logstash
```

Kafka와 Elasticsearch는 `healthy` 상태면 좋습니다.

## 6. Kafka Topic 확인

```powershell
docker exec etl-kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

아래 topic이 보여야 합니다.

```text
security-auth-log
nginx-access-log
docker-container-log
```

`__consumer_offsets`가 함께 보여도 정상입니다.

## 7. Elasticsearch 확인

Elasticsearch cluster health를 확인합니다.

```powershell
curl.exe http://localhost:9200/_cluster/health
```

정상 예시:

```json
{
  "status": "green"
}
```

`green`이면 정상입니다.

## 8. Kibana 접속

브라우저에서 Kibana에 접속합니다.

```text
http://localhost:5601
```

Discover로 바로 이동하려면 다음 주소를 사용합니다.

```text
http://localhost:5601/app/discover
```

## 9. Logstash 상태 확인

Logstash가 Kafka topic을 읽을 준비가 되었는지 확인합니다.

```powershell
docker logs --tail 80 etl-logstash
```

정상 로그 예시:

```text
Successfully joined group
Adding newly assigned partitions
```

이 문구가 보이면 Logstash가 Kafka consumer group에 정상 합류한 것입니다.

## 10. 최소 실행 명령 요약

다음날 빠르게 시작할 때는 아래 순서만 기억해도 됩니다.

```powershell
cd C:\Users\dwjam\OneDrive\문서\ETL_streaming
docker compose up -d
docker compose ps
curl.exe http://localhost:9200/_cluster/health
```

그리고 브라우저에서 접속합니다.

```text
http://localhost:5601
```

## 11. 다음 작업

환경이 정상적으로 올라오면 다음 단계로 넘어갑니다.

다음 단계:

```text
auth.log Logstash 파싱 구현
```

목표:

```text
raw_message
-> source_ip
-> username
-> event_type
-> status
-> service
```

즉, 단순히 원본 로그를 저장하는 단계에서 로그를 분석 가능한 구조화 데이터로 바꾸는 단계입니다.
