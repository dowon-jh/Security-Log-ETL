# Dashboard 04. root 로그인 발생 목록

## 1. 패널 목적

이 패널은 `root` 계정으로 로그인한 이벤트를 목록 형태로 확인하기 위한 패널입니다.

`root`는 Linux 시스템에서 가장 높은 권한을 가진 관리자 계정입니다. 따라서 root 계정으로 직접 로그인한 기록은 정상 작업인지, 비정상 접근인지 반드시 확인해야 합니다.

## 2. 사용하는 데이터

Data View:

```text
security_logs2
```

대상 index:

```text
security-auth-log-*
```

주요 이벤트 타입:

```text
root_login
```

## 3. Kibana 설정

이 패널은 Lens Table이 아니라 Discover에서 저장한 Table로 구성합니다.

KQL 필터:

```text
event_type: root_login
```

시간 범위:

```text
Last 30 days
```

추가한 컬럼:

```text
@timestamp
auth_method
source_ip
username
status
raw_message
```

저장 이름:

```text
root 로그인 발생 목록
```

## 4. 왜 Discover Table을 사용하는가?

root 로그인은 통계보다 실제 로그 원문 확인이 더 중요합니다.

예를 들어 다음 내용을 한 줄씩 확인해야 합니다.

```text
언제 로그인했는가?
어떤 IP에서 접근했는가?
어떤 인증 방식으로 로그인했는가?
로그인이 성공했는가?
원본 로그 메시지는 무엇인가?
```

따라서 값의 순위를 보여주는 Lens Top values보다, 실제 문서 목록을 보여주는 Discover Saved Search가 더 적합합니다.

## 5. 이 패널로 알 수 있는 것

이 패널은 다음 질문에 답합니다.

```text
root 계정으로 로그인한 기록이 있는가?
root 로그인이 성공했는가?
접근한 IP는 어디인가?
정상적인 관리자 작업으로 볼 수 있는가?
```

## 6. 관련 탐지 시나리오

```text
root 계정 로그인 발생 -> 관리자 계정 접근 탐지
```

## 7. 실무 관점

실무에서는 root 직접 로그인을 제한하거나 금지하는 경우가 많습니다.

관리자는 일반 계정으로 접속한 뒤 `sudo`를 통해 필요한 명령만 실행하는 방식이 더 안전합니다. 그래서 root 로그인 이벤트는 단순한 로그인 성공 기록이 아니라, 보안 관점에서 확인해야 할 위험 이벤트로 분류할 수 있습니다.

