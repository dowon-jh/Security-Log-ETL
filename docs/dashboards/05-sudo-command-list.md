# Dashboard 05. sudo 명령어 사용 목록

## 1. 패널 목적

이 패널은 Linux 서버에서 `sudo` 명령어가 사용된 기록을 목록 형태로 확인하기 위한 패널입니다.

`sudo`는 일반 사용자가 관리자 권한으로 명령을 실행할 때 사용합니다. 따라서 sudo 사용 기록은 권한 상승 행위를 추적하는 데 중요합니다.

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
sudo_command
```

## 3. Kibana 설정

이 패널은 Lens Table이 아니라 Discover에서 저장한 Table로 구성합니다.

KQL 필터:

```text
event_type: sudo_command
```

시간 범위:

```text
Last 30 days
```

추가한 컬럼:

```text
@timestamp
username
sudo_command
status
raw_message
```

저장 이름:

```text
sudo 명령어 사용 목록
```

## 4. 왜 Lens Table을 사용하지 않았는가?

Lens Table의 Rows에 필드를 넣으면 Kibana가 자동으로 `Top values` 집계를 만듭니다.

예를 들어 `username`을 넣으면 다음처럼 표시됩니다.

```text
Top 3 values of username
```

이 방식은 "어떤 사용자가 가장 많이 sudo를 사용했는가?" 같은 통계에는 좋습니다. 하지만 이번 패널의 목적은 실제 sudo 사용 로그를 한 줄씩 확인하는 것입니다.

그래서 Discover Saved Search로 저장한 Table을 사용했습니다.

## 5. 이 패널로 알 수 있는 것

이 패널은 다음 질문에 답합니다.

```text
누가 sudo 명령어를 실행했는가?
어떤 명령어를 실행했는가?
언제 실행했는가?
실행 결과는 성공인가 실패인가?
원본 로그에는 어떤 내용이 남았는가?
```

## 6. 관련 탐지 시나리오

```text
sudo 명령어 사용 이벤트 탐지
```

## 7. 실무 관점

실무 보안 관제에서는 sudo 사용 기록을 통해 권한 상승 행위를 확인합니다.

예를 들어 일반 계정이 갑자기 `cat /etc/shadow`, `useradd`, `chmod`, `systemctl` 같은 민감한 명령을 실행했다면 점검 대상이 될 수 있습니다.

이번 프로젝트에서는 sudo 이벤트를 먼저 탐지하고 목록화하는 것까지 구현했습니다. 이후 확장 단계에서는 위험 명령어 목록을 따로 정의해서 `sudo_command` 값을 기준으로 위험도를 부여할 수 있습니다.

