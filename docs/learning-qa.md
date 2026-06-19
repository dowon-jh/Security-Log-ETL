# 학습 Q&A 정리

## 1. 이 문서의 목적

이 문서는 프로젝트를 진행하면서 나온 질문과 답변을 정리한 학습 노트입니다.

단순히 설정 파일을 만드는 것에서 끝나지 않고, 왜 이런 구성을 했는지, 실무에서는 어떻게 달라지는지 이해하는 것이 목적입니다.

## 2. auth.log, nginx access log, docker log는 다음 단계에서 알 수 있는가?

### 질문

```text
auth.log, nginx access log, docker log가 각각 무엇인지 다음 단계에서 알 수 있어?
없다면 다음 단계와 함께 답변해줘.
```

### 답변

알 수 있습니다.

프로젝트 1단계의 목적이 바로 로그 종류를 이해하는 것입니다.

처음부터 Filebeat, Kafka, Logstash를 모두 이해할 필요는 없습니다. 먼저 로그 한 줄을 보고 어떤 의미인지 읽을 수 있어야 합니다.

### auth.log

`auth.log`는 Linux 서버의 인증 관련 로그입니다.

주로 다음 이벤트가 기록됩니다.

1. SSH 로그인 성공
2. SSH 로그인 실패
3. 존재하지 않는 계정으로 로그인 시도
4. root 계정 로그인
5. sudo 명령어 사용

예시:

```text
Jun 17 21:10:33 ubuntu sshd[1234]: Failed password for invalid user admin from 192.168.0.10 port 54321 ssh2
```

해석:

```text
admin이라는 계정으로 SSH 로그인을 시도했지만 실패했다.
접속 IP는 192.168.0.10이다.
```

### nginx access log

`nginx access log`는 Nginx 웹서버에 들어온 HTTP 요청 기록입니다.

주로 다음 정보를 확인할 수 있습니다.

1. 요청한 IP
2. 요청 시간
3. HTTP method
4. 요청 URL
5. 응답 코드
6. User-Agent

예시:

```text
203.0.113.10 - - [17/Jun/2026:21:10:35 +0900] "POST /login HTTP/1.1" 401 342 "-" "curl/8.0"
```

해석:

```text
203.0.113.10 IP가 /login에 POST 요청을 보냈고,
서버는 401 Unauthorized 응답을 반환했다.
```

### docker log

`docker log`는 Docker 컨테이너 내부 애플리케이션이 출력한 로그입니다.

주로 다음 정보를 확인할 수 있습니다.

1. 애플리케이션 시작 여부
2. 로그인 실패 같은 서비스 이벤트
3. 에러 발생 여부
4. 장애 원인
5. 로그 레벨

예시:

```json
{"time":"2026-06-17T21:11:02.234567890+09:00","stream":"stdout","log":"WARN auth-service login failed username=admin source_ip=192.168.0.10"}
```

해석:

```text
auth-service에서 admin 사용자의 로그인 실패가 발생했다.
접속 IP는 192.168.0.10이고 로그 레벨은 WARN이다.
```

## 3. Kafka topic의 역할은 무엇인가?

### 질문

```text
각 topic의 역할과 네가 추가한 파일의 역할과 설정이 어떻게 되어있는지 알려줘.
```

### 답변

Kafka topic은 로그 종류별 메시지 저장소입니다.

이 프로젝트에서는 로그 종류가 다르기 때문에 topic도 분리했습니다.

### security-auth-log

Linux 인증 로그를 담는 topic입니다.

대상 로그:

```text
auth.log
```

주요 이벤트:

1. SSH 로그인 성공
2. SSH 로그인 실패
3. root 계정 로그인
4. sudo 명령어 사용

나중에 연결되는 탐지 시나리오:

```text
SSH 로그인 실패 20회 이상
root 로그인 탐지
sudo 명령어 사용 탐지
```

### nginx-access-log

Nginx 웹서버 접속 로그를 담는 topic입니다.

대상 로그:

```text
nginx access log
```

주요 이벤트:

1. 특정 IP의 반복 요청
2. 로그인 페이지 요청
3. 관리자 페이지 접근
4. 401, 403, 404 같은 응답 코드

나중에 연결되는 탐지 시나리오:

```text
특정 IP에서 과도한 요청 발생
/login 반복 실패
/admin 접근 시도
```

### docker-container-log

Docker 컨테이너 애플리케이션 로그를 담는 topic입니다.

대상 로그:

```text
docker container log
```

주요 이벤트:

1. 애플리케이션 에러
2. 인증 서비스 실패
3. 서비스 장애
4. WARN 또는 ERROR 로그

## 4. 추가한 파일의 역할은 무엇인가?

### docker-compose.yml

`docker-compose.yml`은 프로젝트 실행 환경의 중심 파일입니다.

Docker로 다음 서비스를 한 번에 실행합니다.

```text
Kafka
Kafka topic 초기화 컨테이너
Elasticsearch
Kibana
Logstash
```

주요 서비스 역할:

| 서비스 | 역할 |
|---|---|
| kafka | 로그 메시지를 topic에 저장 |
| kafka-init | 필요한 topic 3개 자동 생성 |
| elasticsearch | Logstash가 보낸 로그 저장 |
| kibana | Elasticsearch 데이터를 시각화 |
| logstash | Kafka에서 로그를 읽어 Elasticsearch로 전송 |

### logstash/pipeline/logstash.conf

`logstash.conf`는 Logstash의 처리 흐름을 정하는 파일입니다.

Logstash 설정은 크게 3단계입니다.

```text
input
filter
output
```

현재 설정:

```text
input  = Kafka topic 3개에서 로그 읽기
filter = raw_message 필드 추가
output = Elasticsearch에 etl-raw-log-YYYY.MM.dd 형식으로 저장
```

현재는 아직 로그 파싱을 하지 않습니다.

즉, 원본 로그를 그대로 `raw_message`에 저장합니다.

이 단계의 목적은 아래 흐름이 정상 작동하는지 확인하는 것입니다.

```text
Kafka topic -> Logstash -> Elasticsearch
```

### docs/docker-compose-environment.md

`docs/docker-compose-environment.md`는 실행 환경 사용 설명서입니다.

포함 내용:

1. 각 서비스 역할
2. 실행 명령
3. 종료 명령
4. 상태 확인 명령
5. Kafka topic 확인 명령
6. Elasticsearch 확인 명령
7. Kibana 접속 주소
8. 샘플 로그 전송 테스트 방법
9. 다음 단계 안내

## 5. kafka-init은 임시 구현인가, 실제 업무에서도 필요한가?

### 질문

```text
kafka-init이 Kafka가 켜진 뒤 topic 3개를 자동 생성 하는 행위는 실제 상황을 구현하기 위해 지금 프로젝트에서 임시로 만든거야?
아니면 이런 파일이 실제 업무에서도 필요해서 만든거야?
```

### 답변

`kafka-init`은 현재 프로젝트에서는 로컬 학습 환경을 쉽게 재현하기 위해 만든 보조 컨테이너입니다.

하지만 실무에서도 “Kafka topic을 누가, 어떻게, 어떤 설정으로 만들 것인가?”는 반드시 관리해야 합니다.

즉, `kafka-init`이라는 Docker Compose 서비스 자체는 로컬 개발용 구현에 가깝지만, 그 안에 담긴 개념은 실무적입니다.

핵심 개념:

```text
필요한 Kafka topic을 코드로 관리한다.
환경을 자동으로 준비한다.
누가 실행해도 같은 topic 구조가 만들어진다.
```

실무에서는 보통 다음 방식으로 topic을 관리합니다.

1. 인프라 담당자가 Kafka topic을 미리 생성
2. Terraform 같은 IaC 도구로 topic 관리
3. Helm chart나 Kubernetes Job으로 초기 topic 생성
4. CI/CD 배포 과정에서 topic 생성 스크립트 실행
5. Kafka 관리 도구에서 승인 후 생성

면접 답변 예시:

```text
kafka-init은 로컬 Docker Compose 환경에서 topic 생성을 자동화하기 위한 초기화 컨테이너입니다.
실무에서도 topic 생성과 설정 관리는 필요하지만, 운영 환경에서는 보통 Terraform, Kubernetes Job, CI/CD, 또는 Kafka 관리 정책을 통해 관리합니다.
이 프로젝트에서는 누구나 docker compose up만으로 동일한 topic 구조를 재현할 수 있도록 kafka-init을 추가했습니다.
```

## 6. 실무 개념을 로컬 프로젝트에 맞게 단순 구현한 설정이 또 있는가?

### 질문

```text
이거 말고도 실무 개념을 로컬 프로젝트에 맞게 단순 구현한 설정이 있어?
```

### 답변

있습니다.

현재 프로젝트는 실무 구조를 로컬에서 학습할 수 있게 줄인 버전입니다.

### 6.1 단일 Kafka 브로커

현재 Kafka는 1대만 실행됩니다.

```text
etl-kafka 1개
```

실무에서는 장애 대응과 처리량 확보를 위해 Kafka broker를 여러 대 둡니다.

로컬 학습 환경에서는 메시지 흐름을 이해하는 것이 우선이므로 1대로 단순화했습니다.

### 6.2 replication factor 1

현재 topic 생성 설정:

```text
--replication-factor 1
```

실무에서는 보통 복제본을 여러 개 둡니다.

예:

```text
replication-factor 3
```

브로커 한 대가 죽어도 다른 브로커에 데이터가 남아야 하기 때문입니다.

현재는 Kafka 브로커가 1대이므로 replication factor도 1입니다.

### 6.3 partition 1

현재 topic 생성 설정:

```text
--partitions 1
```

실무에서는 로그량과 처리량에 따라 partition을 여러 개 둡니다.

예:

```text
security-auth-log: 3 partitions
nginx-access-log: 6 partitions
docker-container-log: 3 partitions
```

현재는 학습 단계라 메시지 흐름을 보기 쉽게 1개로 두었습니다.

### 6.4 Elasticsearch single-node

현재 설정:

```yaml
discovery.type: "single-node"
```

Elasticsearch를 1대로 실행한다는 뜻입니다.

실무에서는 여러 노드로 구성하고, 역할을 나누기도 합니다.

예:

```text
master node
data node
ingest node
coordinating node
```

### 6.5 Elasticsearch 보안 비활성화

현재 설정:

```yaml
xpack.security.enabled: "false"
```

학습 편의를 위해 Elasticsearch 인증과 보안 기능을 꺼둔 상태입니다.

실무에서는 보통 다음을 적용합니다.

1. 계정과 비밀번호
2. TLS 인증서
3. role-based access control
4. API key
5. 네트워크 접근 제한

### 6.6 Docker Compose 사용

현재는 Docker Compose로 전체 환경을 실행합니다.

```powershell
docker compose up -d
```

실무 운영환경에서는 보통 Kubernetes, Helm, Terraform, Ansible, CI/CD 등을 사용합니다.

하지만 로컬 개발, PoC, 포트폴리오 프로젝트에서는 Docker Compose가 적합합니다.

### 6.7 Logstash pipeline 하나로 topic 3개 처리

현재 `logstash.conf`는 3개 topic을 한 번에 읽습니다.

```text
security-auth-log
nginx-access-log
docker-container-log
```

실무에서는 로그 종류별로 pipeline을 분리하는 경우가 많습니다.

예:

```text
auth-log pipeline
nginx-access-log pipeline
docker-container-log pipeline
```

로그 형식과 파싱 규칙이 다르기 때문입니다.

### 6.8 raw index 하나에 저장

현재 Elasticsearch index:

```text
etl-raw-log-YYYY.MM.dd
```

모든 topic의 로그가 일단 같은 raw index로 들어갑니다.

실무에서는 보통 로그 종류별로 index를 나눕니다.

```text
security-auth-log-YYYY.MM.DD
nginx-access-log-YYYY.MM.DD
docker-container-log-YYYY.MM.DD
```

현재는 “로그가 들어오는지”를 먼저 확인하기 위해 raw index 하나를 사용했습니다.

### 6.9 Filebeat 미연결

실무 흐름은 원래 다음과 같습니다.

```text
Filebeat -> Kafka -> Logstash -> Elasticsearch
```

하지만 현재는 테스트를 위해 사람이 직접 Kafka에 샘플 로그를 넣었습니다.

```text
수동 producer -> Kafka -> Logstash -> Elasticsearch
```

처음부터 Filebeat까지 붙이면 문제 발생 지점을 구분하기 어렵기 때문에, Kafka 이후 구간을 먼저 검증한 것입니다.

### 6.10 로그 파싱 미적용

현재 Logstash는 원본 로그를 그대로 저장합니다.

```text
raw_message
pipeline_stage = raw_ingest
```

실무에서는 grok, dissect, json filter 등을 사용해 필드를 분리합니다.

예:

```text
source_ip
username
event_type
status
service
```

현재는 첫 연결 확인 단계라 정제 전 raw 수집까지만 구현했습니다.

## 7. 현재 구성 요약

현재 프로젝트는 다음 흐름까지 검증했습니다.

```text
Kafka topic -> Logstash -> Elasticsearch
```

아직 구현 전인 부분:

```text
Filebeat 수집
Logstash 상세 파싱
로그 종류별 Elasticsearch index 분리
Elasticsearch mapping
Kibana Dashboard
보안 탐지 시나리오
```

## 8. 다음 학습 주제

다음 단계에서는 Logstash 파싱을 구현합니다.

목표는 `auth.log` 원본 문자열에서 아래 필드를 분리하는 것입니다.

```text
timestamp
source_ip
username
event_type
status
service
raw_message
```

이 단계가 끝나면 단순히 로그를 저장하는 수준에서 벗어나, 검색과 탐지에 사용할 수 있는 구조화된 로그 데이터가 만들어집니다.
