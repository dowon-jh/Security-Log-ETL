# Dashboard 03. 계정별 SSH 로그인 실패 횟수

## 1. 패널 목적

이 패널은 어떤 계정명으로 SSH 로그인 실패가 많이 발생했는지 확인하기 위한 시각화입니다.

공격자는 흔한 계정명을 반복적으로 시도하는 경우가 많습니다.

예:

```text
root
admin
test
attacker
ubuntu
```

## 2. 사용하는 데이터

Data View:

```text
security_logs2
```

대상 index:

```text
security-auth-log-*
```

시간 범위:

```text
Last 30 days
```

## 3. Kibana 설정

KQL 필터:

```text
event_type: ssh_failed_login
```

차트 타입:

```text
Horizontal bar
```

또는:

```text
Vertical bar
```

Metric:

```text
Records
```

Group by 또는 Breakdown:

```text
username
```

만약 `username`이 집계 필드로 동작하지 않으면 다음 필드를 사용합니다.

```text
username.keyword
```

정렬:

```text
Count descending
```

표시 개수:

```text
Top 5 또는 Top 10
```

## 4. 확인한 결과

테스트 로그에서는 `attacker` 계정으로 SSH 로그인 실패가 반복 발생했습니다.

예상되는 상위 계정:

```text
attacker
root
admin
test
```

## 5. 이 패널로 알 수 있는 것

이 패널은 다음 질문에 답합니다.

```text
어떤 계정명이 공격 대상이 되었는가?
root 계정에 대한 실패 시도가 있었는가?
admin, test 같은 흔한 계정명이 반복적으로 사용되었는가?
```

## 6. 관련 탐지 시나리오

```text
root 계정 로그인 실패 반복
admin 계정 로그인 실패 반복
존재하지 않는 계정명으로 반복 로그인 시도
```

## 7. 다음 개선 방향

추후 Logstash 파싱에서 `invalid user` 여부를 별도 필드로 분리하면 더 좋은 분석이 가능합니다.

예:

```text
is_invalid_user: true
```

그러면 존재하지 않는 계정을 대상으로 한 공격 시도를 더 명확히 구분할 수 있습니다.
