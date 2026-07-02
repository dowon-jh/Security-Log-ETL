# 2026-07-03 Nginx access log 수집 확장 작업 정리

## 1. 오늘 작업의 목표

오늘은 Filebeat 실제 수집 구조를 `auth.log`에서 `nginx-access.log`까지 확장했습니다.

목표 흐름:

```text
sample-logs/nginx-access.log
-> Filebeat
-> Kafka nginx-access-log topic
-> Logstash
-> Elasticsearch nginx-access-log-* index
-> Kibana
```

## 2. 변경한 파일

```text
filebeat/filebeat.yml
logstash/pipeline/logstash.conf
docs/nginx-access-log-parsing.md
docs/daily-work-2026-07-03.md
```

## 3. Filebeat 변경 내용

기존에는 Filebeat가 `auth.log`만 읽었습니다.

이번에는 `nginx-access.log` input을 추가했습니다.

```yaml
- type: filestream
  id: nginx-access-log-sample
  paths:
    - /var/log/security-samples/nginx-access.log
  fields:
    log_source: nginx-access-log
    kafka_topic: nginx-access-log
  fields_under_root: true
```

또한 Kafka topic을 고정값이 아니라 이벤트 필드 기반으로 보내도록 변경했습니다.

```yaml
topic: "%{[kafka_topic]}"
```

이 설정 덕분에 하나의 Filebeat가 여러 로그 파일을 읽고, 로그 종류별로 다른 Kafka topic에 보낼 수 있습니다.

## 4. Logstash 변경 내용

`nginx-access-log` topic 전용 파싱 분기를 추가했습니다.

정규화한 주요 필드:

```text
source_ip
nginx_timestamp
http_method
url_path
http_version
http_status
bytes_sent
referrer
user_agent
event_type
service
status
log_level
raw_message
```

HTTP 상태 코드는 다음 기준으로 간단히 분류했습니다.

```text
400 이상 = failed
400 미만 = success
```

## 5. Elasticsearch 저장 방식

Nginx access log는 auth log와 별도 index로 저장됩니다.

```text
nginx-access-log-YYYY.MM.DD
```

이렇게 분리하면 로그 종류별 보관, 검색, 대시보드 구성이 쉬워집니다.

## 6. 확인 명령어

Docker Compose 설정 검증:

```bash
docker compose config --quiet
```

컨테이너 재시작:

```bash
docker compose up -d
```

Filebeat 로그 확인:

```bash
docker logs etl-filebeat --tail 50
```

Logstash 로그 확인:

```bash
docker logs etl-logstash --tail 80
```

Nginx index 확인:

```bash
curl "http://localhost:9200/_cat/indices/nginx-access-log-*?v"
```

Nginx 문서 확인:

```bash
curl "http://localhost:9200/nginx-access-log-*/_search?pretty&size=5"
```

## 7. Kibana 확인

새 Data View:

```text
Name: nginx_access_logs
Index pattern: nginx-access-log-*
Timestamp field: @timestamp
```

KQL 예시:

```kql
event_type: nginx_access
```

```kql
http_status >= 400
```

```kql
url_path: "/admin"
```

## 8. 오늘 작업의 의미

이번 작업으로 프로젝트는 서버 인증 로그뿐 아니라 웹 접근 로그까지 처리할 수 있게 되었습니다.

즉, 보안 분석 관점이 다음처럼 확장되었습니다.

```text
auth.log = 서버 로그인과 권한 상승 관점
nginx-access.log = 웹 요청과 비정상 접근 관점
```

면접에서는 다음처럼 설명할 수 있습니다.

```text
Filebeat input을 확장하여 auth.log와 nginx-access.log를 동시에 수집하도록 구성했습니다.
각 input에 kafka_topic 필드를 지정했고, Kafka output에서 해당 필드를 사용해 로그 종류별 topic으로 분리 전송했습니다.
Logstash에서는 Kafka topic 기준으로 파싱 로직을 분리하여 Nginx access log를 source_ip, http_method, url_path, http_status, user_agent 등으로 정규화했습니다.
```
