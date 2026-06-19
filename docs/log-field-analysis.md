# 로그 필드 분석

## 1. 이 문서의 목적

이 문서는 ETL 프로젝트의 첫 단계입니다.

목표는 로그 한 줄을 보고 다음 질문에 답할 수 있게 되는 것입니다.

1. 언제 발생한 로그인가?
2. 어떤 서비스에서 발생했는가?
3. 어떤 IP가 관련되어 있는가?
4. 어떤 사용자가 관련되어 있는가?
5. 성공인가, 실패인가?
6. 보안적으로 의미 있는 이벤트인가?

아직 Filebeat, Kafka, Logstash, Elasticsearch를 몰라도 괜찮습니다.

먼저 로그를 읽을 줄 알아야 이후에 어떤 필드를 추출해야 하는지 이해할 수 있습니다.

## 2. 공통 필드

이 프로젝트에서는 서로 다른 로그를 비슷한 형태로 분석하기 위해 아래 필드를 공통으로 사용합니다.

| 필드명 | 의미 | 예시 |
|---|---|---|
| timestamp | 이벤트 발생 시간 | 2026-06-17T21:10:33+09:00 |
| source_ip | 접속을 시도하거나 요청을 보낸 IP | 192.168.0.10 |
| username | 로그인 또는 명령 실행 계정 | admin, ubuntu, root |
| event_type | 이벤트 종류 | ssh_failed_login, sudo_command |
| status | 성공 또는 실패 | success, failed |
| service | 로그를 만든 서비스 | sshd, nginx, docker |
| log_level | 로그 심각도 | info, warn, error |
| raw_message | 원본 로그 전체 | 로그 원문 |

## 3. auth.log란?

`auth.log`는 Linux 서버의 인증 관련 로그입니다.

주로 다음 이벤트가 기록됩니다.

1. SSH 로그인 성공
2. SSH 로그인 실패
3. 존재하지 않는 계정으로 로그인 시도
4. root 계정 로그인
5. sudo 명령어 사용

보안 로그 분석에서 가장 중요한 로그 중 하나입니다.

### 3.1 SSH 로그인 실패 예시

원본 로그:

```text
Jun 17 21:10:33 ubuntu sshd[1234]: Failed password for invalid user admin from 192.168.0.10 port 54321 ssh2
```

해석:

```text
6월 17일 21시 10분 33초에
ubuntu 서버에서
sshd 서비스가
admin이라는 존재하지 않는 계정으로 들어온 SSH 로그인 실패를 기록했다.
접속 IP는 192.168.0.10이다.
```

필드 분석:

| 필드명 | 값 |
|---|---|
| timestamp | Jun 17 21:10:33 |
| source_ip | 192.168.0.10 |
| username | admin |
| event_type | ssh_failed_login |
| status | failed |
| service | sshd |
| log_level | info |
| raw_message | 원본 로그 전체 |

### 3.2 SSH 로그인 성공 예시

원본 로그:

```text
Jun 17 21:12:15 ubuntu sshd[1236]: Accepted password for ubuntu from 192.168.0.20 port 50222 ssh2
```

해석:

```text
ubuntu 계정으로 SSH 로그인이 성공했다.
접속 IP는 192.168.0.20이다.
```

필드 분석:

| 필드명 | 값 |
|---|---|
| timestamp | Jun 17 21:12:15 |
| source_ip | 192.168.0.20 |
| username | ubuntu |
| event_type | ssh_success_login |
| status | success |
| service | sshd |
| log_level | info |
| raw_message | 원본 로그 전체 |

### 3.3 sudo 명령어 사용 예시

원본 로그:

```text
Jun 17 21:13:44 ubuntu sudo:   ubuntu : TTY=pts/0 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/bin/apt update
```

해석:

```text
ubuntu 사용자가 sudo를 이용해 root 권한으로 apt update 명령어를 실행했다.
```

필드 분석:

| 필드명 | 값 |
|---|---|
| timestamp | Jun 17 21:13:44 |
| source_ip | 없음 |
| username | ubuntu |
| event_type | sudo_command |
| status | success |
| service | sudo |
| log_level | info |
| raw_message | 원본 로그 전체 |

## 4. nginx access log란?

`nginx access log`는 Nginx 웹서버에 들어온 HTTP 요청 기록입니다.

주로 다음 정보를 확인할 수 있습니다.

1. 요청한 클라이언트 IP
2. 요청 시간
3. HTTP method
4. 요청 URL
5. 응답 코드
6. 응답 크기
7. User-Agent

웹 공격, 비정상 접근, 과도한 요청 탐지에 사용됩니다.

### 4.1 Nginx 요청 성공 예시

원본 로그:

```text
192.168.0.10 - - [17/Jun/2026:21:10:33 +0900] "GET /login HTTP/1.1" 200 1532 "-" "Mozilla/5.0"
```

해석:

```text
192.168.0.10 IP가 /login 페이지를 GET 방식으로 요청했고,
서버는 200 응답을 반환했다.
```

필드 분석:

| 필드명 | 값 |
|---|---|
| timestamp | 17/Jun/2026:21:10:33 +0900 |
| source_ip | 192.168.0.10 |
| username | 없음 |
| event_type | web_access |
| status | success |
| service | nginx |
| http_method | GET |
| request_path | /login |
| response_code | 200 |
| raw_message | 원본 로그 전체 |

### 4.2 로그인 실패 요청 예시

원본 로그:

```text
203.0.113.10 - - [17/Jun/2026:21:10:35 +0900] "POST /login HTTP/1.1" 401 342 "https://example.com/login" "curl/8.0"
```

해석:

```text
203.0.113.10 IP가 /login에 POST 요청을 보냈고,
서버는 401 Unauthorized 응답을 반환했다.
```

필드 분석:

| 필드명 | 값 |
|---|---|
| timestamp | 17/Jun/2026:21:10:35 +0900 |
| source_ip | 203.0.113.10 |
| username | 없음 |
| event_type | web_login_failed |
| status | failed |
| service | nginx |
| http_method | POST |
| request_path | /login |
| response_code | 401 |
| raw_message | 원본 로그 전체 |

## 5. docker-container.log란?

Docker container log는 컨테이너 내부 애플리케이션이 출력한 로그입니다.

예를 들어 `user-service`, `auth-service`, `payment-service` 같은 애플리케이션이 남긴 실행 기록입니다.

주로 다음 정보를 확인할 수 있습니다.

1. 애플리케이션 시작 여부
2. 로그인 실패 같은 비즈니스 이벤트
3. 에러 발생 여부
4. 장애 원인
5. 로그 레벨

### 5.1 Docker 로그 예시

원본 로그:

```json
{"time":"2026-06-17T21:11:02.234567890+09:00","stream":"stdout","log":"WARN auth-service login failed username=admin source_ip=192.168.0.10"}
```

해석:

```text
auth-service에서 로그인 실패가 발생했다.
사용자명은 admin이고, 접속 IP는 192.168.0.10이다.
로그 레벨은 WARN이다.
```

필드 분석:

| 필드명 | 값 |
|---|---|
| timestamp | 2026-06-17T21:11:02.234567890+09:00 |
| source_ip | 192.168.0.10 |
| username | admin |
| event_type | application_login_failed |
| status | failed |
| service | auth-service |
| log_level | warn |
| raw_message | 원본 로그 전체 |

### 5.2 Docker 에러 로그 예시

원본 로그:

```json
{"time":"2026-06-17T21:12:15.345678901+09:00","stream":"stderr","log":"ERROR payment-service database connection timeout"}
```

해석:

```text
payment-service에서 데이터베이스 연결 시간 초과 에러가 발생했다.
```

필드 분석:

| 필드명 | 값 |
|---|---|
| timestamp | 2026-06-17T21:12:15.345678901+09:00 |
| source_ip | 없음 |
| username | 없음 |
| event_type | application_error |
| status | failed |
| service | payment-service |
| log_level | error |
| raw_message | 원본 로그 전체 |

## 6. 1단계 완료 기준

아래 질문에 답할 수 있으면 1단계는 완료입니다.

1. `auth.log`에는 어떤 로그가 남는가?
2. `nginx access log`에는 어떤 로그가 남는가?
3. `docker-container.log`에는 어떤 로그가 남는가?
4. SSH 로그인 실패 로그에서 IP와 username을 찾을 수 있는가?
5. Nginx 로그에서 요청 URL과 응답 코드를 찾을 수 있는가?
6. Docker 로그에서 log level과 service를 찾을 수 있는가?

## 7. 다음 단계

다음 단계는 Docker Compose 기반 실행 환경을 준비하는 것입니다.

다음 단계에서 만들 파일은 다음과 같습니다.

```text
docker-compose.yml
```

그리고 실행할 서비스는 다음과 같습니다.

1. Kafka
2. Elasticsearch
3. Kibana
4. Logstash

처음에는 Filebeat까지 한 번에 붙이지 않고, 전체 도구가 정상 실행되는지 먼저 확인합니다.
