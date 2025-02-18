### 9. 인기글 Producer 설계 - Transactional Messaging
___
- 게시글/댓글/좋아요/조회수 서비스는 Kafka로 이벤트를 전달하는 Producer 역할

#### Transactional Messaging
```text
- Producer는 비즈니스 로직을 수행 (게시글/댓글/좋아요/조회수 서비스의 API)
- Producer는 Kafka로 이벤트 데이터를 전송 (게시글/댓글/좋아요/조회수 서비스의 이벤트)
- Consumer는 Kafka에서 이벤트 데이터를 가져와서 처리
producer -> kafka -> consumer 처리과정에 장애가 발생할 수 있다. 
Kafka는 데이터 유실을 방지 하기 위해 방법들을 제공 -> 장애 발생시 복구된다.
Consumer 에서 이벤트 처리를 정상적으로 처리하고 offset을 commit 하면 데이터의 유실없이 모두 처리된 것

* producer -> Kafka 이벤트 전달과정에서 장애가 발생하면 
producer 는 kafka로 데이터 전송과 무관하게 정상동작해야하지 Producer로 장애가 전파되면 안된다.
게시글이 정상 생성되도 이벤트가 전달이안되서 데이터가 유실되고 각 서비스마다 데이터의 일관성이 깨진다.
Producer에 게시글이 생성됬는데 Consumer는 받지 못하는 상황
==> 비즈니스 로직 수행과 이벤트 전송이 하나의 트랜잭션으로 처리
(비즈니스 로직 수행과 이벤트 전송이 모두 일어나거나 , 모두 일어나지 않거나)
이러한 보장은 실시간 처리 안해도됨, 비즈니스 로직은 우선적으로 처리되도 이벤트는 장애 처리후 뒤늦게 되도된다.
최종적으로 일관성을 유지 할 수 있다.(Eventually Consistency)

* 트랜잭션
Transaction start -> 비즈니스 로직 수행 -> commit/rollback
트랜잭션 범위 내 비즈니스 로직을 처리한 후 이벤트를 Kafka로 전송 추가?
Transaction start -> 비즈니스 로직 수행 -> publishEvent() ->commit/rollback
=> 트랜잭션은 MySQL 단일 DB에 대한 트랜잭션
=> MySQL 과 Kafka 는 서로 다른 시스템

만약 publishEvent() 과정이 처리된 후 트랜잭션을 종료해도 이상은 없지만 여기서 3초동안 장애가 나면
- 트랜잭션을 점유하고 있기 때문에 Kafka의 장애가 서버 애플리케이션 , MySQL로 장애가 전파될 수 있다.
- commit 에 실패 했는데 이벤트전송은 될수있다.

비동기로 처리해도 이벤트 전송이 실패한다고 MySQL의 트랜잭션이 자동으로 롤백 되진 않는다.
롤백을 위한 보상 트랜잭션을 직접 수행할 수도 있겠지만, 이것도 누락될 수 있으니 문제는 더욱 복잡해진다.

* 2개의 다른시스템은 어떻게 단일 트랜잭션으로 묶을까? 분산 시스템 간에도 트랜잭션 관리가 필요
- Distributed Transaction : 분산 시스템에서 트랜잭션을 보장하기 위한 방법
- Transactional Messaging : 메시지 전송과 타 시스템 작업 간의 분산 트랜잭션을 보장하는 방법

```
- Transactional Messaging을 달성하기 위한 분산 트랜잭션
  - Two Phase Commit
  - Transactional Outbox
  - Transaction Log Tailing

<br>

#### Two Phase Commit
```text
- Two Phase Commit
분산 시스템에서 모든 시스템이 하나의 트랜잭션을 수행할 때,
모든 시스템이 성공적으로 작업을 완료하면 트랜잭션 commit
하나라도 실패하면 rollback
 
1. Prepare phase(준비 단계)
- Coordinator는 각 참여자에게 트랜잭션을 커밋할 준비가 되었는지 물어본다
- 각 참여자는 트랜잭션을 커밋할 준비가 되었는지 응답한다.

2. Commit phase(커밋 단계)
- 모든 참여자가 준비 완료 응답을 보내면, Coordinator는 모든 참여자에게 트랜잭션 커밋을 요청한다.
- 모든 참여자는 트랜잭션을 커밋한다.

- 모든 참여자의 응답을 기다려야 하기 때문에 지연이 길어질 수 있다.
- Coordinator 또는 참여자 장애가 발생하면, 참여자들은 현재 상태를 모른 채 대기해야 할 수도 있다.
=> 트랜잭션 복구가 복잡해질 수 있다.

* 성능 및 오류 처리의 복잡성 문제가 여전히 남아있다.
* Kafka와 MySQL은 자체적으로 이러한 방식의 트랜잭션 통합을 지원 X
```

<br>

#### Transactional Outbox
```text
이벤트 전송 작업을 일반적인 데이터베이스 트랜잭션에 포함시킬 수는 없지만
이벤트 전송 정보를 데이터베이스 트랜잭션에 포함하여 기록할 수는 있다.
=> 트랜잭션을 지원하는 DB에 Outbox 테이블을 생성하고 서비스 로직 수행과 Outbox 테이블 이벤트 메시지 기록을 단일 트랜잭션으로 묶는다.

1. 비즈니스 로직 수행 및 Outbox 테이블 이벤트 기록
  1. Transaction start
  2. 비즈니스 로직 수행
  3. Outbox 테이블에 전송 할 이벤트 데이터 기록
  4. commit or abort
2. Outbox 테이블을 이용한 이벤트 전송 처리
  1. Outbox 테이블 미전송 이벤트 조회
  2. 이벤트 전송
  3. Outbox 테이블 전송 완료 처리

Client 요청에 의해, 단일 트랜잭션 내에서 Data Table의 상태 변경과 Outbox Table에 이벤트 삽입이 처리
- Message Relay는 Outbox Table에서 미전송 이벤트를 조회 -> 이벤트를 Message Broker로 전송
    - Message Relay : Message Broker로 이벤트를 전송하는 역할  
Message Relay는 이벤트 전송이 정상적으로 완료되었으면, Outbox Table의 이벤트를 완료 상태로 변경

- 데이터베이스 트랜잭션 커밋이 완료되었다면, Outbox 테이블에 이벤트 정보가 함께 기록되었기 때문에, 이벤트가 유실되지 않는다.
- 추가적인 Outbox 테이블 생성 및 관리가 필요
- Outbox 테이블의 미전송 이벤트를 Message Broker로 전송하는 작업이 필요
```
- Message Broker 로 이벤트 전송하는 작업은
  - Message Relay 를 직접 구축할 수 도 있고 Transaction Log Tailing Pattern을 활용할 수도 있다.

<br>

#### Transaction Log Tailing
```text
- 데이터베이스의 트랜잭션 로그를 추적 및 분석하는 방법
    - 데이터베이스는 각 트랜잭션의 변경 사항을 로그로 기록(MySQL binlog, PostgreSQL WAL, SQL Server Transaction Log 등)
- 이러한 로그를 읽어서 Message Broker에 이벤트를 전송
    - CDC(Change Data Capture) 기술을 활용하여 데이터의 변경 사항을 다른 시스템에 전송
    - 변경 데이터 캡처(CDC) : 데이터 변경 사항을 추적하는 기술
    
Client 요청에 의해, 단일 트랜잭션 내에서 Data Table의 상태 변경과 Outbox Table에 이벤트 삽입이 처리
-> 데이터베이스에 의해 트랜잭션 변경 사항이 로그로 기록
-> Transaction Log Miner는 Transaction Log를 조회
-> Transaction Log Miner는 Message Broker로 이벤트를 전송
Data Table의 변경 사항만 추적해도 충분하다면, Outbox Table은 없앨 수도 있다.

데이터베이스에 저장된 트랜잭션 로그를 기반으로, Message Broker로의 이벤트 전송 작업을 구축하기 위한 방법으로 활용될 수 있다.
Data Table을 직접 추적하면, Outbox Table은 미사용 할 수도있다.
트랜잭션 로그를 추적하기 위해 CDC 기술을 활용해야 한다
```
- Two Phase Commit :지연, 성능 문제, 오류 처리 및 구현의 복잡성, Kafka와 MySQL 통합의 어려움
- Transaction Log Tailing 을 사용하면 Outbox Table이 필요하지 않지만   
Data Table은 메시지 정보가 데이터베이스 변경 사항에 종속된 구조로 한정 된다.
- Outbox Table을 활용하면 이벤트 정보를 더욱 구체적이고 명확하게 정의

<br>

- Transactional Outbox
  - 데이터의 변경 사항과 Outbox Table로의 이벤트 데이터 기록을 MySQL의 단일 트랜잭션으로 처리
  - 이벤트 전송이 필요한 Article, Comment, Like, View 서비스는 트랜잭션을 지원하는  
MySQL을 사용하고 있기 때문에, Outbox 테이블과 트랜잭션을 통합하여 구현할 수 있다
  - Message Broker로의 전송은 스프링부트에서 직접 개발하고 처리
  
<br>

#### Transactional Outbox 설계
```text
- shard_key의 존재
    - 트랜잭션은 각 샤드에서 단일 트랜잭션으로 빠르고 안전하게 수행될 수 있다.
    - 여러 Shard 간에 분산 트랜잭션을 지원하는 데이터베이스도 있으나, 성능이 다소 떨어진다
    - Outbox Table에 이벤트 데이터 기록과 비즈니스 로직에 의한 상태 변경이 동일한 Shard에서 단일 트랜잭션으로 처리

- 게시글 서비스에서 처리되는 상황
게시글 서비스로 게시글 생성/수정/삭제 API가 호출
-> 게시글 서비스는 비즈니스 로직을 수행하면서, Article Table 상태 변경과 Outbox Table 이벤트 기록을 단일 트랜잭션으로 처리
-> Message Relay는 Outbox Table에서 미전송 데이터를 주기적으로 polling하여 조회 -> Kafka로 전송

10초 간격으로 polling 한다해도 지연은 크다. 
게시글 서비스에서 트랜잭션이 commit 되면, Message Relay로 이벤트를 즉시전달 
-> Message Relay는 전달 받은 이벤트를 비동기로 카프카 전송할 수 있다.
** Message Relay는 전송이 실패한 이벤트에 대해서만 Outbox Table에서 polling하면 된다.
실패한 이벤트는 장애 상황에만 발생할 것이고, 정상 상황에서는 10초 정도면 이벤트 전송할 시간으로 충분 했을 것이다.
생성된 지 10초가 지난 이벤트만 polling
이벤트는 중복 처리될 수 있으므로 Consumer 측에서 멱등성을 고려한 개발은 필요하다
전송 완료되었다면, 간단하게 Outbox Table에서 삭제

- scale out 되거나 샤딩이 고려된 분산시스템
Message Relay가 Outbox Table의 미전송 이벤트를 polling 하는 작업은 어떻게 처리??
미전송 이벤트를 polling하는 것은 특정한 샤드 키가 없으므로, 모든 샤드에서 직접 polling해야 한다.
모든 애플리케이션이 동시에 polling 하면, 동일한 이벤트를 중복으로 처리할 수도 있고,
각 애플리케이션마다 모든 샤드를 polling 하면, 처리에 지연이 생길 수 있다.
=> 처리할 샤드를 중복 없이 적절히 분산할 필요가 있다.

각 애플리케이션은 샤드의 일부만 할당 받아서 처리할 수 있도록 해보자.
이러한 할당은 Message Relay 내의 Coordinator가 처리해준다.
Coordinator는 자신을 실행한 애플리케이션의 식별자와 현재 시간으로, 중앙 저장소에 3초 간격으로 ping을 보낸다.
이를 통해 Coordinator는 실행 중인 애플리케이션 목록을 파악하고, 각 애플리케이션에 샤드를 적절히 분산한다.
중앙 저장소는 마지막 ping을 받은지 9초가 지났으면 애플리케이션이 종료되었다고 판단하고 목록에서 제거한다.

중앙 저장소는 Redis의 Sorted Set을 이용, 애플리케이션의 식별자와 마지막 ping 시간을 정렬된 상태로 저장

4개의 샤드가 있다면?
Coordinator는 N개의 애플리케이션에 4개의 샤드를 범위 기반(Range-Based)으로 할당한다.
어, 2개의 애플리케이션이 있다면, 0~1번 샤드와 2~3번 샤드를 각각 polling 한다.
```
- Outbox table 생성
  - article/comment/article_view/article_like DB 모두 테이블 생성
```sql
create table outbox (
    outbox_id bigint not null primary key,
    shard_key bigint not null,
    event_type varchar(100) not null,
    payload varchar(5000) not null,
    created_at datetime not null
);

create index idx_shard_key_created_at on outbox(shard_key asc, created_at asc);
```

<br>
