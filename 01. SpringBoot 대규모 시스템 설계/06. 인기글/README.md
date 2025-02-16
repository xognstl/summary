## 인기글

### 1. Kafka Cluster
___
- Kafka
  - 분산 이벤트 스트리밍 플랫폼
  - 대규모 데이터를 실시간으로 처리하기 위해 사용
  - 고성능, 확장성, 내구성, 가용성
  - 서비스간 대규모 이벤트를 생산, 소비하며 통신하기 위해 사용

```text
Producer : 데이터 생산자, Consumer : 데이터 소비자

Producer -> Consumer
- API 로 통신하면 Consumer 에서 장애 발생 시 
Producer는 Consumer를 직접 호출하기 때문에 장애가 전파될 수도 있고, Consumer에 요청이 전달되지 않는다면, 데이터 유실 가능성이 있다.
=> 대규모 데이터를 안전하게, 고성능으로 처리하는데 한계

Producer -> Message Queue -> Consumer
- Producer와 Consumer가 직접적으로 결합되지 않기 때문에 장애 전파, 데이터 유실 가능성이 낮아지고, 데이터에 대한 처리가 비동기로 수행될 수도 있다.

Producer, Consumer 가 많아지고, 데이터가 늘어나고, Message Queue 에 장애가 발생하고 하면 단일 Message Queue로는 대규모 데이터 처리가 어려울 수도 있다.
=> Message Broker(메시지 중개자)가 여러 Message Queue를 관리함으로써 대규모 데이터를 병렬로 처리하며, 복잡한 데이터 요구사항을 처리
=> Producer가 생산하는 데이터가 Broker나 Consumer의 처리량을 넘어가면, 리소스 부족으로 인해 장애가 전파될 수도 있고, 데이터가 유실될 수 있다.
=> Consumer가 Message Broker에서 데이터를 push 받는게 아니라, Consumer가 Message Broker에서 데이터를 pull 해온다.
Consumer는 자신의 처리량에 따라서 조절할 수 있다.
즉, Producer가 데이터를 생산(publish)하면, Consumer는 데이터를 구독(subscribe)해서 가져오는 것이다. => pub/sub 패턴
```

- Kafka Broker : Kafka에서 데이터를 중개 및 처리해주는 애플리케이션 실행 단위
    - Producer는 Broker에 데이터를 생산하고, Consumer는 Broker에서 데이터를 소비한다.

- topic : Kafka에서 생산 및 소비되는 데이터를 논리적으로 구분하는 단위
    - Producer는 topic 단위로 이벤트를 생산 및 전송하고, Consumer는 topic 단위로 이벤트를 구독 및 소비

```text
- 처리해야 할 데이터가 많아진다면? 
여러 대의 Kafka Broker를 연결하여 Cluster를 이루게 하고, 처리량을 늘려볼 수 있다.
topic은 논리적인 구분 단위이기 때문에, 여러 Broker에서 병렬 처리 함으로써 처리량을 늘릴 수 있다.

- 이러한 topic은 여러 Broker로 어떻게 분산되어 처리될 수 있는 것일까?
각 topic은 partition 단위로 물리적으로 분산될 수 있다.
topic 1과 topic 2의 partition=3이라면? 각 topic은 3개의 partition으로 분산하여 데이터를 처리한다.
Producer는 topic의 partition 단위로 데이터를 생산하고, Consumer는 topic의 partition 단위로 데이터를 처리

- 3대의 Broker가 있는데, 각 topic의 모든 partition을 1대의 Broker에서 처리할 필요가 있을까?
각 topic의 모든 partition에 대한 부하를 1대의 Broker가 처리할 필요는 없다.
Kafka에서 각 topic의 partition은 여러 Broker에 균등하게 분산될 수 있다.
물리적으로 분산된 여러 개의 partition에 대해 각 Broker가 분산 처리할 수 있는 것이다.
Consumer들은 partition 단위로 구독하여 데이터를 처리할 수 있다.

- 그렇다면, 데이터가 partition 단위로 분산된 상황에, 순서를 고려한 처리가 필요하다면 어떻게 할 수 있을까?
Producer는 topic에 생산되는 이벤트에 대해 직접 partition을 지정할 수도 있고, partition을 지정하지 않는다면 라운드 로빈 방식으로 적절히 분산할 수도 있다.
즉, 순서 보장이 필요한 이벤트들에 대해서는 동일한 partition으로 보내준다.

- 장애 발생 시 
broker 중 하나에 장애가 발생하면 해당 데이터는 유실되고 다음 데이터는 어떻게 처리 해야할까?
- 장애 상황에도 데이터를 보존하고 처리할 방법 => 데이터의 복제본을 관리
replication factor=3 설정을 한다면, 각 partition의 데이터는 3개로 복제된다.
leader에 데이터를 쓰면, follower로 데이터가 복제된다. 각 복제본은 Kafka에서 여러 Broker 간에 균등하게 분산

=> 3개의 broker 중 하나에 장애가 발생해도 데이터는 다른 broker 에 복제되어있다.
leader를 정상 Broker의 follower에서 재선출하면,데이터 생산 및 소비를 계속 처리해나갈 수 있다.

- 하지만, 이러한 시스템을 위한 데이터 복제 과정에는 추가적인 비용이 생긴다. Producer는 모든 복제가 완료될 때까지 기다려야할까?
=> Producer의 acks 설정으로 제어
    - acks=0 : Broker에 데이터 전달되었는지 확인하지 않음. 매우 빠르지만, 데이터 유실 가능성.
    - acks=1 : leader에 전달되면 성공. follower 전달 안되면 장애 시에 유실 가능성 있으나, acks=0보다 안전하다.
    - acks=all : leader와 모든 follower(min.insync.replicas 만큼)에 데이터 기록되면 성공. 가장 안전하지만, 지연될 수 있다.
* min.insync.replicas : 데이터 전송 성공으로 간주하기 위해 최소 몇 개의 ISR이 있어야하는지 설정
* ISR(In-Sync Replicas) : leader의 데이터가 복제본으로 동기화되어 있는 follower들을 의미, acks=all 설정일 때 함께 동작

- Kafka는 순서가 보장된 데이터 로그를 각 topic의 partition 단위로 Broker의 디스크에 저장한다.
그리고 각 데이터는 고유한 offset을 가지고 있다. Consumer는 offset을 기반으로 데이터를 읽어갈 수 있다
Consumer가 (topic 1, partition 1)의 마지막 읽은 offset=4라면? 
=> offset=4까지 데이터를 읽었음을 의미하므로, offset=5부터 새로운 데이터를 읽어나갈 수 있다.
=> 데이터를 잘 관리하고 있다.

- 새로운 Consumer가 다른 목적으로 데이터를 처리해야한다면?
새로운 Consumer는 기존의 Consumer들과 다른 목적을 가지고 병렬로 데이터를 처리해야할 수도 있다.
즉, 모든 Consumer가 전역적으로 동일한 Offset을 사용할 수 없다.
Offset은 Consumer Group 단위로 관리된다.
여러 Consumer가 동일한 Consumer Group이라면, 각 topic의 각 partition에 대해 동일한 offset을 공유한다.
Consumer Group을 달리하면, 별도의 목적으로 동일한 데이터를 병렬로 처리할 수 있는 것이다.

- Broker, Topic, Partition Consumer Group, Offset 등의 정보(메타데이터)는 Zookeeper가 관리 한다.
Zookeeper : Kafka에서 사용되는 메타데이터를 관리
Zookeeper도 고가용성을 위해 여러 대를 연결하여 클러스터를 이룰 수 있다. 하지만 Zookeeper에 의존성이 생기므로 더욱 복잡한 구조가 된다.
=> Kafka 2.8 이후부터 메타데이터 관리에 대해 Kafka Broker 자체적으로 관리할 수 있게 되었다.
KRaft 모드로 Zookeeper 의존성 제거하여, 더 간단한 구조


- article/comment/like/view 서비스는 각각 생산하는 데이터를 토픽으로 구분하여 카프카로 전송
  - 이 때 전송되는 데이터는, 각 서비스에서 생산한 이벤트가 된다. (게시글 생성/삭제 이벤트 등)
- hot article 서비스의 모든 서버 군은 hot article Consumer Group으로 그룹화되고, 필요한 토픽을 구독하여 데이터를 처리
- article read 서비스의 모든 서버 군도 article read Consumer Group으로 그룹화되고, 필요한 토픽을 구독
=> hot article, article read 서비스는 Consumer Group을 달리하기 때문에, Producer가 생산한 이벤트를 목적에 따라 병렬로 처리
```
#### Kafka 주요 기능 정리
- Producer : Kafka로 데이터를 보내는 클라이언트, 데이터를 생산 및 전송, Topic 단위로 데이터 전송
- Consumer : Kakfa에서 데이터를 읽는 클라이언트, 데이터를 소비 및 처리, Topic 단위로 구독하여 데이터 처리
- Broker
  - Kafka에서 데이터를 중개 및 처리해주는 애플리케이션 실행 단위
  - Producer와 Consumer 사이에서 데이터를 주고 받는 역할
- Kafka Cluster
  - 여러 개의 Broker가 모여서 하나의 분산형 시스템을 구성한 것
  - 대규모 데이터에 대해 고성능, 안전성, 확장성, 고가용성 등 지원 => 데이터의 복제, 분산 처리, 장애 복구 등
- Topic : 데이터가 구분되는 논리적인 단위
- Partition : Topic이 분산되는 단위
  - 각 Topic은 여러 개의 Partition으로 분산 저장 및 병렬 처리된다.
  - 각 Partition 내에서 데이터가 순차적으로 기록되므로, Partition 간에는 순서가 보장되지 않는다.
  - Partition은 여러 Broker에 분산되어 Cluster의 확장성을 높인다.
- Offset
  - 각 데이터에 대해 고유한 위치 : 데이터는 각 Topic의 Partition 단위로 순차적으로 기록되고, 기록된 데이터는 offset을 가진다.
  - Consumer Group은 각 그룹이 처리한 Offset을 관리 : 데이터를 어디까지 읽었는지
- Consumer Group
  - Consumer Group은 각 Topic의 Partition 단위로 Offset을 관리
  - Consumer Group 내의 Consumer들은 데이터를 중복해서 읽지 않을 수 있다.
  - Consumer Group 별로 데이터를 병렬로 처리할 수 있다.

<br>

### 2. Kafka 개발 환경 세팅
___
- $ docker run -d --name board-kafka -p 9092:9092 apache/kafka:3.8.0
- $ docker exec --workdir /opt/kafka/bin/ -it board-kafka sh
- Topic 생성(article, comment, like, view)
- $ ./kafka-topics.sh --bootstrap-server localhost:9092 --create --topic board-article --replication-factor 1 --partitions 3
- $ ./kafka-topics.sh --bootstrap-server localhost:9092 --create --topic board-comment --replication-factor 1 --partitions 3
- $ ./kafka-topics.sh --bootstrap-server localhost:9092 --create --topic board-like --replication-factor 1 --partitions 3
- $ ./kafka-topics.sh --bootstrap-server localhost:9092 --create --topic board-view --replication-factor 1 --partitions 3

<br>

### 3. 인기글 Consumer 설계
___
#### 인기글 요구사항
- 일 단위로 상위 10건 인기글 선정
  - 매일 오전 1시 업데이트 , 좋아요 수/댓글 수/조회수 기반 점수 계산 => 점수 = (좋아요 수 * 3) + (댓글 수 * 2) + (조회수 * 1)
- 최근 7일 인기글 내역 조회

<br>

#### 인기글 설계
- 매일 게시글 생성 트래픽은 수백~수천만 건 이상 일때 하루 동안 생성된 모든 게시글에 대해서, 점수를 계산하고, 상위 10건을 찾아야 하는 작업
- 대규모 데이터를 빠르고 효율적으로 처리하기 위한 방법이 필요하다.
```text
- 배치 처리를 고려해볼 수 있다.
    1. 오전 12시가 되자마자, 전날 작성된 게시글을 모두 순회
    2. 각 게시글에 대해서 좋아요 수, 조회수, 댓글 수를 조회
    3. 게시글의 점수를 계산
    4. 모든 게시글에 대해서 상위 10건을 선정

- 대규모 데이터라 시간이 촉박할 수 있다. => 병렬 시스템을 구축할수도 있지만 장비의 리소스도 고려야해야한다.
- 인기글 선정에는 게시글, 좋아요 수, 조회수, 댓글 수 데이터가 필요 => 데이터는 각각의 microservice 에서 가지고 있다.
- 1시간 만에 각 서비스에 많은 데이터 질의가 필요하다. => 인기글 서비스에 의해 타 서비스가 부하가 증가
이러한 문제를 해결하기 위해 스트림 처리 사용
```
- 스트림(Stream)
  - 연속적인 데이터 흐름
  - 실시간으로 발생하는 로그, 센서 감지, 주식 거래 데이터 등과 같이 연속적으로 들어오는 데이터
- 스트림 처리(Stream Processing)
  - 연속적으로 들어오는 실시간 데이터를 처리하는 방식

- 게시글 생성/수정/삭제 이벤트, 댓글 생성/삭제 이벤트, 좋아요 생성/삭제 이벤트, 조회수 집계 이벤트
  - 이벤트를 실시간으로 받아서 처리하면 배치 시스템을 구축하지 않아도 인기글을 구할수 있다.
1. 인기글 선정에 필요한 이벤트를 스트림으로 받는다.
2. 실시간으로 각 게시글의 점수를 계산한다.
3. 실시간으로 상위 10건의 인기글 목록을 만든다.
4. Client는 인기글 목록을 조회한다.
```text
스트림 데이터 소스(Source)는 스트림 데이터를 생산한다. (실시간으로, 연속적으로 생산)
그리고 스트림 데이터는, 데이터 처리기로 전달된다. 데이터 처리기는 스트림 데이터를 처리(가공, 필터링, 집계등)
스트림 데이터의 작업을 수행한 데이터 처리기는, 데이터를 처리한 결과를 생성
이러한 결과는 Client 에 의해 사용된다.

*** 게시글,댓글,좋아요,좋아요 서비스 이벤트 -> 인기글 서비스 -> 인기글 데이터(Redis)
게시글/댓글/좋아요/조회수 서비스는, 데이터 변경 사항에 대해서 실시간으로 스트림 데이터를 생산한다.
스트림 데이터 소스는 게시글 생성/수정/삭제, 댓글 생성/삭제, 좋아요 생성/삭제, 조회수 집계 이벤트를 스트림 데이터로 생산하는 것
인기글 서비스는 이러한 이벤트를 받아서 인기글 선정 작업에 사용
이벤트를 받을 때마다 점수를 계산하고, 상위 10건의 인기글 목록을 생성해 둔다.
인기글은 7일간만 내역을 관리(휘발성), 10건의 데이터만 필요 => Redis에 저장
Redis의 TTL과 Sorted Set(=zset) 사용
Sorted Set(=zset) 자료 구조는 점수를 기반으로 정렬된 집합 데이터를 관리

- 게시글/댓글/좋아요/조회수 서비스는, 실시간 이벤트를 인기글 서비스로 어떻게 전달할 수 있을까?
인기글 서비스로 이벤트를 실시간으로 전달해야하고, 이벤트는 대규모 데이터가 들어올 수 있다.
인기글 서비스는, 자신의 리소스에 알맞게 인기글 선정 작업을 유연하게 처리할 수 있어야한다.
이러한 이벤트 전달 및 처리 과정은 대규모 데이터에 대해서 안전하고 빠르게 처리되어야 한다.
```

<br>

#### 인기글 설계 - API
- 데이터 변경이 생기면 인기글 서비스로 API를 이용한 이벤트 전송
- 간단한 구현 but 타 서비스에 직접적 의존, 시스템 간 결합도 증가
- 인기글은 데이터를 실시간으로 push 받고, 게시글,좋아요,댓글,조회수는 실시간으로 push 한다.
- 장애 전파, 유실등의 위험이 높다.

#### 인기글 설계 - Message Broker
- 데이터 변경이 생기면 메세지 브로커로 이벤트를 전송하고, 인기글 서비스에서 이벤트를 가져와서 처리
- 복잡한 구현, 타서비스에 메시지 브로커를 이용한 간접적 의존성, 시스템 결합도 감소
- 인기글은 메시지 브로커에서 적절하게 이벤트를 가져와서 비동기 처리가능
- 게시글,좋아요,댓글,조회수에서는 메시지 브로커로 이벤트만 전송하면된다.
- 장애전파, 유실등의 위험이 낮다.
- 한번 구축해두면 Consumer Group만 달리하면 각 목적에 따라 병렬 처리 가능

```text
게시글/댓글/좋아요/조회수 서비스는 Producer, 인기글 서비스는 Consumer
Kafka를 사용하여 이러한 이벤트 전송을 중개
- Event Driven Architecture : 이벤트를 주고받으며 마이크로서비스 간에 통신하는 아키텍처

- 인기글 서비스가 인기글 데이터를 만들어내는 과정
2024년 10월 1일일때 10월 1일에 작성된 게시글은, 10월 2일 오전 1시부터 제공해야 할 인기글이 된다.
0월 1일 00시 00분 ~ 10월 2일 00시 00분 사이에 작성된 게시글에 대한 이벤트는,
각 게시글의 점수를 실시간으로 계산하여 Redis(key=20241001)에 삽입

key=20241001은 Sorted Set(zset) 자료구조를 사용
data=게시글 ID, score=게시글 점수 => 이를 이용하여 Sorted Set에 상위 10건의 데이터만 유지할 수 있다.

하루가 지나고, 2024년 10월 2일 00시가 되었다.
이제부터 작성되는 게시글의 점수는, key=20241002의 Sorted Set에 삽입된다.
이 때, 2024년 10월 1일의 인기글은 이미 완성되었다. 더이상 건드릴 필요는 없다.

2024년 10월 2일 오전 1시 에 10월 1일의 인기글은 사용자들에게 제공
오늘 제공해야 하는 인기글을 어제 미리 실시간으로 만들고 있었기 때문에 배치작업 없이 Client가 조회해야하는 데이터만 지정
날짜가 지나면 Client는 key 를 변경하여 데이터를 조회

- 점수 계산 처리 방법
인기글 서비스는, 댓글/좋아요/조회수 이벤트를 개별적으로 받는다.
즉, 댓글 이벤트를 수신했을 때, 현재의 좋아요 수와 조회수는 이벤트 정보에 포함되어 있지 않다.
이러한 다른 데이터에 대해 현재의 상태를 알아야하는 것이다.

또, 점수 계산은, 댓글 수/좋아요 수/조회수로 이루어진다.
이러한 데이터는 각 서비스에서 원본 데이터를 가지고 있다.
하지만 만약, 점수 계산에 댓글의 유니크 작성자 수가 필요하다면?
이러한 데이터는 댓글 서비스에서 가지고 있지도 않고, 제공하지도 않는다.
인기글을 위해 댓글 서비스에서 데이터를 만들어서 제공할 수도 있지만, 인기글 기능을 계속 신경써야 한다.
댓글 서비스가 인기글을 위한 데이터를 생산하고 관리할 책임은 없다.
즉, 이러한 데이터를 생산하고 관리할 책임은 인기글인 것이다.

인기글 서비스가 인기글에 필요한 데이터를 자체적으로 보관한다면?
이벤트로 전달 받은 원본 데이터를 기반으로, 인기글 선정에 필요한 데이터를 자체적으로 가공 및 생산할 수도 있다.
다른 서비스에 의존하지 않고, 독립적으로 인기글 기능을 제공해나갈 수 있는 것이다.
따라서, 점수 계산에 필요한 데이터를 실시간으로 각 서비스에 다시 요청하지 않고, 인기글 서비스가 자체적인 데이터를 가지도록 한다
```
  
