# Kibana Dashboard 구성 가이드

## 1. 이 단계의 목표

이번 단계의 목표는 `security-auth-log-*` index에 저장된 인증 로그를 Kibana Dashboard로 시각화하는 것입니다.

현재까지 만든 데이터 흐름:

```text
Kafka
-> Logstash auth.log parsing
-> Elasticsearch security-auth-log-* index
-> Kibana Discover
```

이번 단계에서는 Discover에서 확인하던 로그를 Dashboard에 모아 보안 관제 화면처럼 만듭니다.

## 2. Dashboard에 포함할 항목

우선 auth 로그 기준으로 다음 6개 항목을 만듭니다.

```text
1. 시간대별 SSH 로그인 실패 추이
2. TOP 공격 IP
3. 계정별 로그인 실패 횟수
4. root 로그인 발생 목록
5. sudo 명령어 사용 목록
6. 위험 이벤트 목록
```

## 3. 사전 준비

### 3.1 Kibana 접속

브라우저에서 접속합니다.

```text
http://localhost:5601
```

### 3.2 Data View 확인

Kibana에서 다음 Data View가 있어야 합니다.

```text
security-auth-log-*
```

Time field:

```text
@timestamp
```

없다면 다음 메뉴에서 생성합니다.

```text
Stack Management
-> Data Views
-> Create data view
```

### 3.3 시간 범위 설정

Dashboard와 Discover에서 오른쪽 위 시간 범위를 넓게 설정합니다.

추천:

```text
Last 30 days
```

현재 테스트 데이터는 2026년 6월 17일, 6월 22일, 6월 25일에 걸쳐 있습니다.

## 4. 시각화 1: 시간대별 SSH 로그인 실패 추이

### 목적

시간 흐름에 따라 SSH 로그인 실패가 언제 많이 발생했는지 확인합니다.

### 만들 화면

```text
Line chart 또는 Bar chart
```

### 만드는 방법

1. 왼쪽 메뉴에서 `Dashboard`로 이동합니다.
2. `Create dashboard`를 클릭합니다.
3. `Create visualization` 또는 `Create panel`을 클릭합니다.
4. Data View는 `security-auth-log-*`를 선택합니다.
5. 필터를 추가합니다.

```text
event_type: ssh_failed_login
```

6. X축은 `@timestamp`를 사용합니다.
7. Y축은 `Records` 또는 `Count of records`를 사용합니다.
8. 시각화 제목을 입력합니다.

```text
시간대별 SSH 로그인 실패 추이
```

### 이 시각화의 의미

```text
특정 시간대에 로그인 실패가 몰리면 Brute Force 공격 가능성을 의심할 수 있습니다.
```

## 5. 시각화 2: TOP 공격 IP

### 목적

SSH 로그인 실패를 많이 발생시킨 IP를 확인합니다.

### 만들 화면

```text
Horizontal bar chart
```

### 만드는 방법

1. Dashboard에서 `Create visualization`을 클릭합니다.
2. Data View는 `security-auth-log-*`를 선택합니다.
3. 필터를 추가합니다.

```text
event_type: ssh_failed_login
```

4. Y축 또는 Breakdown 기준으로 `source_ip`를 선택합니다.
5. Metric은 `Count of records`를 사용합니다.
6. 정렬은 count 내림차순으로 설정합니다.
7. 표시 개수는 5개 또는 10개로 설정합니다.
8. 제목을 입력합니다.

```text
TOP 공격 IP
```

### 이 시각화의 의미

```text
특정 IP가 반복적으로 로그인 실패를 발생시키는지 확인합니다.
```

현재 테스트 데이터에서는 `203.0.113.200`이 가장 많이 나와야 합니다.

## 6. 시각화 3: 계정별 로그인 실패 횟수

### 목적

어떤 계정으로 로그인 실패가 많이 발생했는지 확인합니다.

### 만들 화면

```text
Bar chart
```

### 만드는 방법

1. Dashboard에서 `Create visualization`을 클릭합니다.
2. Data View는 `security-auth-log-*`를 선택합니다.
3. 필터를 추가합니다.

```text
event_type: ssh_failed_login
```

4. 분류 기준으로 `username`을 선택합니다.
5. Metric은 `Count of records`를 선택합니다.
6. 제목을 입력합니다.

```text
계정별 로그인 실패 횟수
```

### 이 시각화의 의미

```text
admin, root, test, attacker 같은 계정으로 공격 시도가 집중되는지 확인합니다.
```

## 7. 시각화 4: root 로그인 발생 목록

### 목적

root 계정 로그인 성공 이벤트를 확인합니다.

### 만들 화면

```text
Data table
```

### 필터

```text
event_type: root_login
```

### 표시할 필드

```text
@timestamp
source_ip
username
auth_method
status
raw_message
```

### 제목

```text
root 로그인 발생 목록
```

### 이 시각화의 의미

```text
root 계정은 권한이 매우 높기 때문에 직접 로그인 성공이 발생하면 확인이 필요합니다.
```

## 8. 시각화 5: sudo 명령어 사용 목록

### 목적

sudo 명령어 사용 이벤트를 확인합니다.

### 만들 화면

```text
Data table
```

### 필터

```text
event_type: sudo_command
```

### 표시할 필드

```text
@timestamp
username
sudo_command
status
raw_message
```

### 제목

```text
sudo 명령어 사용 목록
```

### 이 시각화의 의미

```text
sudo는 권한 상승 행위이므로 누가 어떤 명령어를 실행했는지 확인해야 합니다.
```

## 9. 시각화 6: 위험 이벤트 목록

### 목적

보안적으로 중요한 이벤트를 한 곳에 모아 봅니다.

### 만들 화면

```text
Data table
```

### 필터

```text
event_type: ssh_failed_login or event_type: root_login or event_type: sudo_command
```

### 표시할 필드

```text
@timestamp
event_type
source_ip
username
status
service
raw_message
```

### 제목

```text
위험 이벤트 목록
```

## 10. Dashboard 저장

모든 panel을 만든 뒤 Dashboard를 저장합니다.

추천 이름:

```text
Security Auth Log Monitoring
```

또는:

```text
보안 인증 로그 모니터링
```

## 11. 캡처 저장

Dashboard를 만든 뒤 캡처 이미지를 저장합니다.

저장 위치:

```text
kibana/screenshots/
```

추천 파일명:

```text
security-auth-dashboard.png
```

## 12. 지금 단계에서 기억할 개념

### 12.1 Discover

로그를 한 줄씩 검색하고 확인하는 화면입니다.

### 12.2 Dashboard

여러 시각화를 모아서 한 화면에서 보는 공간입니다.

### 12.3 Lens

Kibana에서 차트를 만드는 도구입니다.

### 12.4 Data View

Kibana가 Elasticsearch index를 이해하기 위한 연결 설정입니다.

### 12.5 KQL

Kibana에서 사용하는 검색 문법입니다.

예:

```text
event_type: ssh_failed_login
source_ip: "203.0.113.200"
username: root
```

## 13. 다음 단계

Dashboard를 만든 뒤 다음 단계는 두 가지입니다.

1. Dashboard 캡처를 저장합니다.
2. `docs/detection-scenarios.md`에 탐지 시나리오를 문서화합니다.

탐지 시나리오 예:

```text
5분 내 SSH 로그인 실패 20회 이상
root 계정 로그인 발생
sudo 명령어 사용 발생
특정 IP에서 과도한 로그인 실패 발생
```
