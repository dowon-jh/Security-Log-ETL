# Nginx access log 수집 및 파싱

## 1. 이번 단계의 목표

이번 단계는 Filebeat가 `nginx-access.log` 파일도 읽어서 Kafka로 보내고, Logstash가 Nginx access log를 필드 단위로 정제하도록 확장하는 것입니다.

전체 흐름:

```text
sample-logs/nginx-access.log
-> Filebeat
-> Kafka nginx-access-log topic
-> Logstash parsing
-> Elasticsearch nginx-access-log-* index
-> Kibana Discover
```

## 2. Nginx access log란?

Nginx access log는 웹 서버에 들어온 요청 기록입니다.

예시:

```text
203.0.113.10 - - [17/Jun/2026:21:10:35 +0900] "POST /login HTTP/1.1" 401 342 "https://example.com/login" "curl/8.0"
```

이 한 줄에는 다음 정보가 들어 있습니다.

| 값 | 의미 |
|---|---|
| `203.0.113.10` | 요청을 보낸 클라이언트 IP |
| `17/Jun/2026:21:10:35 +0900` | 요청 발생 시각 |
| `POST` | HTTP 메서드 |
| `/login` | 요청 경로 |
| `401` | HTTP 응답 상태 코드 |
| `342` | 응답 바이트 크기 |
| `https://example.com/login` | 이전 페이지 또는 referrer |
| `curl/8.0` | User-Agent |

## 3. Filebeat 설정

`filebeat/filebeat.yml`에 Nginx 로그 input을 추가했습니다.

```yaml
- type: filestream
  id: nginx-access-log-sample
  enabled: true
  paths:
    - /var/log/security-samples/nginx-access.log
  fields:
    log_source: nginx-access-log
    kafka_topic: nginx-access-log
  fields_under_root: true
```

의미:

```text
Filebeat가 nginx-access.log 파일을 읽고,
해당 로그를 nginx-access-log Kafka topic으로 보냅니다.
```

Kafka output은 고정 topic이 아니라 이벤트에 들어 있는 `kafka_topic` 값을 사용하도록 변경했습니다.

```yaml
output.kafka:
  topic: "%{[kafka_topic]}"
```

이렇게 하면 같은 Filebeat 컨테이너가 여러 로그 파일을 읽더라도 로그 종류별로 다른 Kafka topic에 보낼 수 있습니다.

## 4. Logstash 파싱 방식

`logstash/pipeline/logstash.conf`에 `nginx-access-log` topic 전용 분기를 추가했습니다.

주요 파싱 필드:

| 필드 | 의미 |
|---|---|
| `source_ip` | 요청을 보낸 IP |
| `nginx_timestamp` | Nginx 원본 로그 시간 |
| `http_method` | GET, POST 같은 HTTP 메서드 |
| `url_path` | 요청 URI |
| `http_version` | HTTP 버전 |
| `http_status` | 응답 상태 코드 |
| `bytes_sent` | 응답 크기 |
| `referrer` | 이전 페이지 |
| `user_agent` | 클라이언트 도구 또는 브라우저 |
| `event_type` | `nginx_access` |
| `service` | `nginx` |

## 5. 상태 코드 해석

이번 프로젝트에서는 HTTP status code를 간단히 다음처럼 분류했습니다.

```text
400 이상 = failed
400 미만 = success
```

예시:

| HTTP status | 의미 | status |
|---|---|---|
| `200` | 정상 요청 | `success` |
| `401` | 인증 실패 | `failed` |
| `403` | 접근 금지 | `failed` |
| `500` | 서버 오류 | `failed` |

## 6. Elasticsearch index

정제된 Nginx 로그는 다음 index로 저장됩니다.

```text
nginx-access-log-YYYY.MM.DD
```

확인 명령어:

```bash
curl "http://localhost:9200/_cat/indices/nginx-access-log-*?v"
```

샘플 문서 확인:

```bash
curl "http://localhost:9200/nginx-access-log-*/_search?pretty&size=5"
```

## 7. Kibana에서 확인

Kibana에서 새 Data View를 만들 수 있습니다.

```text
Name: nginx_access_logs
Index pattern: nginx-access-log-*
Timestamp field: @timestamp
```

Discover에서 사용할 수 있는 KQL 예시:

```kql
event_type: nginx_access
```

```kql
http_status >= 400
```

```kql
source_ip: "203.0.113.10"
```

## 8. 보안 분석 의미

Nginx access log는 웹 서비스 관점의 공격 흔적을 찾는 데 중요합니다.

예를 들어 다음 이벤트를 탐지할 수 있습니다.

```text
특정 IP에서 /login 요청이 반복됨
401, 403 상태 코드가 많이 발생함
/admin 같은 민감 경로 접근 시도가 있음
curl, python-requests 같은 자동화 도구 User-Agent가 보임
```

즉, auth.log가 서버 로그인 관점의 보안 로그라면, nginx-access.log는 웹 접근 관점의 보안 로그입니다.

## 9. 면접 답변 예시

```text
auth.log 수집 구조를 Filebeat 기반으로 확장한 뒤, 같은 방식으로 nginx-access.log도 수집하도록 구성했습니다.
Filebeat input별로 kafka_topic 필드를 지정해 로그 종류에 따라 다른 Kafka topic으로 전달했고,
Logstash에서는 Kafka topic을 기준으로 auth log와 nginx access log 파싱 로직을 분리했습니다.
Nginx 로그는 source_ip, http_method, url_path, http_status, user_agent 등으로 정규화하여 웹 접근 이상 행위를 분석할 수 있게 했습니다.
```
