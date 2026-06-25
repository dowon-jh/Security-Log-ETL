# Kibana 필터링 연습과 탐지 시나리오 쿼리

## 1. 이 단계의 목표

이번 단계의 목표는 `security-auth-log-*` index에 저장된 구조화 로그를 Kibana Discover와 Elasticsearch query로 검색하는 것입니다.

이전 단계까지는 다음을 완료했습니다.

```text
auth.log 원본
-> Logstash parsing
-> security-auth-log-* index 저장
-> Elasticsearch mapping 적용
```

이번 단계에서는 저장된 필드를 이용해 보안 탐지 조건을 검색합니다.

## 2. 내가 하는 행동과 영향

### 2.1 Kibana에서 필터링

Kibana Discover에서 필터를 입력하면 Elasticsearch에 저장된 로그 중 조건에 맞는 문서만 화면에 표시됩니다.

예:

```text
event_type: ssh_failed_login
```

영향:

```text
데이터를 수정하거나 삭제하지 않는다.
조건에 맞는 로그만 조회한다.
```

### 2.2 Elasticsearch query 실행

PowerShell에서 `curl.exe`로 query를 실행하면 Kibana가 내부적으로 하는 검색을 API로 직접 확인할 수 있습니다.

예:

```powershell
curl.exe -s -X GET "http://localhost:9200/security-auth-log-*/_search?pretty" -H "Content-Type: application/json" --data-binary "@elasticsearch/queries/ssh-failed-login.json"
```

영향:

```text
Elasticsearch에 저장된 로그를 조회한다.
데이터를 수정하지 않는다.
탐지 조건이 실제로 동작하는지 확인할 수 있다.
```

### 2.3 테스트 로그 추가

Brute Force 탐지처럼 특정 조건을 만족하는 데이터가 부족할 때는 테스트 로그를 Kafka에 넣습니다.

영향:

```text
Kafka topic에 테스트 로그가 추가된다.
Logstash가 로그를 consume하고 파싱한다.
Elasticsearch에 새 문서가 저장된다.
```

## 3. Kibana Discover 준비

Kibana 접속:

```text
http://localhost:5601
```

Discover 이동:

```text
Analytics -> Discover
```

또는:

```text
http://localhost:5601/app/discover
```

Data View:

```text
security-auth-log-*
```

Time field:

```text
@timestamp
```

시간 범위:

```text
Last 30 days
```

## 4. Kibana에서 연습할 KQL 필터

### 4.1 SSH 로그인 실패

```text
event_type: ssh_failed_login
```

의미:

```text
SSH 로그인 실패 이벤트만 조회한다.
```

### 4.2 root 로그인

```text
event_type: root_login
```

의미:

```text
root 계정으로 로그인에 성공한 이벤트만 조회한다.
```

### 4.3 sudo 명령어 사용

```text
event_type: sudo_command
```

의미:

```text
sudo 명령어 사용 이벤트만 조회한다.
```

### 4.4 실패 상태만 조회

```text
status: failed
```

의미:

```text
실패한 보안 이벤트만 조회한다.
```

### 4.5 root 계정 관련 이벤트

```text
username: root
```

의미:

```text
root 계정과 관련된 이벤트를 조회한다.
```

### 4.6 특정 IP의 실패 로그인

```text
event_type: ssh_failed_login and source_ip: "203.0.113.10"
```

의미:

```text
특정 IP에서 발생한 SSH 로그인 실패만 조회한다.
```

## 5. Elasticsearch query 파일

이번 단계에서 만든 query 파일:

```text
elasticsearch/queries/ssh-failed-login.json
elasticsearch/queries/root-login.json
elasticsearch/queries/sudo-command.json
elasticsearch/queries/failed-login-by-ip.json
elasticsearch/queries/brute-force-5m.json
```

테스트 데이터 파일:

```text
sample-logs/brute-force-auth.log
```

이 파일에는 같은 IP에서 5분 안에 SSH 로그인 실패가 20회 발생하는 샘플 로그가 들어 있습니다.

## 6. Query 실행 명령어

### 6.1 SSH 로그인 실패 조회

```powershell
curl.exe -s -X GET "http://localhost:9200/security-auth-log-*/_search?pretty" -H "Content-Type: application/json" --data-binary "@elasticsearch/queries/ssh-failed-login.json"
```

역할:

```text
event_type이 ssh_failed_login인 문서를 조회한다.
```

### 6.2 root 로그인 조회

```powershell
curl.exe -s -X GET "http://localhost:9200/security-auth-log-*/_search?pretty" -H "Content-Type: application/json" --data-binary "@elasticsearch/queries/root-login.json"
```

역할:

```text
event_type이 root_login인 문서를 조회한다.
```

### 6.3 sudo 명령어 사용 조회

```powershell
curl.exe -s -X GET "http://localhost:9200/security-auth-log-*/_search?pretty" -H "Content-Type: application/json" --data-binary "@elasticsearch/queries/sudo-command.json"
```

역할:

```text
event_type이 sudo_command인 문서를 조회한다.
```

### 6.4 로그인 실패 TOP IP 조회

```powershell
curl.exe -s -X GET "http://localhost:9200/security-auth-log-*/_search?pretty" -H "Content-Type: application/json" --data-binary "@elasticsearch/queries/failed-login-by-ip.json"
```

역할:

```text
SSH 로그인 실패가 많이 발생한 source_ip를 집계한다.
```

주의:

```text
security-auth-log-2026.06.17은 mapping template 적용 전에 생성되어 source_ip가 text 타입입니다.
security-auth-log-2026.06.22 이후 index는 source_ip가 ip 타입입니다.
```

서로 다른 mapping을 가진 index를 함께 집계하면 오류가 날 수 있습니다.

그래서 이 query는 `runtime_mappings`로 `source_ip_group`이라는 임시 집계 필드를 만들어 mixed mapping 상황에서도 집계되도록 했습니다.

### 6.5 5분 내 SSH 실패 20회 이상 탐지

```powershell
curl.exe -s -X GET "http://localhost:9200/security-auth-log-*/_search?pretty" -H "Content-Type: application/json" --data-binary "@elasticsearch/queries/brute-force-5m.json"
```

역할:

```text
5분 단위 시간 구간마다 source_ip별 SSH 실패 횟수를 집계하고,
20회 이상 발생한 IP만 탐지한다.
```

테스트 데이터 전송:

```powershell
Get-Content sample-logs\brute-force-auth.log | docker exec -i etl-kafka /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic security-auth-log
```

이 명령의 영향:

```text
sample-logs/brute-force-auth.log의 20줄이 Kafka security-auth-log topic에 들어간다.
Logstash가 각 로그를 파싱한다.
Elasticsearch에 20개의 ssh_failed_login 이벤트가 저장된다.
```

성공 결과 예시:

```json
{
  "key_as_string": "2026-06-25T12:10:00.000Z",
  "doc_count": 20,
  "by_source_ip": {
    "buckets": [
      {
        "key": "203.0.113.200",
        "doc_count": 20
      }
    ]
  }
}
```

## 7. 탐지 시나리오 정리

| 시나리오 | 조건 | 사용하는 필드 |
|---|---|---|
| SSH 로그인 실패 | `event_type = ssh_failed_login` | `event_type`, `status` |
| root 로그인 | `event_type = root_login` | `event_type`, `username` |
| sudo 명령어 사용 | `event_type = sudo_command` | `event_type`, `sudo_command` |
| TOP 공격 IP | SSH 실패를 `source_ip`별 집계 | `source_ip`, `event_type` |
| Brute Force 의심 | 5분 내 같은 IP에서 SSH 실패 20회 이상 | `@timestamp`, `source_ip`, `event_type` |

## 8. 다음 단계

다음 단계는 Kibana Lens 또는 Dashboard에서 다음 시각화를 만드는 것입니다.

```text
시간대별 로그인 실패 추이
TOP 공격 IP
계정별 로그인 실패 횟수
root 로그인 발생 현황
sudo 명령어 사용 현황
위험 이벤트 목록
```
