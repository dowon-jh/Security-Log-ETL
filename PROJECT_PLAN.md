# 보안 로그 ETL 파이프라인 프로젝트 기획안

## 1. 프로젝트 한 줄 소개

Linux 서버, Nginx, Docker 컨테이너 로그를 수집해서 Kafka, Logstash, Elasticsearch, Kibana로 이어지는 ETL 파이프라인을 구축하고, 보안 이상 징후를 탐지하는 실전형 로그 분석 프로젝트입니다.

이 프로젝트의 핵심은 단순히 도구를 실행하는 것이 아니라, 로그 데이터가 어떤 흐름으로 이동하고, 어떻게 정제되며, 어떤 기준으로 위험 이벤트를 판단하는지 직접 이해하는 것입니다.

## 2. 프로젝트 목표

이 프로젝트를 완료하면 다음을 설명하고 시연할 수 있어야 합니다.

1. Filebeat가 Linux 로그 파일을 읽어 Kafka로 전송하는 방식
2. Kafka topic을 로그 종류별로 설계하는 방식
3. Logstash가 Kafka에서 로그를 받아 필드를 정규화하는 방식
4. Elasticsearch index에 로그를 저장하고 검색하는 방식
5. Kibana에서 보안 이벤트를 시각화하는 방식
6. 로그인 실패, root 로그인, sudo 사용, 과도한 요청 같은 보안 탐지 시나리오를 구현하는 방식

## 3. 초보자를 위한 ETL 개념 정리

ETL은 Extract, Transform, Load의 약자입니다.

Extract는 데이터를 가져오는 단계입니다. 이 프로젝트에서는 Linux 서버 로그, Nginx access log, Docker container log를 Filebeat가 읽습니다.

Transform은 데이터를 정리하고 표준화하는 단계입니다. 이 프로젝트에서는 Logstash가 로그 문자열에서 IP, 사용자명, 상태, 서비스명 같은 필드를 뽑아냅니다.

Load는 정리된 데이터를 저장소에 넣는 단계입니다. 이 프로젝트에서는 Elasticsearch에 index 형태로 저장합니다.

전체 흐름은 다음과 같습니다.

```text
Linux 로그 파일
  -> Filebeat
  -> Kafka Topic
  -> Logstash Pipeline
  -> Elasticsearch Index
  -> Kibana Dashboard
```

## 4. 전체 아키텍처

```text
+-------------------+        +-------------------+        +-------------------+
| Linux Log Files   |        | Filebeat          |        | Kafka             |
| auth.log          | -----> | log collector     | -----> | security-auth-log |
| nginx access log  |        |                   |        | nginx-access-log  |
| docker logs       |        |                   |        | docker-container  |
+-------------------+        +-------------------+        +-------------------+
                                                               |
                                                               v
                                                    +-------------------+
                                                    | Logstash          |
                                                    | parsing/filtering |
                                                    +-------------------+
                                                               |
                                                               v
                                                    +-------------------+
                                                    | Elasticsearch     |
                                                    | indexed logs      |
                                                    +-------------------+
                                                               |
                                                               v
                                                    +-------------------+
                                                    | Kibana            |
                                                    | dashboard/rules   |
                                                    +-------------------+
```

## 5. 기술 스택

| 구분 | 기술 | 역할 |
|---|---|---|
| 로그 수집 | Filebeat | 서버 로그 파일을 읽어 Kafka로 전송 |
| 메시지 브로커 | Kafka | 로그를 topic 단위로 임시 저장하고 전달 |
| 로그 정제 | Logstash | 로그를 파싱하고 공통 필드로 변환 |
| 저장소 | Elasticsearch | 정제된 로그 검색 및 분석 |
| 시각화 | Kibana | 대시보드, 검색, 탐지 결과 확인 |
| 실행 환경 | Docker Compose | 전체 서비스를 로컬에서 쉽게 실행 |
| 운영 환경 연습 | Linux | 보안 로그와 서버 로그 이해 |

## 6. 수집 대상 로그

### 6.1 Linux 인증 로그

예시 파일:

```text
/var/log/auth.log
```

주요 이벤트:

1. SSH 로그인 성공
2. SSH 로그인 실패
3. root 계정 로그인
4. sudo 명령어 사용

Kafka topic:

```text
security-auth-log
```

Elasticsearch index:

```text
security-auth-log-YYYY.MM.DD
```

### 6.2 Nginx Access Log

예시 파일:

```text
/var/log/nginx/access.log
```

주요 이벤트:

1. 특정 IP의 요청량
2. HTTP status code
3. 요청 URL
4. User-Agent

Kafka topic:

```text
nginx-access-log
```

Elasticsearch index:

```text
nginx-access-log-YYYY.MM.DD
```

### 6.3 Docker Container Log

예시 경로:

```text
/var/lib/docker/containers/*/*.log
```

주요 이벤트:

1. 컨테이너 애플리케이션 로그
2. error, warn, info 로그 레벨
3. 서비스별 장애 로그

Kafka topic:

```text
docker-container-log
```

Elasticsearch index:

```text
docker-container-log-YYYY.MM.DD
```

## 7. 표준 필드 설계

서로 다른 로그를 분석하기 쉽게 만들기 위해 공통 필드를 정의합니다.

| 필드명 | 의미 | 예시 |
|---|---|---|
| timestamp | 이벤트 발생 시간 | 2026-06-17T21:00:00+09:00 |
| source_ip | 접속을 시도한 IP | 192.168.0.10 |
| username | 로그인 또는 명령 수행 계정 | ubuntu, root |
| event_type | 이벤트 종류 | ssh_failed_login, sudo_command |
| status | 성공/실패 상태 | success, failed |
| service | 로그 발생 서비스 | sshd, nginx, docker |
| log_level | 로그 레벨 | info, warn, error |
| raw_message | 원본 로그 메시지 | 원본 로그 전체 |

추가로 Nginx 로그에는 다음 필드를 확장합니다.

| 필드명 | 의미 |
|---|---|
| http_method | GET, POST 등 |
| request_path | 요청 경로 |
| response_code | HTTP 응답 코드 |
| user_agent | 클라이언트 정보 |
| bytes_sent | 전송 바이트 |

## 8. Kafka Topic 설계

처음에는 로그 종류별로 topic을 분리합니다.

| Topic | 대상 로그 | 이유 |
|---|---|---|
| security-auth-log | Linux auth log | 보안 탐지와 직접 연결 |
| nginx-access-log | Nginx access log | 웹 접근 분석용 |
| docker-container-log | Docker container log | 애플리케이션 로그 분석용 |

초보자 단계에서는 partition과 replication을 복잡하게 가져가지 않습니다.

권장 시작 설정:

| 항목 | 값 |
|---|---|
| partitions | 1 |
| replication factor | 1 |
| retention | 7일 |

이후 학습 확장 단계에서 partition을 늘리고 consumer group 개념을 추가합니다.

## 9. Elasticsearch Index 설계

로그 종류별로 daily index를 사용합니다.

```text
security-auth-log-2026.06.17
nginx-access-log-2026.06.17
docker-container-log-2026.06.17
```

처음에는 날짜별 index를 사용하면 Kibana에서 시간 기반 검색을 학습하기 쉽습니다.

기본 mapping 방향:

| 필드 | 타입 |
|---|---|
| timestamp | date |
| source_ip | ip |
| username | keyword |
| event_type | keyword |
| status | keyword |
| service | keyword |
| log_level | keyword |
| raw_message | text |
| response_code | integer |
| request_path | keyword |

## 10. 보안 탐지 시나리오

최소 3개 이상이 요구사항이지만, 포트폴리오 완성도를 위해 4개를 구현합니다.

### 10.1 SSH Brute Force 의심

조건:

```text
5분 내 같은 source_ip에서 SSH 로그인 실패 20회 이상
```

탐지 의미:

공격자가 비밀번호를 계속 대입하는 상황을 의심할 수 있습니다.

필요 필드:

```text
timestamp
source_ip
event_type
status
service
```

탐지 결과 예시:

```text
event_type = ssh_failed_login
source_ip = 203.0.113.10
count >= 20 within 5 minutes
```

### 10.2 root 계정 로그인 탐지

조건:

```text
username이 root이고 로그인 성공 이벤트 발생
```

탐지 의미:

root 계정은 권한이 강하기 때문에 직접 로그인이 발생하면 확인이 필요합니다.

필요 필드:

```text
timestamp
source_ip
username
event_type
status
```

### 10.3 특정 IP 과도한 요청 탐지

조건:

```text
1분 내 같은 source_ip에서 Nginx 요청 100회 이상
```

탐지 의미:

스크래핑, 무차별 요청, 단순 DoS 시도 가능성을 확인할 수 있습니다.

필요 필드:

```text
timestamp
source_ip
service
request_path
response_code
```

### 10.4 sudo 명령어 사용 탐지

조건:

```text
sudo 명령어 사용 이벤트 발생
```

탐지 의미:

권한 상승 행위가 발생했음을 확인할 수 있습니다.

필요 필드:

```text
timestamp
username
event_type
raw_message
```

## 11. Kibana Dashboard 구성

대시보드는 “보안 관제 화면”처럼 보이도록 구성합니다.

포함 항목:

1. 시간대별 로그인 실패 추이
2. TOP 공격 IP
3. 계정별 로그인 실패 횟수
4. root 로그인 발생 현황
5. sudo 명령어 사용 현황
6. 위험 이벤트 목록

권장 시각화:

| 항목 | Kibana 시각화 |
|---|---|
| 시간대별 로그인 실패 추이 | Line chart |
| TOP 공격 IP | Horizontal bar chart |
| 계정별 로그인 실패 횟수 | Bar chart |
| root 로그인 발생 현황 | Data table |
| sudo 명령어 사용 현황 | Data table |
| 위험 이벤트 목록 | Discover saved search 또는 Data table |

## 12. 프로젝트 디렉터리 구조

초기 권장 구조:

```text
ETL_streaming/
  docker-compose.yml
  README.md
  PROJECT_PLAN.md
  docs/
    kafka-topic-design.md
    detection-scenarios.md
    troubleshooting.md
  filebeat/
    filebeat.yml
  logstash/
    pipeline/
      auth.conf
      nginx.conf
      docker.conf
    patterns/
      custom-patterns
  elasticsearch/
    mappings/
      security-auth-log.json
      nginx-access-log.json
      docker-container-log.json
  kibana/
    screenshots/
  sample-logs/
    auth.log
    nginx-access.log
    docker-container.log
```

## 13. 단계별 진행 계획

### 1단계: 기초 개념 잡기

목표:

ETL, 로그, Kafka, Elastic Stack이 각각 무엇을 하는지 이해합니다.

할 일:

1. ETL 흐름을 그림으로 이해하기
2. Linux auth log 샘플 읽어보기
3. Nginx access log 샘플 읽어보기
4. Elasticsearch index와 document 개념 이해하기

완료 기준:

로그 한 줄을 보고 “이 로그는 누가, 언제, 어디서, 어떤 행동을 했다는 뜻인지” 설명할 수 있어야 합니다.

### 2단계: Docker Compose 실행 환경 만들기

목표:

Kafka, Logstash, Elasticsearch, Kibana를 로컬에서 실행합니다.

할 일:

1. `docker-compose.yml` 작성
2. Kafka 실행 확인
3. Elasticsearch 실행 확인
4. Kibana 접속 확인
5. Logstash 실행 확인

완료 기준:

브라우저에서 Kibana에 접속할 수 있고, Elasticsearch health check가 정상이어야 합니다.

### 3단계: Kafka Topic 설계 및 생성

목표:

로그 종류별 topic을 생성합니다.

할 일:

1. `security-auth-log` topic 생성
2. `nginx-access-log` topic 생성
3. `docker-container-log` topic 생성
4. topic 설계 문서 작성

완료 기준:

Kafka topic 목록에서 3개 topic이 확인되어야 합니다.

### 4단계: Filebeat 로그 수집 설정

목표:

Filebeat가 로그 파일을 읽고 Kafka로 전송하게 만듭니다.

할 일:

1. `filebeat.yml` 작성
2. auth log input 설정
3. nginx access log input 설정
4. docker log input 설정
5. Kafka output 설정

완료 기준:

Kafka consumer로 topic 메시지를 확인했을 때 로그가 들어와야 합니다.

### 5단계: Logstash Pipeline 작성

목표:

Kafka에서 로그를 가져와 표준 필드로 정제합니다.

할 일:

1. Kafka input 설정
2. auth log grok filter 작성
3. nginx access log grok filter 작성
4. docker log json 파싱 또는 grok 파싱 작성
5. Elasticsearch output 설정

완료 기준:

Elasticsearch에 저장된 document에서 `timestamp`, `source_ip`, `event_type`, `status` 같은 필드가 분리되어 보여야 합니다.

### 6단계: Elasticsearch Mapping 작성

목표:

검색과 집계에 적합한 필드 타입을 지정합니다.

할 일:

1. auth log mapping 작성
2. nginx log mapping 작성
3. docker log mapping 작성
4. index template 적용

완료 기준:

`source_ip`는 ip 타입, `timestamp`는 date 타입, `event_type`은 keyword 타입으로 저장되어야 합니다.

### 7단계: 탐지 시나리오 구현

목표:

정제된 로그에서 보안 이벤트를 탐지합니다.

할 일:

1. SSH Brute Force 탐지 쿼리 작성
2. root 로그인 탐지 쿼리 작성
3. 과도한 요청 탐지 쿼리 작성
4. sudo 사용 탐지 쿼리 작성
5. 탐지 시나리오 문서 작성

완료 기준:

각 시나리오별로 테스트 로그를 넣었을 때 Kibana 또는 Elasticsearch query에서 결과가 확인되어야 합니다.

### 8단계: Kibana Dashboard 구성

목표:

보안 이벤트를 한 화면에서 볼 수 있는 대시보드를 만듭니다.

할 일:

1. Data view 생성
2. 로그인 실패 추이 차트 생성
3. TOP 공격 IP 차트 생성
4. 계정별 실패 횟수 차트 생성
5. root 로그인 테이블 생성
6. sudo 사용 테이블 생성
7. 위험 이벤트 목록 생성
8. 대시보드 캡처 저장

완료 기준:

대시보드 캡처가 `kibana/screenshots/` 아래에 저장되어야 합니다.

### 9단계: README 및 트러블슈팅 정리

목표:

채용 담당자나 면접관이 프로젝트를 빠르게 이해할 수 있게 문서화합니다.

할 일:

1. 프로젝트 소개 작성
2. 아키텍처 작성
3. 실행 방법 작성
4. 주요 설정 파일 설명
5. 탐지 시나리오 설명
6. 대시보드 캡처 첨부
7. 트러블슈팅 기록 작성

완료 기준:

README만 읽어도 프로젝트 목적, 실행 방법, 기술적 포인트가 이해되어야 합니다.

## 14. 최종 산출물 체크리스트

| 산출물 | 파일 또는 위치 | 상태 |
|---|---|---|
| Docker Compose 기반 실행 환경 | `docker-compose.yml` | 예정 |
| Kafka Topic 설계 문서 | `docs/kafka-topic-design.md` | 예정 |
| Logstash Pipeline 설정 파일 | `logstash/pipeline/*.conf` | 예정 |
| Elasticsearch Index Mapping | `elasticsearch/mappings/*.json` | 예정 |
| Kibana Dashboard 캡처 | `kibana/screenshots/` | 예정 |
| 탐지 시나리오 문서 | `docs/detection-scenarios.md` | 예정 |
| README.md | `README.md` | 예정 |
| 트러블슈팅 기록 | `docs/troubleshooting.md` | 예정 |

## 15. 학습 순서 추천

처음 접한다면 아래 순서로 공부하는 것이 좋습니다.

1. Linux 로그 읽기
2. Docker Compose 기본 사용법
3. Kafka topic과 producer/consumer 개념
4. Filebeat input/output 개념
5. Logstash input/filter/output 구조
6. Grok 패턴으로 로그 파싱하기
7. Elasticsearch index, mapping, query 개념
8. Kibana Discover와 Dashboard 사용법
9. 보안 탐지 조건을 쿼리로 표현하는 방법

## 16. 예상 난이도와 기간

초보자 기준 예상 기간은 3주에서 4주입니다.

| 기간 | 목표 |
|---|---|
| 1주차 | 개념 학습, Docker Compose, Kafka/Elastic Stack 실행 |
| 2주차 | Filebeat 수집, Logstash 파싱, Elasticsearch 적재 |
| 3주차 | 탐지 시나리오, Kibana Dashboard |
| 4주차 | 문서 정리, 캡처, README 개선, 트러블슈팅 정리 |

## 17. 포트폴리오에서 강조할 포인트

이 프로젝트는 다음 역량을 보여줄 수 있습니다.

1. 로그 수집 파이프라인을 직접 설계하고 구현한 경험
2. Kafka 기반 스트리밍 데이터 처리 이해
3. Elastic Stack을 이용한 보안 로그 분석 경험
4. Linux 인증 로그와 웹 access log 분석 경험
5. 보안 탐지 시나리오를 데이터 조건으로 변환한 경험
6. 운영 중 발생할 수 있는 트러블슈팅 기록

면접에서 특히 강조할 문장:

```text
Linux 서버 로그를 Filebeat로 수집해 Kafka topic으로 분리하고,
Logstash에서 공통 필드로 정규화한 뒤 Elasticsearch에 적재했습니다.
이후 Kibana Dashboard와 탐지 쿼리를 통해 SSH Brute Force, root 로그인,
sudo 사용, 과도한 요청 이벤트를 분석할 수 있도록 구성했습니다.
```

## 18. 다음 작업

다음으로 진행할 작업은 프로젝트 뼈대를 만드는 것입니다.

1. 디렉터리 구조 생성
2. `README.md` 초안 작성
3. `docker-compose.yml` 초안 작성
4. Kafka, Elasticsearch, Kibana가 먼저 실행되는지 확인
5. 샘플 로그 파일 추가
