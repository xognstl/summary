### 3. 좋아요 수 설계 - 테이블 정의 
___
#### 좋아요 수 설계
```text
좋아요 수에서는 전체 개수를 실시간으로 빠르게 보여줘야 한다.
조회 시점에 전체 개수를 실시간 조회하는게 큰 비용이 든다면, 좋아요가 생성/삭제될 때마다 미리 좋아요 수를 갱신해 두는 방법이 있다.
좋아요(article_like) 테이블의 게시글 별 데이터 개수를 미리 하나의 데이터로 비정규화해두는 것이다.

좋아요 수는 비교적 쓰기 트래픽이 작고, 데이터 일관성이 중요하다.
트래픽이 작고, 데이터 일관성이 중요하면 트랜잭션을 활용할 수 있다.
좋아요 테이블의 데이터 생성/삭제와 좋아요 수 갱신을 하나의 트랜잭션으로 묶는 것이다

게시글 테이블에 좋아요 수 컬럼을 추가하고, 좋아요가 생성/삭제될 때마다 좋아요 수를 갱신한다.
좋아요 수는 게시글과 1:1 관계의 데이터이기 때문에, 게시글 테이블에 비정규화 한다고 해서 어색함은 없어 보인다.
하지만 Record Lock , 분산 트랜잭션 같은 제약이 생길 수 있다.
```

<br>

#### Record Lock
- Record : 테이블의 행 데이터 
- Lock : 여러 프로세스 또는 스레드가 자원에 동시에 접근하는 경쟁 상태를 방지하기 위해 제한을 거는 것
- Record Lock : 동일한 레코드를 동시에 조회 또는 수정할 때 데이터의 무결성 보장, 경쟁 상태 방지


- record lock 테스트
```text
- test 용 테이블 생성
create table lock_test (
    id bigint not null primary key,
    content varchar(100) not null
);
insert into lock_test values(1234, 'test'); => 테스트 데이터 삽입

- transaction 시작 , 트랜잭션 시작하고 테스트 데이터 업데이트 , commit 하지 않고 트랜잭션을 유지 
start transaction;
update lock_test set content='test2' where id=1234; 

- select * from performance_schema.data_locks; 을 이용하여 lock을 조회할 수 있다.
Exclusive Lock(=X Lock)이 걸린 것을 확인할 수 있다.  
LOCK_TYPE=RECORD, LOCK_MODE=X, LOCK_DATA=1234

- 새로운 터미널에서 select * from lock_test where id=1234; 레코드 조회시 content는 변하지 않았다.(commit 안되서 기존데이터 조회)

- 트랜잭션2에서 레코드 수정
start transaction;
update lock_test set content='test2' where id=1234; => 수행 시 commit, rollback 이 안되고 그대로 유지된다.(멈춰있다?)
mysql> update lock_test set content='test2' where id=1234;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

=> 트랜잭션 1에서는 update 구문이 즉시 처리되었는데, 트랜잭션 2에서는 한참을 기다려도 안되더니 타임 아웃으로 종료되었다.
트랜잭션 1에서 잡힌 Exclusive Lock에 의해, 트랜잭션 2는 Lock이 해제될 때까지 기다려야 했던 것이다.
=> 트랜잭션2에서 업데이트 시 유지되고 있을 때 트랜잭션1 이 commit 되면 트랜잭션2에서도  수행된다.

락을 조회하면 트랜잭션ID 가 트랜잭션1이 종료되면서 Exclusive Lock을 해제하고, 트랜잭션2에서 Exclusive Lock을 획득했다.

락을 오래 점유하면 발생하는 문제는 치명적이고 다양하다. => 리소스 고갈 등으로 장애 발생가능
```

<br>

```text
Record Lock으로 인해 게시글 테이블에 좋아요 수 컬럼을 비정규화하는 것은 제약이 생길 수도 있다
게시글과 좋아요 수의 변경은 Lifecycle이 다르다.
- 게시글 : 작성한 사용자가 쓰기 작업을 수행한다. 트래픽이 적다.
- 좋아요수 : 게시글을 조회한 사용자들이 쓰기 작업을 수행한다. 트래픽이 많다.

서로 다른 주체에 의해서 레코드에 락이 잡힐 수 있다.
게시글 쓰기와 좋아요 수 쓰기는 사용자 입장에서 독립적으로 수행되는 기능이다. => 각 기능이 서로 영향을 끼칠 수 있다.
타임아웃을 짧게 가져가거나 요청량에 제한을 둔다면? => 독립적인 두 기능이 서로에 의해 실패할 수 있다. 좋아요 때문에 작성자가 쓰기가 실패 하는 경우 발생

게시글과 좋아요 수는 1:1 관계지만 독립적인 테이블로 분리하여야 한다.

** 좋아요 수 테이블의 관리 위치
- MSA 를 사용하기에 서비스 별로 독립적인 DB 구성, 샤딩이 고려된 분산 DB 사용 => 분산 시스템을 구성
- 좋아요, 좋아요 수 데이터의 일관성을 위해 트랜잭션 사용
분산된 시스템에서 트랜잭션을 지원하려면 분산 트랜잭션이 필요 => but 상대적으로 느리고 복잡

좋아요 수 테이블과 좋아요 테이블이 물리적으로 다른 샤드에 있을 수 있기 때문에 
좋아요 수 테이블의 Shard Key 는 좋아요 테이블과 같이 article_id 로 한다. 
```

<br>

### 4. 좋아요 수 설계 - 동시성 문제 
___
```text
- 높은 쓰기 트래픽이 들어올 수 있는 상황
=> 좋아요 수 증가 /감소 는 어떻게 해야할까? : 트랜잭션 사용? , 현재 좋아요 수를 조회하고, 좋아요 생성/삭제에 따라 갱신?
하지만 동시성 문제가 발생할 수 있는 상황

* 좋아요 생성 요청이 동시에 2개가 들어올때 2개의 트랜잭션이 처리되어야 하는 상황
- 현재 좋아요 수 조회 및 갱신
start transaction1 -> 좋아요 생성(t1) -> start transaction2 -> 좋아요 생성(t2) -> 좋아요 수 조회 = 0(t1) -> 좋아요 수 조회 = 0(t2)
-> 좋아요 수 갱신 = 1(t1) -> commit(t1) -> 좋아요 수 갱신 = 1(t2) -> commit(t2)
*** 각 트랜잭션은 정상적으로 요청을 처리했다고 판단하여 commit => 동시요청으로 인해 증가 처리가 누락될 수 있는 상황

트랜잭션을 사용해도 동시성 문제로 데이터의 일관성은 깨질수 있다. => Lock 을 이용하여 동시성 문제 해결

- 현재 좋아요 수 조회 및 갱신 ( Lock 이용)
start transaction1 -> 좋아요 생성(t1) -> start transaction2 -> 좋아요 생성(t2)
-> 좋아요 수 증가(좋아요 수 = 좋아요 수 + 1)(t1) , 이 때 트랜잭션 1은 해당 레코드에 락을 점유
-> 좋아요 수 증가(좋아요 수 = 좋아요 수 + 1)(t2) , 이미 transaction1 에서 record lock 을 점유한 상태
-> wait(t2) -> commit(t1) , record lock 해제 
-> 좋아요 수 증가 성공(t2) transaction2 에서 record lock 점유 , 대기중이던 좋아요 수 증가 요청 성공 -> commit(t2)
*** 트랜잭션 1에서 점유한 Record Lock에 의해 처리가 지연되고 있다. 이렇게 리소스를 점유하고 있는 블로킹 작업은 장애가 발생할 여지가 있다.
동일한 데이터를 수정하기 때문에, 동시성 문제를 제어하기 위한 방법이 필요
```

<br>

- 동시 쓰기 요청이 들어올 때, 데이터 유실 또는 장애 없이 처리하기 위한 방법
    - 비관적 락(Pessimistic Lock)
    - 낙관적 락(Optimistic Lock)
    - 비동기 순차 처리

<br>

#### 비관적 락(Pessimistic Lock)
- 데이터 접근 시에 항상 충돌이 발생할 가능성이 있다고 가정(Record Lock도 비관적 락)
- 데이터를 보호하기 위해 항상 락을 걸어 다른 트랜잭션 접근을 방지
  - 다른 트랜잭션은 락이 해제되기까지 대기
  - 락을 오래 점유하고 있으면, 성능 저하 또는 deadlock 등으로 인한 장애 문제

```text
- 비관적 락 방법1 : DB에 저장된 데이터 기준으로 update 문 수행
1. transaction start;
2. insert into article_like values({article_like_id}, {article_id}, {user_id}, {created_at}); : 좋아요 데이터 삽입
3. update article_like_count set like_count = like_count + 1 where article_id = {article_id}; : 좋아요 수 데이터 갱신, Pessimistic Lock 점유
4. commit; : Pessimistic Lock 해제

- 비관적 락 방법2 : 트랜잭션에 조회된 데이터 기준으로 UPDATE 문을 수행, for update 구문으로 조회 결과에 대해 락을 점유하겠다고 명시
1. transaction start;
2. insert into article_like values({article_like_id}, {article_id}, {user_id}, {created_at}); : 좋아요 데이터 삽입
3. select * from article_like_count where article_id = {article_id} for update;
    • for update 구문으로 데이터 조회
    • 조회된 데이터에 대해서 Pessimistic Lock 점유(이 시점부터 다른 Lock은 점유될 수 없다.)
    • 애플리케이션에서 JPA를 사용하는 경우, 객체(엔티티)로 조회할 수 있다.
4. update article_like_count set like_count = {updated_like_count} where article_id = {article_id};
    • 좋아요 수 데이터 갱신
    • 조회된 데이터를 기반으로 새로운 좋아요 수를 만들어준다. (조회 시점부터 Lock을 점유하고 있기 때문에 가능)
    • Client(애플리케이션)에서 JPA를 사용하는 경우, 엔티티로 위 과정을 수행 가능
5. commit;
• Pessimistic Lock 해제

- 락 점유
방법 1 : UPDATE 문 수행하는 시점에 락을 점유한다.   => 상대적으로 락 점유 시간이 짧다.
방법 2 : 데이터 조회 시점부터 락을 점유한다. => 상대적으로 락 점유 시간이 길다. 
데이터를 조회한 뒤 중간 과정을 수행해야 하기 때문에, 락 해제가 지연될 수 있다.

- 애플리케이션 개발
방법 1 : 데이터베이스의 현재 저장된 데이터 기준으로 증감 처리하기 때문에 SQL문을 직접 전송
방법 2 : : JPA를 사용하는 경우, 엔티티를 이용가능
```

<br>

#### 낙관적 락(Optimistic Lock)
- 데이터 접근 시에 항상 충돌이 발생할 가능성이 없다고 가정
- 데이터의 변경 여부를 확인하여 충돌을 처리
  - 데이터가 다른 트랜잭션에 의해 수정되었는지 확인 => 수정된 내역이 있으면 후처리(rollback 또는 재처리 등)
```text
- 데이터 변경 여부 확인 : 각 테이블은 version 컬럼으로 데이터의 변경 여부를 추적
- 충돌 확인
    1. 각 트랜잭션에서 version을 함께 조회한다.
    2. 레코드를 업데이트 한다. => 이 때, WHERE 조건에 조회된 version을 넣고, version은 증가시킨다.
    3. 충돌을 확인
    • 데이터 변경이 성공 했다면, 충돌 X, 데이터 변경이 실패 했다면, 충돌 O
    • 다른 트랜잭션에서 version을 이미 증가 시켰음을 의미하므로, 충돌이 생긴 것이다.
    
DB 에 좋아요 데이터 (like_count=1, version=1)
-> start transaction, 데이터 조회(like_count=1, version=1) (t1,t2) 트랜잭션1, 2에서 좋아요 생성 및 count 조회
-> 좋아요 수/버전 갱신 (like_count=2, version=2  => update … where version=1)(t1) : 조회된 데이터 기반으로 +1, 버전 증가
-> DB 데이터 반영 (like_count=2, version=2)
-> 좋아요 수/버전 갱신 (like_count=2, version=2  => update … where version=1)(t2) : version=1 데이터가 없으므로 트랜잭션2는 갱신 실패
-> 트랜잭션 2는 데이터 변경에 대해 충돌이 발생했다고 판단하고 rollback을 수행
-> 트랜잭션 1은 정상 commit, 트랜잭션 2는 충돌 감지 rollback
트랜잭션 2는 update 문 수행 실패로 충돌을 감지, 즉시 rollback
요청에는 실패하였지만, 데이터 일관성은 깨지지 않았다.

*** 그러므로 충돌을 감지하고 후처리를 위한 추가 작업이 필요
- 충돌 발생 시 commit or rollback or 재시도, 애플리케이션에서 직접 구현해야한다.
()
```

<br>

#### 비동기 순차 처리
- 모든 상황을 실시간으로 처리하고 즉시 응답해줄 필요는 없다는 관점
  - 요청을 대기열에 저장해두고, 이후에 비동기로 순차적으로 처리할 수도 있다.
  - 게시글마다 1개의 스레드에서 순차적으로 처리하면, 동시성 문제도 사라진다.
  - 락으로 인한 지연이나 실패 케이스가 최소화된다. 즉시 처리되지 않기 때문에 사용자 입장에서는 지연될 수 있다.
- 하지만 큰 비용이 든다
  - 비동기 처리를 위한 시스템 구축 비용
  - 실시간으로 결과 응답이 안되기 때문에 클라이언트 측 추가 처리 필요
  - 이미 처리된 것처럼 보이게 하고, 실패 시에 알림을 준다든지?
  - 서비스 정책으로 납득이 되어야 한다.
  - 데이터의 일관성 관리를 위한 비용
  - 대기열에서 중복/누락 없이 반드시 1회 실행 보장되기 위한 시스템 구축이 필요하다.

<br>

- 비관적 락 vs 낙관적 락
  - 비관적 락은 락을 명시적으로 잡아야하지만 게시글 단위로 좋아요가 처리되기 때문에, 좋아요 쓰기 트래픽에서 단일 레코드에 대한 잠깐의 락은 문제가 안될것 같다.
  - 낙관적 락은 락을 잡지 않기 떄문에 지연은 낮을 수 있지만, 충돌 감지 시 애플리케이션 추가적인 처리가 필요
- update, select for update + update, version 세가지 방법 다 구현 예정


-  데이터베이스에 저장된 데이터 기준(비관적 락 1번)
```sql
update article_like_count set like_count = like_count + 1 where article_id = {article_id}
```

 
- 트랜잭션에 조회된 데이터 기준(비관적 락 2번)
  - for update 구문으로 데이터를 조회한 뒤, 조회된 데이터 기반으로 좋아요 수를 갱신해준다.
```sql
select * from article_like_count where article_id = {article_id} for update;
update article_like_count set like_count = {updated_like_count} where article_id = {article_id}
```

 
- 낙관적 락
- version 컬럼을 추가하고, 애플리케이션에서 낙관적 락에 의한 충돌 처리 작업이 필요하다.
  - 단순히 rollback 처리 


- 테이블 설계
```sql
create table article_like_count (
    article_id bigint not null primary key,
    like_count bigint not null,
    version bigint not null
);

```

<br>

### 5. 좋아요 수 구현
___
- Entity 생성
```java
@Table(name = "article_like_count")
@Getter
@ToString
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class ArticleLikeCount {
    @Id
    private Long articleId;  // shardKey
    private Long likeCount;
    @Version
    private Long version;

    public static ArticleLikeCount init(Long articleId, Long likeCount) { // 데이터가 없을 때 초기화 시켜주는 메서드
        ArticleLikeCount articleLikeCount = new ArticleLikeCount();
        articleLikeCount.articleId = articleId;
        articleLikeCount.likeCount = likeCount;
        articleLikeCount.version = 0L;
        return articleLikeCount;
    }
    
    public void increase() {
        this.likeCount++;
    }
    
    public void decrease() {
        this.likeCount--;
    }
}
```
- repository
```java
@Repository
public interface ArticleLikeCountRepository extends JpaRepository<ArticleLikeCount, Long> {

    // select ... for update -> @Lock 을 달면 자동으로 해준다. 
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    Optional<ArticleLikeCount> findLockedByArticleId(Long articleId);

    // 비관적 락 1번
    @Query(
            value = "update article_like_count set like_count = like_count + 1 where article_id = :articleId",
            nativeQuery = true
    )
    @Modifying
    int increase(@Param("articleId") Long articleId);

    @Query(
            value = "update article_like_count set like_count = like_count - 1 where article_id = :articleId",
            nativeQuery = true
    )
    @Modifying
    int decrease(@Param("articleId") Long articleId);
}
```
- service
```java
@Service
@RequiredArgsConstructor
public class ArticleLikeService {
    private final Snowflake snowflake = new Snowflake();
    private final ArticleLikeRepository articleLikeRepository;
    private final ArticleLikeCountRepository articleLikeCountRepository;

    public ArticleLikeResponse read(Long articleId, Long userId) {  // 사용자가 좋아요 했는지 유무
        return articleLikeRepository.findByArticleIdAndUserId(articleId, userId)
                .map(ArticleLikeResponse::from)
                .orElseThrow();
    }

    // update 구문
    @Transactional
    public void likePessimisticLock1(Long articleId, Long userId) { // 좋아요 수행
        articleLikeRepository.save(
                ArticleLike.create(
                        snowflake.nextId(),
                        articleId,
                        userId
                )
        ); // unique index 떄문에 한건만 데이터 생성

        int result = articleLikeCountRepository.increase(articleId);
        if(result == 0) {   // update 데이터가 없을 때
            // 최초 요청 시에 update 되는 레코드가 없으므로 , 1로 초기화
            // 트래픽이 순식간에 몰릴 수 있는 상황에서는 유실될 수 있으므로, 게시글 생성 시점에 미리 0으로 초기화 전략도 가능
            articleLikeCountRepository.save(
                    ArticleLikeCount.init(articleId, 1L)
            );
        }

    }

    @Transactional
    public void unlikePessimisticLock1(Long articleId, Long userId) {   // 좋아요 취소
        articleLikeRepository.findByArticleIdAndUserId(articleId, userId)
                .ifPresent(articleLike -> {
                            articleLikeRepository.delete(articleLike);
                            articleLikeCountRepository.decrease(articleId);
                });
    }

    // select ... for update + update
    @Transactional
    public void likePessimisticLock2(Long articleId, Long userId) { // 좋아요 수행
        articleLikeRepository.save(
                ArticleLike.create(
                        snowflake.nextId(),
                        articleId,
                        userId
                )
        );
        ArticleLikeCount articleLikeCount = articleLikeCountRepository.findLockedByArticleId(articleId)
                .orElseGet(() -> ArticleLikeCount.init(articleId, 0L));// 데이터가 없으면 0으로
        articleLikeCount.increase();
        articleLikeCountRepository.save(articleLikeCount);
    }

    @Transactional
    public void unlikePessimisticLock2(Long articleId, Long userId) {   // 좋아요 취소
        articleLikeRepository.findByArticleIdAndUserId(articleId, userId)
                .ifPresent(articleLike -> {
                    articleLikeRepository.delete(articleLike);
                    ArticleLikeCount articleLikeCount = articleLikeCountRepository.findLockedByArticleId(articleId).orElseThrow();
                    articleLikeCount.decrease();
                });
    }

    // 낙관적 락
    @Transactional
    public void likeOptimisticLock(Long articleId, Long userId) { // 좋아요 수행
        articleLikeRepository.save(
                ArticleLike.create(
                        snowflake.nextId(),
                        articleId,
                        userId
                )
        );

        ArticleLikeCount articleLikeCount = articleLikeCountRepository.findById(articleId).orElseGet(() -> ArticleLikeCount.init(articleId, 0L));
        articleLikeCount.increase();
        articleLikeCountRepository.save(articleLikeCount);
    }

    @Transactional
    public void unlikeOptimisticLock(Long articleId, Long userId) {   // 좋아요 취소
        articleLikeRepository.findByArticleIdAndUserId(articleId, userId)
                .ifPresent(articleLike -> {
                    articleLikeRepository.delete(articleLike);
                    ArticleLikeCount articleLikeCount = articleLikeCountRepository.findById(articleId).orElseThrow();
                    articleLikeCount.decrease();
                });
    }

    public Long Count(Long articleId) {
        return articleLikeCountRepository.findById(articleId)
                .map(ArticleLikeCount::getLikeCount)
                .orElse(0L);
    }
}
```
- controller
```java
@RestController
@RequiredArgsConstructor
public class ArticleLikeController {
    private final ArticleLikeService articleLikeService;

    // 조회
    @GetMapping("/v1/article-likes/articles/{articleId}/users/{userId}")
    public ArticleLikeResponse read(
            @PathVariable("articleId") Long articleId,
            @PathVariable("userId") Long userId
    ) {
        return articleLikeService.read(articleId, userId);
    }

    // 좋아요 수 
    @GetMapping("/v1/article-likes/articles/{articleId}/count")
    public Long count(
            @PathVariable("articleId") Long articleId
    ) {
        return articleLikeService.count(articleId);
    }
    
    // 좋아요
    @PostMapping("/v1/article-likes/articles/{articleId}/users/{userId}/pessimistic-lock-1")
    public void likePessimisticLock1(
            @PathVariable("articleId") Long articleId,
            @PathVariable("userId") Long userId
    ) {
        articleLikeService.likePessimisticLock1(articleId, userId);
    }

    // 좋아요 취소
    @DeleteMapping("/v1/article-likes/articles/{articleId}/users/{userId}/pessimistic-lock-1")
    public void unlikePessimisticLock1(
            @PathVariable("articleId") Long articleId,
            @PathVariable("userId") Long userId
    ) {
        articleLikeService.unlikePessimisticLock1(articleId, userId);
    }

    // 좋아요
    @PostMapping("/v1/article-likes/articles/{articleId}/users/{userId}/pessimistic-lock-2")
    public void likePessimisticLock2(
            @PathVariable("articleId") Long articleId,
            @PathVariable("userId") Long userId
    ) {
        articleLikeService.likePessimisticLock2(articleId, userId);
    }

    // 좋아요 취소
    @DeleteMapping("/v1/article-likes/articles/{articleId}/users/{userId}/pessimistic-lock-2")
    public void unlikePessimisticLock2(
            @PathVariable("articleId") Long articleId,
            @PathVariable("userId") Long userId
    ) {
        articleLikeService.unlikePessimisticLock2(articleId, userId);
    }

    // 좋아요
    @PostMapping("/v1/article-likes/articles/{articleId}/users/{userId}/optimistic-lock")
    public void likeOptimisticLock(
            @PathVariable("articleId") Long articleId,
            @PathVariable("userId") Long userId
    ) {
        articleLikeService.likeOptimisticLock(articleId, userId);
    }

    // 좋아요 취소
    @DeleteMapping("/v1/article-likes/articles/{articleId}/users/{userId}/optimistic-lock")
    public void unlikeOptimisticLock(
            @PathVariable("articleId") Long articleId,
            @PathVariable("userId") Long userId
    ) {
        articleLikeService.unlikeOptimisticLock(articleId, userId);
    }
}
```
- test
```java
public class LikeApiTest {
    RestClient restClient = RestClient.create("http://localhost:9002");

    @Test
    void likeAndUnlikeTest() {
        Long articleId = 9999L;

        like(articleId, 1L, "pessimistic-lock-1");
        like(articleId, 2L, "pessimistic-lock-1");
        like(articleId, 3L, "pessimistic-lock-1");

        ArticleLikeResponse response1 = read(articleId, 1L);
        ArticleLikeResponse response2 = read(articleId, 2L);
        ArticleLikeResponse response3 = read(articleId, 3L);
        System.out.println("response1 = " + response1);
        System.out.println("response2 = " + response2);
        System.out.println("response3 = " + response3);

        unlike(articleId, 1L);
        unlike(articleId, 2L);
        unlike(articleId, 3L);
    }

    void like(Long articleId, Long userId, String lockType) {
        restClient.post()
                .uri("/v1/article-likes/articles/{articleId}/users/{userId}/" + lockType, articleId, userId)
                .retrieve();
    }

    void unlike(Long articleId, Long userId) {
        restClient.delete()
                .uri("/v1/article-likes/articles/{articleId}/users/{userId}", articleId, userId)
                .retrieve();
    }

    ArticleLikeResponse read(Long articleId, Long userId) {
        return restClient.get()
                .uri("/v1/article-likes/articles/{articleId}/users/{userId}", articleId, userId)
                .retrieve()
                .body(ArticleLikeResponse.class);
    }

    @Test
    void likePerformanceTest() throws InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(100); // 고정된 스레드 풀 100개 생성
        likePerformanceTest(executorService, 1111L, "pessimistic-lock-1");
        likePerformanceTest(executorService, 2222L, "pessimistic-lock-2");
        likePerformanceTest(executorService, 3333L, "optimistic-lock");
    }

    void likePerformanceTest(ExecutorService executorService, Long articleId, String lockType) throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(3000);    //3000번 호출
        System.out.println(lockType + " started");

        like(articleId, 1L, lockType); // 최초 요청시 레코드가 초기화가 안된경우가있어서 한번 호출

        long start = System.nanoTime();
        for (int i = 0; i < 3000; i++) {    // 멀티 스레드로 동시 호출
            long userId = i + 2; // 아이디 중복되면 안됨
            executorService.submit(() -> {
                like(articleId, userId, lockType);
                latch.countDown();
            });
        }
        latch.await();
        long end = System.nanoTime();

        System.out.println("lockType = " + lockType + ", time = " + (end - start) / 1000000 + "ms");
        System.out.println(lockType + " end");

        // 카운트 조회
        Long count = restClient.get()
                .uri("/v1/article-likes/articles/{articleId}/count", articleId)
                .retrieve()
                .body(Long.class);
        System.out.println("count = " + count);
    }
}
```
