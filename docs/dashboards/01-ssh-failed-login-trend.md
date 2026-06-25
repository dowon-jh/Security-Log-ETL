# Dashboard 01. 시간대별 SSH 로그인 실패 추이

## 1. 패널 목적

이 패널은 시간 흐름에 따라 SSH 로그인 실패가 얼마나 발생했는지 확인하기 위한 시각화입니다.

보안 관점에서는 특정 시간대에 로그인 실패가 갑자기 몰리면 Brute Force 공격 가능성을 의심할 수 있습니다.

## 2. 사용하는 데이터

Data View:

```text
security_logs2
```

대상 index:

```text
security-auth-log-*
```

시간 필드:

```text
@timestamp
```

## 3. Kibana 설정

KQL 필터:

```text
event_type: ssh_failed_login
```

차트 타입:

```text
Bar vertical
```

또는:

```text
Line chart
```

X축:

```text
@timestamp
```

Y축:

```text
Records
```

의미:

```text
시간 구간별 SSH 로그인 실패 로그 개수
```

## 4. 확인한 결과

Dashboard에서 특정 시간대에 큰 막대가 표시되었습니다.

이 막대는 테스트용 Brute Force 로그를 여러 번 넣었기 때문에 발생한 것입니다.

예상되는 공격 IP:

```text
203.0.113.200
```

## 5. 이 패널로 알 수 있는 것

이 패널은 다음 질문에 답합니다.

```text
SSH 로그인 실패가 언제 많이 발생했는가?
특정 시간대에 실패 로그가 급증했는가?
Brute Force 의심 시간대가 있는가?
```

## 6. 관련 탐지 시나리오

```text
5분 내 SSH 로그인 실패 20회 이상 발생
```

이 패널은 Brute Force 탐지의 시간 흐름을 시각적으로 확인하는 데 사용합니다.

## 7. 다음 개선 방향

나중에는 시간 간격을 5분 단위로 조정하면 탐지 시나리오와 더 잘 맞습니다.

예:

```text
@timestamp per 5 minutes
```
