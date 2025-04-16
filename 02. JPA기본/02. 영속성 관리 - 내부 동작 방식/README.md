## 영속성 관리 - 내부 동작 방식

### 1. 영속성 컨텍스트
___
- JPA의 핵심 2가지 : 객체와 관계형 DB 매핑, 영속성 컨텍스트


#### 영속성 컨텍스트
- 엔티티를 영구 저장하는 환경
- EntityManager.persist(entity); => persist 는 DB에 저장하는게 아닌 영속성 컨텍스트에 저장
- 엔티티 매니저를 통해서 영속성 컨텍스트에 접근

#### 엔티티의 생명주기
- 비영속(new/transient) : 영속성 컨텍스트와 전혀 관계가 없는 새로운 상태
```java
//객체를 생성한 상태(비영속)
Member member = new Member();
member.setId("member1");
member.setUsername("회원1");
```

- 영속(managed) : 영속성 컨텍스트에 관리되는 상태
```java
//객체를 저장한 상태(영속), db에 저장 되지않고 영속성 컨텍스트에 올라감.
em.persist(member);
```
- 준영속(detached) : 영속성 컨텍스트에 저장되었다가 분리된 상태
```java
//회원 엔티티를 영속성 컨텍스트에서 분리, 준영속 상태 
em.detach(member);  
```

- 삭제(remove) : 삭제된 상태
```java
//객체를 삭제한 상태(삭제)
em.remove(member);
```

#### 영속성 컨텍스트의 이점
- 1차 캐시
- 동일성(identity) 보장
- 트랜잭션을 지원하는 쓰기 지연(transactional write-behind)
- 변경 감지(Dirty Checking)
- 지연 로딩(Lazy Loading)

#### 1차 캐시
```java
Member member = new Member();
member.setId(101L);
member.setName("HelloJPA");

em.persist(member); // 1차 캐시에 저장
Member findMember1 = em.find(Member.class, 101L);   // 1차 캐시에서 조회 , select 문 X
// 같은것을 두번 조회하면 select 문을 쓰지않고, 1차캐시에서 조회해서 select문이 안찍힌다.( 같은 트랜잭션 안에서만)
Member findMember2 = em.find(Member.class, 101L);   
```
- 조회 시 우선 영속성 컨텍스트에서 1차 캐시에서 찾는다. => 1차 캐시에 없으면 DB에서 조회(1차 캐시에 저장 후 반환)
- 한 트랜잭션안에서만 효과가 있기 때문에 성능에 엄청 이점이 없다.

#### 동일성 보장
```java
Member findMember1 = em.find(Member.class, 101L);
Member findMember2 = em.find(Member.class, 101L);
System.out.println("result = " + (findMember1 == findMember2)); // true 
```

#### 쓰기 지연
```java
EntityManager em = emf.createEntityManager();
EntityTransaction transaction = em.getTransaction();
transaction.begin(); // [트랜잭션] 시작

em.persist(memberA);
em.persist(memberB);
//여기까지 INSERT SQL을 데이터베이스에 보내지 않는다.

transaction.commit(); // [트랜잭션] 커밋
```
- JDBC 배치처럼 모아뒀다가 commit 되는 순간 한번에 insert SQL을 보낸다.
- persistence.xml 에 <property name="hibernate.jdbc.batch_size" value="10"/> 에 batch size 수정 가능
- 

#### 엔티티 수정(변경 감지)
```java
Member member = em.find(Member.class, 150L);
member.setName("ZZZZZ");    // 데이터 수정
//em.persist(member); // 필요 X
transaction.commit(); // [트랜잭션] 커밋
```
- em.persist나 update 같은 코드를 따로 적어 주지 않아도 update가 된다.
- 커밋을 하면 flush() 호출 -> entity와 스냅샷을 비교(캐시에는 id, entity, 스냅샷이 있다.)  
-> 쓰기 지연 저장소에 update sql 생성 -> commit (DB반영)

#### 엔티티 삭제
```java
//삭제 대상 엔티티 조회
Member memberA = em.find(Member.class, “memberA");
em.remove(memberA); //엔티티 삭제
```
