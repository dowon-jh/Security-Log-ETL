# Dashboard 08. Nginx TOP 요청 IP

## 1. 패널 목적

이 패널은 Nginx access log에서 웹 서버에 가장 많이 요청한 IP를 확인하기 위한 패널입니다.

특정 IP에서 요청이 많이 발생하면 정상 사용자의 활동일 수도 있지만, 로그인 반복 시도, 관리자 페이지 접근, 경로 스캔 같은 비정상 접근일 수도 있습니다.

## 2. 사용하는 데이터

Data View:

```text
nginx_access_logs
```

대상 index:

```text
nginx-access-log-*
```

주요 이벤트 타입:

```text
nginx_access
```

## 3. Kibana 설정

Visualization type:

```text
Bar horizontal
```

KQL 필터:

```text
event_type: nginx_access
```

시간 범위:

```text
Last 30 days
```

Horizontal axis:

```text
Count of records
```

Vertical axis:

```text
Top 5 values of source_ip
```

저장 이름:

```text
Nginx TOP 요청 IP
```

## 4. 어떤 필드를 보는가?

주요 필드:

```text
source_ip
http_method
url_path
http_status
user_agent
raw_message
```

`source_ip`는 웹 요청을 보낸 클라이언트 IP입니다.

## 5. 이 패널로 알 수 있는 것

이 패널은 다음 질문에 답합니다.

```text
어떤 IP가 웹 서버에 가장 많이 접근했는가?
특정 IP에서 반복 요청이 발생했는가?
로그인 페이지나 관리자 페이지 접근이 특정 IP에 집중되어 있는가?
자동화 도구로 보이는 요청이 특정 IP에서 반복되는가?
```

## 6. 보안 분석 의미

TOP 요청 IP는 웹 공격의 출발지를 찾는 데 유용합니다.

요청 수가 많은 IP를 발견하면 해당 IP가 접근한 `url_path`, 반환된 `http_status`, 사용한 `user_agent`를 함께 확인해 정상 사용자 활동인지 공격 시도인지 분석할 수 있습니다.

예를 들어 같은 IP가 `/login`에 반복적으로 접근하면서 `401` 응답을 많이 받는다면 웹 로그인 brute force 가능성을 의심할 수 있습니다.

## 7. Dashboard 구성

이 패널은 다음 Dashboard에 포함됩니다.

```text
Nginx 보안 접근 분석 Dashboard
```

Dashboard 구성:

```text
HTTP 상태 코드별 요청 수
Nginx TOP 요청 IP
Nginx 위험 웹 접근 이벤트 목록
```

## 8. 면접 설명 예시

```text
Nginx access log를 source_ip 기준으로 집계해 웹 서버에 가장 많이 요청한 IP를 확인할 수 있도록 구성했습니다.
이를 통해 반복 로그인 시도, 관리자 페이지 접근, 스캔성 요청을 보내는 출발지를 빠르게 식별할 수 있습니다.
```
