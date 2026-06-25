# Dashboard 02. SSH 로그인 실패 TOP IP

## 1. 패널 목적

이 패널은 SSH 로그인 실패를 가장 많이 발생시킨 IP를 확인하기 위한 시각화입니다.

보안 관점에서는 특정 IP가 반복적으로 로그인 실패를 발생시키면 공격 시도 가능성이 높습니다.

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

또는 테스트 데이터 확인 시:

```text
Jun 25, 2026 21:10 ~ Jun 25, 2026 21:20
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

Metric:

```text
Records
```

Group by 또는 Breakdown:

```text
source_ip.keyword
```

정렬:

```text
Count descending
```

표시 개수:

```text
Top 5 또는 Top 10
```

## 4. source_ip.keyword를 사용한 이유

현재 Data View에서는 `source_ip` 대신 `source_ip.keyword`가 집계용 필드로 보였습니다.

`source_ip.keyword`는 IP 값을 문자열 그대로 묶어서 집계하는 데 사용할 수 있습니다.

예:

```text
203.0.113.200
203.0.113.55
192.168.0.10
```

TOP 공격 IP 패널에서는 IP별 그룹화가 목적이므로 `source_ip.keyword`를 사용해도 괜찮습니다.

## 5. 확인한 결과

가장 큰 막대는 다음 IP로 표시되었습니다.

```text
203.0.113.200
```

이 IP는 Brute Force 테스트 로그에서 반복적으로 SSH 로그인 실패를 발생시킨 IP입니다.

## 6. 이 패널로 알 수 있는 것

이 패널은 다음 질문에 답합니다.

```text
어떤 IP가 SSH 로그인 실패를 가장 많이 발생시켰는가?
공격자로 의심되는 source_ip는 무엇인가?
특정 IP가 압도적으로 많은 실패를 만들고 있는가?
```

## 7. 관련 탐지 시나리오

```text
특정 IP에서 과도한 로그인 실패 발생
5분 내 SSH 로그인 실패 20회 이상 발생
```

## 8. 다음 개선 방향

나중에 Elasticsearch index mapping을 정리하거나 reindex를 수행하면 `source_ip.keyword` 대신 `source_ip` 필드를 일관되게 사용할 수 있습니다.
