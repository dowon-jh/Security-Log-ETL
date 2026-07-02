# Dashboard 09. Nginx 위험 웹 접근 이벤트 목록

## 1. 패널 목적

이 패널은 Nginx access log 중 보안적으로 확인이 필요한 웹 접근 이벤트를 목록 형태로 확인하기 위한 패널입니다.

차트 패널이 전체적인 경향을 보여준다면, 이 패널은 실제 요청 원문과 주요 필드를 한 줄씩 확인하는 분석용 목록입니다.

## 2. 사용하는 데이터

Data View:

```text
nginx_access_logs
```

대상 index:

```text
nginx-access-log-*
```

포함하는 이벤트:

```text
HTTP 400 이상 응답
/admin 접근
curl User-Agent 요청
```

## 3. Kibana 설정

이 패널은 Lens Table이 아니라 Discover에서 저장한 Table로 구성합니다.

KQL 필터:

```text
http_status >= 400 or url_path: "/admin" or user_agent: "curl/8.0"
```

시간 범위:

```text
Last 30 days
```

추가한 컬럼:

```text
@timestamp
source_ip
http_method
url_path
http_status
status
user_agent
raw_message
```

저장 이름:

```text
Nginx 위험 웹 접근 이벤트 목록
```

## 4. 이 KQL의 의미

KQL 조건은 다음 의미입니다.

```text
HTTP 상태 코드가 400 이상이거나
/admin 경로에 접근했거나
User-Agent가 curl/8.0인 요청만 보여준다.
```

즉, 웹 접근 로그 중 인증 실패, 접근 거부, 관리자 경로 접근, 자동화 도구 접근처럼 보안적으로 확인할 가치가 있는 이벤트를 모아 보는 조건입니다.

## 5. 어떤 필드를 보는가?

주요 필드:

```text
@timestamp
source_ip
http_method
url_path
http_status
status
user_agent
raw_message
```

각 필드의 의미:

| 필드 | 의미 |
|---|---|
| `@timestamp` | 이벤트 발생 시각 |
| `source_ip` | 요청을 보낸 IP |
| `http_method` | GET, POST 같은 요청 방식 |
| `url_path` | 요청 경로 |
| `http_status` | HTTP 응답 코드 |
| `status` | 성공/실패 분류 |
| `user_agent` | 브라우저 또는 요청 도구 |
| `raw_message` | 원본 Nginx access log |

## 6. 이 패널로 알 수 있는 것

이 패널은 다음 질문에 답합니다.

```text
어떤 IP가 실패성 요청을 보냈는가?
/admin 접근 시도가 있었는가?
curl 같은 자동화 도구 접근이 있었는가?
해당 요청의 원본 로그는 무엇인가?
```

## 7. 보안 분석 의미

위험 웹 접근 이벤트 목록은 차트에서 발견한 이상 징후를 실제 로그로 확인하는 역할을 합니다.

예를 들어 HTTP 상태 코드 패널에서 `401`이나 `403`이 눈에 띄면, 이 목록에서 어떤 IP가 어떤 URL에 접근했고 어떤 User-Agent를 사용했는지 확인할 수 있습니다.

## 8. Dashboard 구성

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

## 9. 면접 설명 예시

```text
상태 코드 400 이상, 관리자 경로 접근, 자동화 도구 User-Agent를 위험 조건으로 보고 Discover Saved Search 형태의 이벤트 목록을 구성했습니다.
차트에서 이상 패턴을 발견한 뒤 실제 요청 원문과 IP, URL, 상태 코드를 확인할 수 있도록 만들었습니다.
```
