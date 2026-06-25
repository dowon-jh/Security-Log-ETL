# Elasticsearch Query 파일 확인 방법

## 1. 이 문서의 목적

이번 단계에서 만든 Elasticsearch query 파일을 어디서 확인하고, 어떻게 열어보고, 어떻게 실행하는지 정리합니다.

## 2. Query 파일 위치

Query 파일은 프로젝트 폴더 안의 아래 경로에 있습니다.

```text
C:\Users\dwjam\OneDrive\문서\ETL_streaming\elasticsearch\queries
```

프로젝트 기준 상대 경로:

```text
elasticsearch/queries
```

## 3. 만든 Query 파일 목록

```text
ssh-failed-login.json
root-login.json
sudo-command.json
failed-login-by-ip.json
brute-force-5m.json
```

각 파일의 역할:

| 파일 | 역할 |
|---|---|
| `ssh-failed-login.json` | SSH 로그인 실패 이벤트 조회 |
| `root-login.json` | root 계정 로그인 이벤트 조회 |
| `sudo-command.json` | sudo 명령어 사용 이벤트 조회 |
| `failed-login-by-ip.json` | SSH 로그인 실패를 IP별로 집계 |
| `brute-force-5m.json` | 5분 내 SSH 실패 20회 이상 발생한 IP 탐지 |

## 4. PowerShell에서 파일 목록 보기

프로젝트 폴더로 이동합니다.

```powershell
cd C:\Users\dwjam\OneDrive\문서\ETL_streaming
```

목록 확인:

```powershell
dir elasticsearch\queries
```

또는:

```powershell
Get-ChildItem elasticsearch\queries
```

이 명령의 역할:

```text
elasticsearch/queries 폴더 안에 어떤 query 파일이 있는지 확인합니다.
```

데이터에는 아무 영향을 주지 않습니다.

## 5. Query 파일 내용 보기

### 5.1 SSH 실패 조회 쿼리 보기

```powershell
Get-Content elasticsearch\queries\ssh-failed-login.json
```

### 5.2 root 로그인 조회 쿼리 보기

```powershell
Get-Content elasticsearch\queries\root-login.json
```

### 5.3 sudo 명령어 조회 쿼리 보기

```powershell
Get-Content elasticsearch\queries\sudo-command.json
```

### 5.4 TOP 실패 IP 집계 쿼리 보기

```powershell
Get-Content elasticsearch\queries\failed-login-by-ip.json
```

### 5.5 Brute Force 탐지 쿼리 보기

```powershell
Get-Content elasticsearch\queries\brute-force-5m.json
```

이 명령들의 역할:

```text
JSON query 파일 내용을 터미널에 출력합니다.
```

데이터에는 아무 영향을 주지 않습니다.

## 6. VS Code에서 열기

VS Code가 설치되어 있다면 폴더 전체를 열 수 있습니다.

```powershell
code elasticsearch\queries
```

특정 파일만 열 수도 있습니다.

```powershell
code elasticsearch\queries\brute-force-5m.json
```

이 명령의 역할:

```text
query 파일을 편집기에서 열어 사람이 읽고 수정할 수 있게 합니다.
```

## 7. Query 파일 실행하기

Query 파일은 Elasticsearch API에 전달해서 실행합니다.

기본 형식:

```powershell
curl.exe -s -X GET "http://localhost:9200/security-auth-log-*/_search?pretty" -H "Content-Type: application/json" --data-binary "@elasticsearch/queries/파일명.json"
```

여기서 중요한 부분:

```text
--data-binary "@elasticsearch/queries/파일명.json"
```

이 부분이 JSON query 파일을 Elasticsearch 요청 본문으로 전달합니다.

## 8. Query 실행 예시

### 8.1 SSH 로그인 실패 조회

```powershell
curl.exe -s -X GET "http://localhost:9200/security-auth-log-*/_search?pretty" -H "Content-Type: application/json" --data-binary "@elasticsearch/queries/ssh-failed-login.json"
```

역할:

```text
event_type이 ssh_failed_login인 로그를 조회합니다.
```

### 8.2 root 로그인 조회

```powershell
curl.exe -s -X GET "http://localhost:9200/security-auth-log-*/_search?pretty" -H "Content-Type: application/json" --data-binary "@elasticsearch/queries/root-login.json"
```

역할:

```text
event_type이 root_login인 로그를 조회합니다.
```

### 8.3 sudo 명령어 사용 조회

```powershell
curl.exe -s -X GET "http://localhost:9200/security-auth-log-*/_search?pretty" -H "Content-Type: application/json" --data-binary "@elasticsearch/queries/sudo-command.json"
```

역할:

```text
event_type이 sudo_command인 로그를 조회합니다.
```

### 8.4 TOP 실패 IP 집계

```powershell
curl.exe -s -X GET "http://localhost:9200/security-auth-log-*/_search?pretty" -H "Content-Type: application/json" --data-binary "@elasticsearch/queries/failed-login-by-ip.json"
```

역할:

```text
SSH 로그인 실패가 많이 발생한 source_ip를 집계합니다.
```

### 8.5 Brute Force 탐지

```powershell
curl.exe -s -X GET "http://localhost:9200/security-auth-log-*/_search?pretty" -H "Content-Type: application/json" --data-binary "@elasticsearch/queries/brute-force-5m.json"
```

역할:

```text
5분 안에 같은 IP에서 SSH 로그인 실패가 20회 이상 발생했는지 탐지합니다.
```

## 9. 명령어가 데이터에 끼치는 영향

파일 목록 보기:

```powershell
dir elasticsearch\queries
```

영향:

```text
파일 목록만 확인합니다.
Elasticsearch 데이터에는 영향이 없습니다.
```

파일 내용 보기:

```powershell
Get-Content elasticsearch\queries\brute-force-5m.json
```

영향:

```text
파일 내용을 읽기만 합니다.
Elasticsearch 데이터에는 영향이 없습니다.
```

Query 실행:

```powershell
curl.exe -s -X GET "http://localhost:9200/security-auth-log-*/_search?pretty" -H "Content-Type: application/json" --data-binary "@elasticsearch/queries/brute-force-5m.json"
```

영향:

```text
Elasticsearch에 저장된 데이터를 조회합니다.
데이터를 수정하거나 삭제하지 않습니다.
```

## 10. 한 줄 요약

```text
파일 보기 = Get-Content
쿼리 실행 = curl.exe --data-binary "@파일경로"
```
