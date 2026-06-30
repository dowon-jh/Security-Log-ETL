# 현재까지 구축 내용 및 로그 확인 방법

## 1. 이 문서의 목적

이 문서는 지금까지 진행한 Security Log ETL 프로젝트의 진행 상황을 한 번에 확인하기 위한 정리 문서입니다.

포함 내용:

1. 지금까지 나온 주요 질문과 답변
2. 현재까지 구축한 파일과 역할
3. 현재 ETL 데이터 흐름
4. Kafka topic 역할
5. Docker Compose 실행 및 확인 방법
6. Elasticsearch 로그 확인 방법
7. Kibana Discover에서 로그 확인하는 방법
8. 다음 단계

## 2. 지금까지의 학습 질문 정리

### 2.1 auth.log, nginx access log, docker log는 무엇인가?

`auth.log`는 Linux 인증 로그입니다.

주로 SSH 로그인 성공/실패, root 로그인, sudo 명령어 사용 같은 보안 이벤트를 확인할 수 있습니다.

`nginx access log`는 Nginx 웹서버 접근 로그입니다.

누가 어떤 IP로 접속했는지, 어떤 URL을 요청했는지, 응답 코드가 무엇인지 확인할 수 있습니다.

`docker log`는 Docker 컨테이너 내부 애플리케이션이 출력한 로그입니다.

애플리케이션의 INFO, WARN, ERROR 로그나 서비스 장애 이벤트를 확인할 수 있습니다.

### 2.2 Kafka topic은 왜 나누는가?

로그 종류가 다르기 때문에 topic도 분리했습니다.

분리하면 다음 장점이 있습니다.

1. 로그 종류별로 관리하기 쉽습니다.
2. Logstash에서 topic별 파싱 규칙을 다르게 적용할 수 있습니다.
3. 장애나 트래픽 증가 시 특정 로그 흐름만 따로 확장할 수 있습니다.
4. Kibana와 Elasticsearch index 설계도 명확해집니다.

### 2.3 kafka-init은 임시 구현인가, 실무에서도 필요한가?

`kafka-init`은 로컬 학습 환경에서 Kafka topic 생성을 자동화하기 위해 만든 보조 컨테이너입니다.

하지만 “Kafka topic을 코드나 배포 과정에서 관리한다”는 개념은 실무에서도 중요합니다.

실무에서는 보통 다음 방식으로 topic을 관리합니다.

1. Terraform 같은 IaC 도구
2. Kubernetes Job
3. Helm chart
4. CI/CD 배포 스크립트
5. Kafka 관리 도구
6. 인프라 담당자의 사전 생성

현재 프로젝트에서는 `docker compose up -d`만 실행해도 필요한 topic이 자동으로 준비되도록 `kafka-init`을 추가했습니다.

### 2.4 현재 설정 중 실무 개념을 로컬용으로 단순화한 부분은 무엇인가?

현재 구성은 실무 환경을 그대로 복제한 것이 아니라, 학습을 위해 단순화한 로컬 환경입니다.

단순화한 부분:

| 항목 | 현재 로컬 구성 | 실무 환경 예시 |
|---|---|---|
| Kafka broker | 1대 | 3대 이상 |
| Kafka partition | topic당 1개 | 트래픽에 따라 여러 개 |
| Replication factor | 1 | 보통 3 |
| Elasticsearch | single-node | multi-node cluster |
| Elasticsearch security | 비활성화 | 인증, TLS, 권한 제어 적용 |
| Logstash pipeline | 하나의 pipeline | 로그 종류별 pipeline 분리 가능 |
| Elasticsearch index | raw index 하나 | 로그 종류별 index |
| Filebeat | 아직 미연결 | 서버 로그 수집기로 사용 |
| 로그 파싱 | 아직 raw 저장 | grok, dissect, json filter 적용 |

즉, 현재는 먼저 데이터 흐름을 이해하기 위한 최소 구성입니다.

### 2.5 나중에 broker와 replication을 늘려 실무 환경과 비슷하게 만들 수 있는가?

가능합니다.

현재 단일 노드 ETL을 먼저 완성한 뒤, 다음과 같이 확장할 수 있습니다.

```text
Kafka broker 1대
-> Kafka broker 3대

replication factor 1
-> replication factor 3

partition 1개
-> partition 여러 개

Elasticsearch single-node
-> Elasticsearch multi-node
```

추천 순서는 먼저 단일 노드 ETL을 완성한 뒤, 별도 Compose 파일로 실무형 확장 환경을 만드는 것입니다.

예:

```text
docker-compose.yml
docker-compose.prod-like.yml
```

## 3. 현재까지 구축한 파일

### 3.1 PROJECT_PLAN.md

전체 프로젝트 기획안입니다.

포함 내용:

1. 프로젝트 목표
2. 전체 아키텍처
3. 기술 스택
4. 수집 대상 로그
5. Kafka topic 설계
6. Elasticsearch index 설계
7. 보안 탐지 시나리오
8. Kibana Dashboard 계획
9. 산출물 체크리스트
10. 학습 순서

### 3.2 sample-logs/auth.log

Linux 인증 로그 샘플입니다.

포함 이벤트:

1. SSH 로그인 실패
2. root 로그인 실패
3. SSH 로그인 성공
4. sudo 명령어 사용
5. root 계정 로그인 성공

### 3.3 sample-logs/nginx-access.log

Nginx access log 샘플입니다.

포함 이벤트:

1. `/login` 요청
2. 로그인 실패로 볼 수 있는 401 응답
3. `/admin` 접근 시도
4. 정상 API 요청

### 3.4 sample-logs/docker-container.log

Docker 컨테이너 로그 샘플입니다.

포함 이벤트:

1. 서비스 시작 로그
2. 인증 서비스 로그인 실패
3. 결제 서비스 DB 연결 오류
4. 주문 서비스 정상 요청
5. 토큰 검증 실패

### 3.5 docs/log-field-analysis.md

로그 한 줄을 어떻게 읽고 어떤 필드를 뽑을지 정리한 문서입니다.

초보자 관점에서 다음을 설명합니다.

1. auth.log 의미
2. nginx access log 의미
3. docker container log 의미
4. 공통 필드 설계
5. 로그별 필드 분석 예시

### 3.6 docker-compose.yml

로컬 실행 환경을 정의하는 Docker Compose 파일입니다.

실행 서비스:

```text
kafka
kafka-init
elasticsearch
kibana
logstash
```

### 3.7 logstash/pipeline/logstash.conf

Logstash pipeline 설정 파일입니다.

현재 역할:

```text
Kafka topic 3개에서 로그 읽기
-> raw_message 필드 추가
-> Elasticsearch에 etl-raw-log-YYYY.MM.dd index로 저장
```

아직 상세 파싱은 하지 않습니다.

### 3.8 docs/docker-compose-environment.md

Docker Compose 실행 환경 사용 설명서입니다.

포함 내용:

1. 실행 명령
2. 종료 명령
3. 서비스 상태 확인
4. Kafka topic 확인
5. Elasticsearch 확인
6. Kibana 접속
7. 샘플 로그 전송 테스트

### 3.9 docs/learning-qa.md

지금까지 학습하면서 나온 질문과 답변을 정리한 문서입니다.

## 4. 현재 ETL 데이터 흐름

현재까지 확인한 흐름은 다음과 같습니다.

```text
수동 Kafka producer
  -> Kafka topic
  -> Logstash
  -> Elasticsearch
  -> Kibana Discover
```

아직 Filebeat는 연결하지 않았습니다.

현재는 사람이 직접 Kafka topic에 샘플 로그를 넣어 흐름을 검증합니다.

## 5. Kafka topic 역할

### 5.1 security-auth-log

Linux 인증 로그를 저장하는 topic입니다.

대상 로그:

```text
auth.log
```

주요 탐지 후보:

1. SSH 로그인 실패
2. Brute Force 의심
3. root 계정 로그인
4. sudo 명령어 사용

### 5.2 nginx-access-log

Nginx 웹서버 접근 로그를 저장하는 topic입니다.

대상 로그:

```text
nginx access log
```

주요 탐지 후보:

1. 특정 IP의 과도한 요청
2. `/login` 반복 실패
3. `/admin` 접근 시도
4. 401, 403 응답 증가

### 5.3 docker-container-log

Docker 컨테이너 애플리케이션 로그를 저장하는 topic입니다.

대상 로그:

```text
docker container log
```

주요 분석 후보:

1. 애플리케이션 ERROR 로그
2. 인증 서비스 실패
3. DB 연결 장애
4. 서비스별 WARN/ERROR 추이

## 6. 현재 Docker Compose 설정 요약

### 6.1 kafka

Kafka broker 역할을 합니다.

외부 접속 포트:

```text
localhost:29092
```

컨테이너 내부 접속:

```text
kafka:9092
```

### 6.2 kafka-init

Kafka가 정상 실행된 뒤 topic 3개를 자동 생성합니다.

생성 topic:

```text
security-auth-log
nginx-access-log
docker-container-log
```

현재 설정:

```text
partitions: 1
replication-factor: 1
```

### 6.3 elasticsearch

Logstash가 보낸 로그를 저장합니다.

접속 주소:

```text
http://localhost:9200
```

현재는 학습 편의를 위해 보안 설정이 비활성화되어 있습니다.

```yaml
xpack.security.enabled: "false"
```

### 6.4 kibana

Elasticsearch에 저장된 로그를 웹 화면에서 조회하고 시각화합니다.

접속 주소:

```text
http://localhost:5601
```

### 6.5 logstash

Kafka topic에서 로그를 읽어 Elasticsearch에 저장합니다.

현재 index:

```text
etl-raw-log-YYYY.MM.dd
```

## 7. 실행 및 확인 명령어

프로젝트 루트로 이동합니다.

```powershell
cd Security-Log-ETL
```

### 7.1 컨테이너 실행

```powershell
docker compose up -d
```

### 7.2 컨테이너 상태 확인

```powershell
docker compose ps
```

정상적으로 보이면 좋은 컨테이너:

```text
etl-kafka
etl-elasticsearch
etl-kibana
etl-logstash
```

### 7.3 Kafka topic 확인

```powershell
docker exec etl-kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

확인할 topic:

```text
security-auth-log
nginx-access-log
docker-container-log
```

`__consumer_offsets`가 함께 보일 수 있는데 정상입니다.

### 7.4 Elasticsearch 실행 확인

```powershell
curl.exe http://localhost:9200
```

cluster health 확인:

```powershell
curl.exe http://localhost:9200/_cluster/health
```

정상 예시:

```json
{
  "status": "green"
}
```

### 7.5 Kibana 접속 확인

브라우저에서 접속합니다.

```text
http://localhost:5601
```

터미널에서 확인하려면:

```powershell
curl.exe -I http://localhost:5601
```

다음과 같은 응답이 나오면 정상입니다.

```text
HTTP/1.1 302 Found
location: /spaces/enter
```

### 7.6 Logstash 상태 확인

```powershell
docker logs --tail 80 etl-logstash
```

정상 로그 예시:

```text
Successfully joined group
Adding newly assigned partitions
security-auth-log-0
nginx-access-log-0
docker-container-log-0
```

이 메시지는 Logstash가 Kafka topic을 읽을 준비가 되었다는 뜻입니다.

## 8. 샘플 로그 전송 및 Elasticsearch 확인

### 8.1 auth 로그 한 줄을 Kafka로 보내기

```powershell
docker exec etl-kafka /bin/bash -c "echo 'Jun 17 21:10:33 ubuntu sshd[1234]: Failed password for invalid user admin from 192.168.0.10 port 54321 ssh2' | /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic security-auth-log"
```

### 8.2 Elasticsearch에서 저장 확인

5초 정도 기다린 뒤 실행합니다.

```powershell
curl.exe "http://localhost:9200/etl-raw-log-*/_search?pretty"
```

성공하면 다음 필드가 보입니다.

```json
{
  "message": "Jun 17 21:10:33 ubuntu sshd[1234]: Failed password for invalid user admin from 192.168.0.10 port 54321 ssh2",
  "raw_message": "Jun 17 21:10:33 ubuntu sshd[1234]: Failed password for invalid user admin from 192.168.0.10 port 54321 ssh2",
  "pipeline_stage": "raw_ingest"
}
```

이 결과가 보이면 아래 흐름이 정상 작동한 것입니다.

```text
Kafka -> Logstash -> Elasticsearch
```

## 9. Kibana Discover에서 로그 확인하는 방법

### 9.1 Kibana 접속

브라우저에서 접속합니다.

```text
http://localhost:5601
```

### 9.2 Data View 생성

Kibana에서 다음 순서로 이동합니다.

```text
Stack Management
-> Data Views
-> Create data view
```

입력값:

```text
Name 또는 Index pattern: etl-raw-log-*
Time field: @timestamp
```

### 9.3 Discover로 이동

왼쪽 위 햄버거 메뉴를 클릭한 뒤 다음 메뉴로 이동합니다.

```text
Analytics
-> Discover
```

또는 브라우저 주소창에 직접 입력합니다.

```text
http://localhost:5601/app/discover
```

### 9.4 Data View 선택

Discover 화면 상단 또는 왼쪽의 Data View 선택 영역에서 다음을 선택합니다.

```text
etl-raw-log-*
```

### 9.5 시간 범위 조정

오른쪽 위 시간 범위를 넓힙니다.

추천:

```text
Last 24 hours
```

또는:

```text
Last 7 days
```

현재 `@timestamp`는 원본 로그 안의 `Jun 17` 시간이 아니라 Logstash가 처리한 현재 시간으로 들어갑니다.

그래서 시간 범위가 너무 좁으면 로그가 보이지 않을 수 있습니다.

### 9.6 확인할 필드

Discover에서 다음 필드가 보이면 정상입니다.

```text
message
raw_message
pipeline_stage
@timestamp
```

특히 다음 값이 보이면 현재 기본 뼈대 검증은 성공입니다.

```text
pipeline_stage: raw_ingest
raw_message: Jun 17 21:10:33 ubuntu sshd[1234]: Failed password ...
```

## 10. 종료 명령어

컨테이너만 중지합니다.

```powershell
docker compose down
```

데이터까지 완전히 삭제하려면 다음 명령을 사용합니다.

```powershell
docker compose down -v
```

주의:

```text
-v 옵션은 Elasticsearch에 저장된 로그 데이터도 삭제합니다.
```

## 11. 현재 완료된 범위

완료:

1. 프로젝트 기획안 작성
2. 샘플 로그 생성
3. 로그 필드 분석 문서 작성
4. Docker Compose 실행 환경 구성
5. Kafka topic 3개 자동 생성
6. Logstash가 Kafka topic을 consume하도록 설정
7. Elasticsearch 저장 확인
8. Kibana 접속 및 Discover 확인 방법 정리

현재 검증 완료 흐름:

```text
수동 Kafka producer
-> Kafka topic
-> Logstash
-> Elasticsearch
-> Kibana Discover
```

## 12. 아직 남은 작업

다음 작업:

1. Logstash에서 auth.log 상세 파싱
2. `source_ip`, `username`, `event_type`, `status`, `service` 필드 분리
3. 로그 종류별 Elasticsearch index 분리
4. Elasticsearch mapping 작성
5. Filebeat 연결
6. Kibana Dashboard 구성
7. 보안 탐지 시나리오 구현
8. README 작성
9. 트러블슈팅 문서 작성

가장 가까운 다음 단계는 Logstash 파싱입니다.

목표:

```text
raw_message 한 줄
-> source_ip, username, event_type, status, service 같은 구조화 필드로 변환
```
