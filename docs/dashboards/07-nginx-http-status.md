# Dashboard 07. HTTP 상태 코드별 요청 수

## 1. 패널 목적

이 패널은 Nginx access log를 HTTP 상태 코드 기준으로 집계하여 정상 요청과 실패성 요청의 비율을 확인하기 위한 패널입니다.

HTTP 상태 코드는 웹 요청이 어떤 결과로 끝났는지 알려줍니다. 따라서 `200`, `401`, `403` 같은 값을 보면 웹 서비스 접근이 정상인지, 인증 실패나 권한 없는 접근이 발생했는지 빠르게 파악할 수 있습니다.

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
Donut
```

KQL 필터:

```text
event_type: nginx_access
```

시간 범위:

```text
Last 30 days
```

Metric:

```text
Count of records
```

Slice by:

```text
Top values of http_status
```

저장 이름:

```text
HTTP 상태 코드별 요청 수
```

## 4. 어떤 필드를 보는가?

주요 필드:

```text
http_status
event_type
status
log_level
raw_message
```

`http_status`는 웹 서버가 클라이언트 요청에 대해 반환한 HTTP 응답 코드입니다.

예시:

```text
200 = 정상 응답
401 = 인증 실패
403 = 접근 금지
404 = 존재하지 않는 경로
500 = 서버 오류
```

## 5. 이 패널로 알 수 있는 것

이 패널은 다음 질문에 답합니다.

```text
정상 요청과 실패성 요청의 비율은 어떤가?
401 인증 실패가 많이 발생했는가?
403 접근 거부가 발생했는가?
웹 서비스에 비정상 요청이 집중되고 있는가?
```

## 6. 보안 분석 의미

HTTP 상태 코드는 웹 공격 징후를 찾는 출발점입니다.

예를 들어 `401`이 반복되면 로그인 실패나 credential stuffing 가능성을 의심할 수 있고, `403`은 권한이 없는 경로에 접근하려는 시도를 의미할 수 있습니다. `404`가 많으면 존재하지 않는 경로를 대량으로 탐색하는 스캔 행위일 수 있습니다.

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
Nginx access log를 http_status 기준으로 집계해 정상 응답과 실패성 응답의 비율을 확인할 수 있도록 구성했습니다.
401, 403 같은 응답은 인증 실패나 권한 없는 접근과 연결될 수 있기 때문에 웹 접근 이상 징후를 빠르게 파악하는 데 활용할 수 있습니다.
```
