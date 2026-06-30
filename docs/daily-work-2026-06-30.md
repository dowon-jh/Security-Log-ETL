# 2026-06-30 작업 정리

## 1. 오늘의 목표

오늘의 목표는 Kibana Dashboard를 완성하는 것이었습니다.

이전 작업에서 이미 다음 패널을 만들었습니다.

```text
시간대별 SSH 로그인 실패 추이
SSH 로그인 실패 TOP IP
계정별 SSH 로그인 실패 횟수
```

오늘은 남은 목록형 패널을 추가해서 보안 로그 분석 Dashboard의 기본 구성을 완성했습니다.

## 2. 오늘 만든 Dashboard 패널

오늘 만든 패널은 다음 3개입니다.

```text
root 로그인 발생 목록
sudo 명령어 사용 목록
위험 이벤트 목록
```

이 3개 패널은 모두 Lens Table이 아니라 Discover에서 저장한 Table 방식으로 만들었습니다.

## 3. root 로그인 발생 목록

문서:

```text
docs/dashboards/04-root-login-list.md
```

KQL:

```text
event_type: root_login
```

목적:

```text
root 계정으로 로그인한 이벤트를 확인합니다.
```

보안 의미:

```text
root는 Linux 최고 권한 계정이므로 직접 로그인 발생 여부를 확인해야 합니다.
```

## 4. sudo 명령어 사용 목록

문서:

```text
docs/dashboards/05-sudo-command-list.md
```

KQL:

```text
event_type: sudo_command
```

목적:

```text
sudo 명령어 사용 기록을 확인합니다.
```

보안 의미:

```text
sudo는 권한 상승 행위와 관련되므로 누가 어떤 명령을 실행했는지 추적해야 합니다.
```

## 5. 위험 이벤트 목록

문서:

```text
docs/dashboards/06-risk-events-list.md
```

KQL:

```text
event_type: ssh_failed_login or event_type: root_login or event_type: sudo_command
```

목적:

```text
주요 보안 이벤트를 하나의 목록에서 확인합니다.
```

보안 의미:

```text
SSH 로그인 실패, root 로그인, sudo 사용 이벤트를 함께 확인하여 위험 이벤트 흐름을 파악합니다.
```

## 6. 오늘 완성한 Dashboard 구성

최종 Dashboard는 다음 6개 패널로 구성됩니다.

```text
1. 시간대별 SSH 로그인 실패 추이
2. SSH 로그인 실패 TOP IP
3. 계정별 SSH 로그인 실패 횟수
4. root 로그인 발생 목록
5. sudo 명령어 사용 목록
6. 위험 이벤트 목록
```

## 7. 오늘 학습한 Kibana 개념

### 7.1 Lens

Lens는 차트와 집계를 만들 때 사용합니다.

예시:

```text
시간대별 추이
TOP IP
계정별 실패 횟수
```

Lens에서는 필드를 Rows에 넣으면 `Top values` 집계가 자동으로 만들어질 수 있습니다.

### 7.2 Discover

Discover는 Elasticsearch에 저장된 실제 로그 문서를 검색하고 확인하는 화면입니다.

예시:

```text
root 로그인 원본 로그 목록
sudo 명령어 사용 원본 로그 목록
위험 이벤트 원본 로그 목록
```

### 7.3 Discover Saved Search

Discover Saved Search는 KQL 조건, 선택한 컬럼, 정렬 상태를 저장해서 Dashboard에 붙이는 방식입니다.

목록형 패널을 만들 때 적합합니다.

```text
통계 차트 = Lens
원본 로그 목록 = Discover Saved Search
```

## 8. 오늘 해결한 질문

### 8.1 Explore in Discover는 무엇인가?

`Explore in Discover`는 현재 Lens에서 보고 있는 조건을 Discover 화면으로 가져가 실제 로그 원본을 확인하는 기능입니다.

즉, 차트를 만들기 전에 실제 데이터가 존재하는지 확인하는 통로로 사용할 수 있습니다.

### 8.2 Lens Table에서 Top values를 없앨 수 있는가?

Lens Table의 Rows는 집계 기반입니다.

그래서 필드를 넣으면 다음처럼 표시됩니다.

```text
Top 3 values of username
Top 3 values of source_ip.keyword
```

이 설정은 통계에는 적합하지만, 원본 로그 목록에는 적합하지 않습니다.

따라서 목록형 패널은 Lens Table이 아니라 Discover Saved Search로 만들었습니다.

### 8.3 source_ip와 source_ip.keyword의 차이는 무엇인가?

```text
source_ip = 실제 IP 값을 표시하는 필드
source_ip.keyword = 집계와 그룹화에 사용하는 하위 필드
```

Discover 목록에서는 `source_ip`를 사용해도 됩니다.

Lens에서 TOP IP처럼 그룹화가 필요하면 `source_ip.keyword`를 사용하는 것이 좋습니다.

### 8.4 source_ip 옆의 경고 표시는 무엇인가?

경고 표시는 Data View에 포함된 여러 index에서 같은 필드의 타입이나 매핑이 완전히 일치하지 않을 수 있다는 의미입니다.

이번 프로젝트에서는 초반에 만든 index와 이후 mapping을 적용한 index가 함께 Data View에 포함되어 있어서 이런 표시가 나올 수 있습니다.

## 9. 현재 프로젝트 진행 상태

현재까지 완료된 항목:

```text
Docker Compose 기반 실행 환경
Kafka topic 구성
Logstash auth.log 파싱
Elasticsearch index 저장
Elasticsearch query 파일 작성
Kibana Discover 필터링 실습
Kibana Dashboard 6개 패널 구성
Dashboard 패널별 문서화
```

아직 남은 항목:

```text
Dashboard 캡처 저장
탐지 시나리오 문서 최종 정리
README.md 최종 정리
트러블슈팅 기록 보강
GitHub 업로드
```

## 10. 다음 단계

다음 작업은 프로젝트를 포트폴리오 형태로 정리하는 것입니다.

추천 순서:

```text
1. Kibana Dashboard 전체 화면 캡처
2. 탐지 시나리오 문서 정리
3. README.md에 전체 실행 방법과 아키텍처 정리
4. 트러블슈팅 기록 보강
5. GitHub에 최종 push
```

이 단계까지 완료하면 단순 실습이 아니라, 면접에서 설명 가능한 보안 로그 ETL 포트폴리오 프로젝트 형태가 됩니다.

