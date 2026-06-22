# Elasticsearch Index Mapping 단계

## 1. 이 단계의 목표

이번 단계의 목표는 `security-auth-log-*` index에 저장될 필드 타입을 명시적으로 정의하는 것입니다.

이전 단계에서는 Elasticsearch가 필드 타입을 자동 추론했습니다.

자동 추론 결과 `source_ip`가 `text`로 생성되었습니다.

하지만 IP 주소는 문자열 검색보다 IP 타입으로 저장하는 것이 좋습니다.

예:

```text
source_ip: text
```

보다:

```text
source_ip: ip
```

가 검색, 필터링, 집계에 더 적합합니다.

## 2. 내가 하는 행동과 영향

### 2.1 Index Template 파일 생성

생성 파일:

```text
elasticsearch/mappings/security-auth-log-template.json
```

이 파일은 Elasticsearch에 전달할 index template 정의입니다.

역할:

```text
security-auth-log-* 이름으로 새 index가 만들어질 때
어떤 필드를 어떤 타입으로 만들지 미리 지정한다.
```

### 2.2 Index Template 적용

실행 명령:

```powershell
curl.exe -s -X PUT "http://localhost:9200/_index_template/security-auth-log-template" -H "Content-Type: application/json" --data-binary "@elasticsearch/mappings/security-auth-log-template.json"
```

이 명령의 역할:

1. Elasticsearch에 `security-auth-log-template`이라는 index template을 등록합니다.
2. 앞으로 생성되는 `security-auth-log-*` index에 이 mapping을 적용합니다.
3. 기존 index의 필드 타입은 바꾸지 않습니다.

중요:

```text
이미 생성된 index의 필드 타입은 쉽게 변경할 수 없습니다.
```

예를 들어 기존 `security-auth-log-2026.06.17`에서 `source_ip`가 이미 `text`로 만들어졌다면, 같은 index 안에서 `ip` 타입으로 직접 변경할 수 없습니다.

실무에서는 이런 경우 새 index를 만들고 reindex 작업을 수행합니다.

이번 학습 단계에서는 앞으로 생성될 index부터 올바른 mapping이 적용되도록 template을 등록합니다.

## 3. 주요 Mapping 설계

| 필드 | 타입 | 이유 |
|---|---|---|
| `@timestamp` | `date` | 시간 기반 검색과 Kibana 시간 필터 |
| `source_ip` | `ip` | IP 필터링, 집계, 범위 검색 |
| `source_port` | `integer` | 포트 번호 숫자 검색 |
| `username` | `keyword` | 계정별 집계 |
| `event_type` | `keyword` | 이벤트 유형별 집계 |
| `status` | `keyword` | 성공/실패 집계 |
| `service` | `keyword` | 서비스별 필터 |
| `log_level` | `keyword` | 로그 레벨별 필터 |
| `raw_message` | `text` | 원문 검색 |
| `message` | `text` | 원문 검색 |
| `sudo_command` | `keyword` | 명령어별 집계 |

## 4. 적용 확인 명령어

### 4.1 등록된 template 확인

```powershell
curl.exe -s "http://localhost:9200/_index_template/security-auth-log-template?pretty"
```

정상이라면 `index_patterns`에 다음 값이 보입니다.

```text
security-auth-log-*
```

### 4.2 새 index 생성 후 mapping 확인

새 날짜의 auth 로그를 넣으면 새 index가 생성됩니다.

예:

```powershell
docker exec etl-kafka /bin/bash -c "echo 'Jun 22 21:10:33 ubuntu sshd[2001]: Failed password for invalid user test from 203.0.113.55 port 55123 ssh2' | /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic security-auth-log"
```

이후 mapping 확인:

```powershell
curl.exe -s "http://localhost:9200/security-auth-log-2026.06.22/_mapping?pretty"
```

확인할 부분:

```json
"source_ip": {
  "type": "ip"
}
```

## 5. 왜 number_of_replicas를 0으로 두었나?

현재 로컬 Elasticsearch는 단일 노드입니다.

기본 replica가 1이면 primary shard는 배치되지만 replica shard를 배치할 두 번째 노드가 없습니다.

그래서 index health가 `yellow`가 됩니다.

로컬 학습 환경에서는 다음처럼 설정했습니다.

```json
"number_of_replicas": 0
```

영향:

1. 단일 노드에서도 index health가 `green`이 될 수 있습니다.
2. 대신 복제본이 없으므로 노드 장애 시 데이터 보호는 약합니다.

실무에서는 보통 여러 Elasticsearch 노드를 사용하고 replica를 1 이상으로 둡니다.

## 6. 실무 관점

실무에서는 mapping을 미리 설계하는 것이 중요합니다.

이유:

1. 잘못 추론된 타입은 나중에 바꾸기 어렵습니다.
2. `text`와 `keyword` 용도가 다릅니다.
3. IP, date, number 타입은 정확히 지정해야 검색과 집계가 안정적입니다.
4. Kibana Dashboard의 집계 기준이 mapping에 영향을 받습니다.

예를 들어 `event_type`이 `text`로만 저장되면 집계가 불편합니다.

반대로 `keyword`로 저장하면 다음 집계가 쉬워집니다.

```text
ssh_failed_login 몇 건
ssh_success_login 몇 건
root_login 몇 건
sudo_command 몇 건
```

## 7. 다음 단계

다음 단계는 Kibana에서 `security-auth-log-*` Data View를 만들고, 필드가 올바르게 보이는지 확인하는 것입니다.

그 다음에는 탐지 시나리오를 구현합니다.

예:

```text
5분 내 SSH 로그인 실패 20회 이상
root 계정 로그인 발생
sudo 명령어 사용 이벤트
```
