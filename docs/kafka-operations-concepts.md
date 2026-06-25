# Kafka 운영 개념 쉽게 이해하기

## 1. 이 문서의 목적

이 문서는 Kafka를 실무에서 다룰 때 자주 나오는 운영 개념을 초보자도 이해할 수 있게 정리한 학습 문서입니다.

이 프로젝트에서는 현재 Kafka를 단일 broker로 단순하게 사용하고 있습니다.

하지만 실무에서는 Kafka broker를 여러 대 운영하고, 장애 상황과 처리 지연을 계속 확인해야 합니다.

이 문서에서 다루는 키워드:

```text
leader partition
follower replica
ISR
acks
unclean leader election
consumer lag
broker disk usage
under replicated partitions
request latency
message in/out rate
controller 상태
consumer 장애
배포
스케일 아웃
네트워크 지연
```

## 2. 기본 예시 상황

실무 Kafka cluster가 아래처럼 3대 broker로 구성되어 있다고 가정합니다.

```text
broker-1
broker-2
broker-3
```

그리고 `security-auth-log` topic이 있습니다.

```text
topic: security-auth-log
partitions: 3
replication factor: 3
```

의미:

```text
partition은 3개
각 partition은 broker 3대에 복제본을 가진다
```

## 3. leader partition

Kafka에서 각 partition은 leader를 하나 가집니다.

producer와 consumer는 기본적으로 leader partition과 통신합니다.

예:

```text
security-auth-log partition-0
  leader: broker-1
  replica: broker-2
  replica: broker-3
```

이 경우 `partition-0`에 메시지를 쓰고 읽는 중심 broker는 `broker-1`입니다.

쉽게 말하면:

```text
leader partition = 해당 partition의 대표 담당자
```

예시:

```text
Filebeat가 security-auth-log에 로그를 보낸다.
Kafka는 해당 로그를 partition-0에 넣기로 결정한다.
partition-0의 leader가 broker-1이면 broker-1이 먼저 로그를 받는다.
```

## 4. follower replica

follower replica는 leader partition의 복제본입니다.

leader가 받은 데이터를 follower들이 따라 복사합니다.

예:

```text
partition-0
  leader: broker-1
  follower: broker-2
  follower: broker-3
```

쉽게 말하면:

```text
follower replica = leader 데이터를 복사해두는 백업 담당자
```

왜 필요할까?

```text
broker-1이 죽어도 broker-2나 broker-3에 데이터가 남아 있어야 하기 때문입니다.
```

## 5. ISR

ISR은 In-Sync Replica의 약자입니다.

leader를 잘 따라가고 있는 replica 목록입니다.

예:

```text
partition-0
  leader: broker-1
  ISR: broker-1, broker-2, broker-3
```

의미:

```text
broker-2와 broker-3이 leader 데이터를 잘 따라가고 있다.
```

만약 broker-3이 느려져서 복제를 제대로 못 따라가면:

```text
ISR: broker-1, broker-2
```

로 줄어들 수 있습니다.

쉽게 말하면:

```text
ISR = 현재 믿을 수 있는 복제본 목록
```

## 6. acks

`acks`는 producer가 Kafka에 메시지를 보낼 때 “어디까지 저장되면 성공으로 볼 것인가”를 정하는 설정입니다.

### 6.1 acks=0

producer가 broker 응답을 기다리지 않습니다.

```text
보냈다고 치고 바로 다음 메시지를 보낸다.
```

장점:

```text
빠르다.
```

단점:

```text
유실 위험이 크다.
```

### 6.2 acks=1

leader가 메시지를 받으면 성공으로 봅니다.

예:

```text
broker-1 leader가 메시지를 받음
producer는 성공으로 판단
```

장점:

```text
속도와 안정성의 균형이 있다.
```

단점:

```text
follower가 복제하기 전에 leader가 죽으면 데이터가 유실될 수 있다.
```

### 6.3 acks=all

leader와 ISR replica들이 메시지를 저장해야 성공으로 봅니다.

장점:

```text
가장 안전하다.
```

단점:

```text
상대적으로 느릴 수 있다.
```

보안 로그에서는 보통 유실을 줄이는 것이 중요하므로 `acks=all`을 고려할 수 있습니다.

## 7. unclean leader election

leader broker가 죽었을 때 새로운 leader를 뽑아야 합니다.

정상적인 경우:

```text
ISR 안에 있는 follower 중 하나를 leader로 승격한다.
```

문제 상황:

```text
ISR에 없는 뒤처진 replica밖에 남지 않았다.
```

이때 뒤처진 replica를 leader로 뽑는 것을 unclean leader election이라고 합니다.

쉽게 말하면:

```text
최신 데이터를 다 못 가진 복제본을 leader로 세우는 위험한 선택
```

장점:

```text
서비스를 계속 살릴 수 있다.
```

단점:

```text
데이터 유실 가능성이 있다.
```

보안 로그처럼 유실이 민감한 데이터에서는 보통 조심해야 하는 설정입니다.

## 8. consumer lag

consumer lag는 Kafka에 쌓인 메시지와 consumer가 처리한 메시지 사이의 차이입니다.

예:

```text
Kafka에는 offset 1000까지 메시지가 있음
Logstash는 offset 700까지 처리함
consumer lag = 300
```

의미:

```text
Logstash가 아직 300개 메시지를 처리하지 못했다.
```

쉽게 말하면:

```text
consumer lag = 소비자가 밀린 양
```

실무에서 매우 중요한 지표입니다.

lag가 계속 증가하면:

```text
Logstash 처리 속도가 로그 유입 속도를 따라가지 못한다.
```

## 9. broker disk usage

broker disk usage는 Kafka broker 디스크 사용량입니다.

Kafka는 메시지를 디스크에 저장합니다.

따라서 디스크가 꽉 차면 매우 위험합니다.

예:

```text
broker-1 disk usage: 92%
```

의미:

```text
Kafka가 더 이상 안전하게 로그를 저장하지 못할 수 있다.
```

원인:

1. 로그 유입량이 너무 많음
2. retention 기간이 너무 김
3. consumer가 느려서 오래 쌓임
4. partition이 특정 broker에 몰림

## 10. under replicated partitions

under replicated partition은 복제본이 정상 개수보다 부족한 partition입니다.

예:

```text
replication factor = 3
정상 replica = broker-1, broker-2, broker-3
현재 replica 정상 동작 = broker-1, broker-2
```

이 경우 under replicated 상태입니다.

쉽게 말하면:

```text
백업 복제본이 부족한 partition
```

위험:

```text
추가 장애가 나면 데이터 유실 가능성이 커진다.
```

## 11. request latency

request latency는 Kafka 요청 처리 시간입니다.

예:

```text
producer가 메시지를 보냄
Kafka가 성공 응답을 주기까지 20ms 걸림
```

여기서 20ms가 latency입니다.

latency가 높아지면:

```text
로그 전송이 느려진다.
producer가 대기한다.
consumer 처리도 늦어질 수 있다.
```

원인:

1. broker 부하
2. 디스크 I/O 병목
3. 네트워크 지연
4. replica 동기화 지연
5. 너무 많은 partition

## 12. message in/out rate

message in rate는 Kafka로 들어오는 메시지 속도입니다.

message out rate는 Kafka에서 consumer가 가져가는 메시지 속도입니다.

예:

```text
message in rate  = 초당 10,000건
message out rate = 초당 7,000건
```

의미:

```text
초당 3,000건씩 Kafka에 밀리고 있다.
```

이 상태가 계속되면 consumer lag가 증가합니다.

쉽게 말하면:

```text
들어오는 속도 > 나가는 속도 = backlog 증가
```

## 13. controller 상태

Kafka controller는 cluster의 관리자 역할을 합니다.

하는 일:

1. broker 상태 관리
2. partition leader 관리
3. 장애 발생 시 leader 재선출
4. metadata 관리

KRaft 모드에서는 ZooKeeper 없이 Kafka controller quorum이 이 역할을 합니다.

controller에 문제가 생기면:

```text
leader 선출 지연
metadata 업데이트 지연
cluster 전체 불안정
```

쉽게 말하면:

```text
controller = Kafka cluster의 교통정리 담당자
```

## 14. consumer 장애

consumer는 Kafka에서 메시지를 읽는 프로그램입니다.

우리 프로젝트에서는 Logstash가 consumer입니다.

예:

```text
Logstash가 죽음
```

영향:

```text
Kafka topic에는 로그가 계속 쌓인다.
Elasticsearch에는 새 로그가 들어가지 않는다.
consumer lag가 증가한다.
```

복구:

```text
Logstash를 다시 시작하면 마지막 offset 이후부터 다시 읽는다.
```

## 15. 배포

배포는 consumer나 producer 애플리케이션을 새 버전으로 바꾸는 작업입니다.

예:

```text
Logstash pipeline 수정 후 재시작
```

영향:

```text
재시작 중에는 잠깐 consume이 멈출 수 있다.
consumer group rebalance가 발생할 수 있다.
```

실무에서는 무중단 또는 순차 배포를 고려합니다.

## 16. 스케일 아웃

스케일 아웃은 처리량을 늘리기 위해 인스턴스를 추가하는 것입니다.

예:

```text
Logstash 1대 -> Logstash 3대
```

효과:

```text
여러 consumer가 partition을 나눠 읽을 수 있다.
```

주의:

```text
partition 수보다 consumer 수가 많으면 남는 consumer가 생길 수 있다.
```

예:

```text
partition 3개
consumer 5개
```

이 경우 최대 3개 consumer만 실제로 partition을 읽고, 2개는 놀 수 있습니다.

## 17. 네트워크 지연

Kafka는 broker, producer, consumer가 네트워크로 통신합니다.

네트워크가 느려지면:

```text
producer 전송 지연
consumer 처리 지연
replica 동기화 지연
request latency 증가
consumer lag 증가
ISR 축소 가능
```

예:

```text
broker-3 네트워크가 느려짐
broker-3이 leader 복제를 따라가지 못함
ISR에서 제외됨
under replicated partition 발생
```

## 18. 전체 흐름 예시

장애 상황 예시:

```text
1. 로그 유입량이 갑자기 증가한다.
2. message in rate가 message out rate보다 커진다.
3. consumer lag가 증가한다.
4. Logstash 처리량이 부족하다고 판단한다.
5. Logstash를 스케일 아웃한다.
6. consumer group rebalance가 발생한다.
7. partition이 여러 Logstash에 나눠 배정된다.
8. message out rate가 증가한다.
9. consumer lag가 줄어든다.
```

broker 장애 예시:

```text
1. broker-1이 장애로 죽는다.
2. broker-1이 leader였던 partition에 문제가 생긴다.
3. controller가 ISR 중 하나를 새 leader로 선출한다.
4. producer와 consumer는 새 leader로 통신한다.
5. follower replica가 부족하면 under replicated partition이 발생한다.
```

## 19. 우리 프로젝트와 연결하기

현재 프로젝트는 학습용 단일 Kafka broker입니다.

현재 구성:

```text
broker 1대
partition 1
replication factor 1
Logstash consumer 1개
```

그래서 아직 다음 개념은 실제로 크게 드러나지 않습니다.

```text
leader/follower replica
ISR
replication
rebalance
under replicated partition
```

하지만 나중에 실무형 확장 구성을 만들면 아래처럼 학습할 수 있습니다.

```text
Kafka broker 3대
partition 3개 이상
replication factor 3
Logstash consumer 여러 개
consumer lag 모니터링
broker 장애 테스트
```

## 20. 한 줄 요약

Kafka가 어려운 이유는 단순 메시지 큐가 아니라 분산 시스템이기 때문입니다.

핵심은 다음을 이해하는 것입니다.

```text
데이터가 어디에 저장되는가?
누가 leader인가?
복제본은 잘 따라오고 있는가?
consumer가 밀리고 있는가?
broker 장애가 나면 누가 대신 처리하는가?
```
