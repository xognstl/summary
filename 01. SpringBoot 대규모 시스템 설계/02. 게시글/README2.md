### 6. 게시글 테스트 데이터 삽입
___
- 1200만건의 데이터 삽입
```java
@SpringBootTest
public class DataInitializer {

    @PersistenceContext
    EntityManager entityManager;
    @Autowired
    TransactionTemplate transactionTemplate;
    Snowflake snowflake = new Snowflake();
    CountDownLatch latch = new CountDownLatch(EXECUTE_COUNT); // CountDownLatch 을 사용하여 6천번 수행을 기다림

    static final int BULK_INSERT_SIZE = 2000; // bulk 로 한번에 2천건
    static final int EXECUTE_COUNT = 6000;  // 6천번 실행

    @Test
    void initialize() throws InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(10); // 멀티 스레드
        for (int i = 0; i < EXECUTE_COUNT; i++) {
            executorService.submit(() -> {
                insert();
                latch.countDown();  //카운트 다운
                System.out.println("latch.getCount() = " + latch.getCount());
                // log 가 많이 남으므로 show sql : false 로 잠시 변경
            });  // 멀티 스레드 실행
        }
        latch.await();  //카운트가 0이될때 까지 기다린다.
        executorService.shutdown(); // 모든 삽입이 끝나면 shutdown
    }

    // 트랜잭션 템플릿을 통해서 2000개의 쿼리를 수행하면 트랜잭션이 종료될 때 한번에 삽입이 수행
    void insert() {
        transactionTemplate.executeWithoutResult(status -> {
            for (int i = 0; i < BULK_INSERT_SIZE; i++) {
                Article article = Article.create(
                        snowflake.nextId(),
                        "title" + i,
                        "content" + i,
                        1L,
                        1L
                );
                entityManager.persist(article);
            }
        });
    }
}
```

<br>

### 7. 게시글 목록 API - 페이지 번호 - N번 페이지 M개 게시글 - 설계
___
- 모든 데이터는 한번에 보여줄 수 없고, 페이징이 필요하다.
- 서버 애플리케이션 내의 메모리로 디스크에 저장된 모든 데이터를 가져오고, 특정 페이지만 추출하는 것은 비효율적
  - 디스크 접근은 메모리 보다 느리다.(디스크 I/O 비용), 메모리 용량 초과시 Out of Memory 발생
- 데이터베이스에서 특정 페이지의 데이터만 바로 추출하는 방법 필요(페이징 쿼리)
- 페이징은 페이지 번호, 무한 스크롤 방식이 있다.
  - 페이지 번호 - 이동할 페이지 번호가 명시적으로 지정 된다.
    - 페이지 번호에는 N번 페이지에서 M개의 게시글, 게시글의 갯수가 필요하다.
  - 무한 스크롤 - 스크롤을 내리면 다음 데이터 조회(더보기), 주로 모바일 환경

#### N번 페이지에서 M개의 게시글 
- SQL offset, limit 활용, offset 지점부터 limit개의 데이터 조회
- 게시판 별 게시글 목록 최신순 조회
  - shard key = board_id 이기 때문에 단일 샤드에서 조회 가능
  - limit : M개의 게시물
  - offset : (N번 페이지 - 1) * M, N > 0
```text
select * from article // 게시글 테이블
  where board_id = {board_id} // 게시판별
  order by created_at desc // 최신순
  limit {limit} offset {offset}; // N번 페이지에서 M개
```
- select * from article where board_id = 1 order by created_at desc limit 30 offset 90;
- 실행시 11초나 걸렸음..
```text
mysql> explain select * from article where board_id = 1 order by created_at desc limit 30 offset 90;
+----+-------------+---------+------------+------+---------------+------+---------+------+----------+----------+-----------------------------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows     | filtered | Extra
              |
+----+-------------+---------+------------+------+---------------+------+---------+------+----------+----------+-----------------------------+
|  1 | SIMPLE      | article | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 11758378 |    10.00 | Using where; Using filesort |
+----+-------------+---------+------------+------+---------------+------+---------+------+----------+----------+-----------------------------+
테이블 전체를 읽는다.(풀 스캔), 
Extras = Using where; Using filesort 
where절로 조건에 대해 필터링. 데이터가 많기 때문에 메모리에서 정렬을 수행할 수 없어서, 파일(디스크)에서 데이터를 정렬하는 filesort 수행.
```
- 이러한 문제를 해결하기 위해 인덱스를 사용한다.

<br>

- 인덱스 
  - 데이터를 빠르게 찾기 위한 방법
  - 인덱스 관리를 위해 부가적인 쓰기 작업과 공간이 필요
  - 다양한 데이터 특성과 쿼리를 지원하는 다양한 자료구조(b+ tree, Hash, LSM tree, R tree, Bitmap)

- Relational Database에서는 주로 B+ tree(Balanced Tree)
  - 데이터가 정렬된 상태로 저장된다.
  - 검색, 삽입, 삭제 연산을 로그 시간에 수행 가능
  - 트리 구조에서 leaf node 간 연결되기 때문에 범위 검색 효율적
- 인덱스를 추가하면, 쓰기 시점에 B+ tree 구조의 정렬된 상태의 데이터가 생성된다.
- 이미 인덱스로 지정된 컬럼에 대해 정렬된 상태를 가지고 있기 때문에, 조회 시점에 전체 데이터를 정렬하고 필터링할 필요가 없다. 따라서, 조회 쿼리를 빠르게 수행할 수 있다.

<br>

- create index idx_board_id_article_id on article(board_id asc, article_id desc);
  - board_id 오름차순 정렬, article_id 내림차순 정렬
  - 인덱스는 순서가 중요하다. 게시판별로 생성 시간 순으로 정렬된 상태의 데이터가 만들어질 것이다.
- 최신순을 위한 인덱스인데 article_id 가 사용된 이유
  - 현재 시스템은 대규모 트래픽을 처리하기 위한 분산 환경이다.
  - 게시글 서비스의 각 애플리케이션은 여러 서버로 분산되어 동시에 처리될 수 있다.
  - 게시글이 동시에 생성될 수 있는 것이다.
  - 예) 2개의 서버에서 동시에 요청을 받고 게시글 생성 시점에 동일한 created_at 생성
  - 동시 요청 시에 생성 시간은 충돌될 수 있다. 따라서, created_at를 정렬 조건으로 사용한다면, 동일한 생성 시간간에 순서가 명확하지 않다.
- article_id는 분산 시스템에서 고유한 **오름차순을 위해 고안된 알고리즘(Snowflake)을 사용한다.**  => 따라서, article_id를 최신순 정렬 용으로 사용할 수 있다.
- Primary key(article_id)에는 자동으로 인덱스가 생성된다. 하지만 테스트 데이터 수준에서는 유의미한 실행 속도 차이를 볼 수 없다.
```text
목록 조회의 최신순 정렬 조건은 article_id로 사용한다.
select * from article where board_id = {board_id} order by article_id desc limit {limit} offset {offset};
board_id 오름차순, article_id 내림차순 => 생성한 인덱스를 쿼리에 사용할 수 있어야 한다.
2번 게시판에서 2페이지를 조회하려면? => where board_id = 2 필터링을 위한 기준점을 먼저 찾고, 인덱스를 스캔하면서 offset = 1까지 스킵
2차 정렬 조건 article_id에 의해, 동일 게시판 내의 게시글 목록은 최신순 정렬되어 있다.
조회 시점에 데이터를 정렬하고 모든 데이터에 대해 직접 필터링하는 과정이 사라졌다.
- select * from article where board_id = 1 order by article_id desc limit 30 offset 90; 
실행시 아까 11초에서 인덱스 생성 후 0초로 실행시간이 줄어들었다. 
쿼리플랜 확인시 key = idx_board_id_article_id 생성한 인덱스가 쿼리에 사용됐다는 것을 확인할 수 있다.
- select * from article where board_id = 1 order by article_id desc limit 30 offset 1499970;
하지만 offset을 늘리면 5초나 걸린다. 뒷페이지로 갈수록 성능이 느려진다.

인덱스에는 Clustered Index, Secondary Index가 있다.

InnoDB는 테이블마다 Clustered Index를 자동 생성한다. => Primary Key를 기준으로 정렬된 Clustered Index(Primary Index)
Clustered Index는 leaf node의 값으로 행 데이터(row data)를 가진다.
select * from article where article_id = 111111111 => 인덱스 없이 엄청 빠르다. => Primary Key에 자동으로 생성된 Clustered Index가 사용됬기 때문

직접 생성한 인덱스는 Secondary Index(보조 인덱스)이다. 
Secondary Index의 leaf node는 인덱스 컬럼 데이터, 데이터에 접근하기 위한 포인터를 가지고 있다.
데이터는 Clustered Index가 가지고 있다. => Clustered Index의 데이터에 접근하기 위한 포인터

Primary Key를 이용한 데이터 조회는 Clustered Index를 통해 데이터를 빠르게 찾을 수 있었다.
Secondary Index를 이용한 조회는 Secondary Index는 데이터에 접근하기 위한 포인터만 가지고 있다. 그리고 데이터는 Clustered Index가 가지고 있다.

Primary Key -> id 컬럼 (Clustered Index), 인덱스 생성 -> col 컬럼 (Secondary Index)
Secondary Index 에서 col=4 을 조회하면 인덱스를 통해 데이터 접근할 수 있는 포인터(id)를 찾고, 이를 이용하여 Clustered Index에서 데이터를 찾는다.

• Secondary Index를 이용한 데이터 조회는, 인덱스 트리를 두 번 타고 있는 것이다.
1.Secondary Index에서 데이터에 접근하기 위한 포인터를 찾은 뒤,
2.Clustered Index에서 데이터를 찾는다.

select * from article where board_id = 1 order by article_id desc limit 30 offset 1499970;
1. (board_id, article_id)에 생성된 Secondary Index에서 article_id를 찾는다.
2. Clustered Index에서 article 데이터를 찾는다.
3. offset 1499970을 만날 때까지 반복하며 skip 한다.
4. limit 30개를 추출한다.

select board_id, article_id from article where board_id = 1 order by article_id desc limit 30 offset 1499970;
select 를 board_id, article_id 만 나오게 했다. 실행하면 0.2초로 시간이 확줄었다.
=> 이렇게 인덱스의 데이터만으로 조회를 수행할 수 있는 인덱스를 Covering Index라고 한다.
=> 이번 쿼리에서는 Clustered Index에 접근하는 과정은 없었다. Covering Index를 활용하여, Secondary Index에서만 board_id와 article_id를 추출한 것이다.

추출된 30건의 article_id 에서만 Clustered Index에 접근 하면 된다.
select * from (
  select article_id from article
  where board_id = 1
  order by article_id desc
  limit 30 offset 1499970
) t left join article on t.article_id = article.article_id;  => 0.23초 소요
Covering Index를 활용하여, offset 1499970에서부터 Clustered Index에 접근하게 된 것이다. 무의미한 데이터 접근 과정이 사라졌다

하지만 offset 8999970을 더늘리면 뒷페이지로 갈수록 결국 시간은 또 늘어났다.
article_id 추출을 위해 Secondary Index만 탄다고 하더라도, offset 만큼 Index Scan이 필요하다.
데이터에 접근하지 않더라도 offset이 늘어날 수록 느려질 수 밖에 없는 것이다.

** 해결 방법
- 데이터를 한 번 더 분리한다. 
  • 게시글을 1년 단위로 테이블 분리
  • 개별 테이블의 크기를 작게 만든다.
  • 각 단위에 대해 전체 게시글 수를 관리한다.
- offset을 인덱스 페이지 단위 skip하는 것이 아니라, 1년 동안 작성된 게시글 수 단위로 즉시 skip한다.
  • 조회하고자 하는 offset이 1년 동안 작성된 게시글 수보다 크다면,
    • 해당 개수 만큼 즉시 skip
    • 더 큰 단위로 skip을 수행하게 되는 것
  • 애플리케이션에서 이처럼 처리하기 위한 코드 작성 필요
  
무한 스크롤을 사용하면 뒷 페이지로 가더라도 균등한 조회 속도를 가진다.
```
- Covering Index
  - 인덱스만으로 쿼리의 모든 데이터를 처리할 수 있는 인덱스
  - 데이터(Clustered Index)를 읽지 않고, 인덱스(Secondary Index) 포함된 정보만으로 쿼리 가능한 인덱스

<br>

### 8. 게시글 목록 API - 페이지 번호 - 게시글 개수 - 설계
___
- 게시글의 개수는, 데이터 유무에 따라서 몇 페이지까지 활성화할지 결정하기 위해 필요하다.
- 페이지 당 30개의 게시글을, 페이지 이동 버튼을 10개씩 노출한다면, 게시글 50개 있다면, 2번 페이지까지 활성화 게시글 301개 이상 있다면, 다음 버튼까지 활성화(11페이지가 있음을 의미)
- select count(*) from article where board_id = 1;
- 1200만건인데도 2~3초가 소요된다. => 전체 게시글 수가 아닌 이동 가능한 페이지 번호 활성화가 필요
- (((n – 1) / k) + 1) * m * k + 1 , 현재 페이지(n) , 페이지 당 게시글 개수(m), 이동 가능한 페이지 개수(k)
- select count(*) from (select article_id from article where board_id = {board_id} limit {limit}) t; 
- select count(*) from (select article_id from article where board_id = 1 limit 300301) t => 0.08초 소요
  - count query에서 limit은 동작하지 않고 전체 개수를 반환하므로, sub query에서 Covering Index로 limit 만큼 조회하고 count하는 방식
-  게시글 개수를 조회 시점에 실시간으로 쿼리할 필요는 없다. 게시글 개수를 하나의 데이터로 미리 만들어둘 수도 있다.  
하지만 이를 위해, 동시 요청이 발생하는 상황에서 안전하고 정확하게 처리하기 위한 방법 필요

<br>

### 9. 게시글 목록 API - 페이지 번호 구현
___
- repository 에  findAll , 게시글 개수 구하는 쿼리 작성
```java
@Repository
public interface ArticleRepository extends JpaRepository<Article, Long> {

    // nativeQuery 사용 , pageable 을 사용하면 최적화된 쿼리 사용 할수 없다.
    @Query(
            value = "select article.article_id, article.title, article.content, article.board_id, article.writer_id, " +
                    "article.created_at, article.modified_at " +
                    "from(" +
                    "   select article_id from article " +  // subquery 에선 secondary index 에서 포인터만 먼저 뽑아낸다.
                    "   where board_id = :boardId " +
                    "   order by article_id desc " +
                    "   limit :limit offset :offset" +
                    ") t left join article on t.article_id = article.article_id",
            nativeQuery = true
    )
    List<Article> findAll(
            @Param("boardId") Long boardId,
            @Param("offset") Long offset,
            @Param("limit") Long limit
    );

    @Query(
            value = "select count(*) from (" +
                    "   select article_id from article where board_id = :boardId limit :limit" +
                    ") t ",
            nativeQuery = true
    )
    Long count(@Param("boardId") Long boardId, @Param("limit") Long limit);
}
```
- repository test
```java
@Slf4j
@SpringBootTest
class ArticleRepositoryTest {
    @Autowired
    ArticleRepository articleRepository;

    @Test
    void findAllTest() {
        List<Article> articles = articleRepository.findAll(1L, 1499970L, 30L);
        log.info("articles.size = {}", articles.size());
        for (Article article : articles) {
            log.info("article = {}", article);
        }
    }

    @Test
    void countTest() {
        Long count = articleRepository.count(1L, 10000L);
        log.info("count = {}", count);
    }
}
```

- Article count 반환을 위한 ArticlePageResponse.java 생성
```java
@Getter
@ToString
public class ArticlePageResponse {
    private List<ArticleResponse> articles;
    private Long articleCount;
    
    public static ArticlePageResponse of(final List<ArticleResponse> articles, final Long articleCount) {
        ArticlePageResponse response = new ArticlePageResponse();
        response.articles = articles;
        response.articleCount = articleCount;
        
        return response;
    }
}
```
- page 번호 활성화에 필요한 공식을 구하는 PageLimitCalculator.java
```java
@NoArgsConstructor(access = AccessLevel.PRIVATE)    // 유틸성 클래스기 때문에 private 생성자 , final 클래스 지정
public final class PageLimitCalculator {

    public static Long calculatePageLimit(Long page, Long pageSize, Long movablePageCount) {
        return (((page - 1) / movablePageCount) + 1) * pageSize * movablePageCount + 1;
    }
}
```
- service, 게시글 조회 메서드 실행 List<ArticleResponse> articles, Long articleCount 반환
```java
@Service
@RequiredArgsConstructor
public class ArticleService {
    private final Snowflake snowflake = new Snowflake();
    private final ArticleRepository articleRepository;
    
    public ArticlePageResponse readAll(Long boardId, Long page, Long pageSize) {
        return ArticlePageResponse.of(
                articleRepository.findAll(boardId, (page - 1) * pageSize, pageSize).stream()
                        .map(ArticleResponse::from) //Article 엔티티를 ArticleResponse DTO로 변환 List<ArticleResponse> 형태로 반환
                        .toList(),
                articleRepository.count(
                        boardId,
                        PageLimitCalculator.calculatePageLimit(page, pageSize, 10L)
                )
        );
    }
}
```

- controller 추가
```java
@RestController
@RequiredArgsConstructor
public class ArticleController {
    private final ArticleService articleService;

    @GetMapping("/v1/articles")
    public ArticlePageResponse readAll(
            @RequestParam("boardId") Long boardId,
            @RequestParam("size") Long page,
            @RequestParam("pageSize") Long pageSize
    ) {
        return articleService.readAll(boardId, page, pageSize);
    }

}
```

- test 추가
```java
public class ArticleApiTest {
    RestClient restClient = RestClient.create("http://localhost:9000");    // http 요청 할 수 있는 클래스

    @Test
    void readAllTest() {
        ArticlePageResponse response = restClient.get()
                .uri("/v1/articles?boardId=1&pageSize=30&page=1")
                .retrieve()
                .body(ArticlePageResponse.class);

        System.out.println("response.getArticleCount() = " + response.getArticleCount());
        for (ArticleResponse article : response.getArticles()) {
            System.out.println("articleId = " + article.getArticleId());
        }
    }
}

```

- PageLimitCalculator.java 테스트
```java
class PageLimitCalculatorTest {

    @Test
    void calculatePageLimitTest() {
        calculatePageLimitTest(1L, 30L, 10L, 301L);
        calculatePageLimitTest(7L, 30L, 10L, 301L);
        calculatePageLimitTest(10L, 30L, 10L, 301L);
        calculatePageLimitTest(11L, 30L, 10L, 601L);
        calculatePageLimitTest(12L, 30L, 10L, 601L);
    }

    void calculatePageLimitTest(Long page, Long pageSize, Long movablePageCount, Long expected) {
        Long result = PageLimitCalculator.calculatePageLimit(page, pageSize, movablePageCount);
        assertThat(result).isEqualTo(expected);

    }
}
```

<br>

### 10. 게시글 목록 API - 무한 스크롤 설계
___
- 무한 스크롤 : 스크롤을 내리면 다음 데이터 조회, 주로 모바일 환경에서 사용, 페이지 번호 방식 XX
```text 
페이징 번호 방식을 사용하면 보여진 데이터가 삭제되거나 데이터가 추가가 되면 데이터가 중복/누락 될 수 있다.
마지막으로 불러온 데이터가 기준점이 되야한다.
정확한 데이터 기준점이 있기 떄문에 인덱스에서 로그 시간에 기준점을 찾을 수 있다.
1. 첫 페이지를 조회한 Client는 마지막 조회한 데이터의 기준점을 알고 있다.
2. 다음 페이지를 불러올 때 마지막 조회한 데이터의 기준점을 파라미터로 전달한다면?
3. 데이터베이스에서는 기준점으로 쿼리를 수행한다.
    • 이 때, 기준점에 생성된 인덱스를 통해 로그 시간에 접근할 수 있다.
    • 즉, offset만큼 scan하는 과정이 필요하지 않다. limit 개수를 즉시 추출할 수 있다.
    • 따라서, 아무리 뒷 페이지로 가더라도, 균등한 속도를 보장할 수 있다.
```
```sql
# 1번 페이지
select * from article
where board_id = {board_id}
order by article_id desc limit 30;
# 2번 페이지 이상, 기준점 = {last_article_id}
select * from article
where board_id = {board_id} and article_id < {last_article_id}
order by article_id desc limit 30;
```
- select * from article where board_id = 1 order by article_id desc limit 30;
- select * from article where board_id = 1 and article_id < 147555803545141852 order by article_id desc limit 30;
- select article_id from article where board_id = 1 order by article_id asc limit 1 offset 30; // 마지막 페이지의 기준점
- select * from article where board_id = 1 and article_id < 147550439382646804 order by article_id desc limit 30;
- 기준점을 인덱스에서 로그 시간에 즉시 찾을 수 있으므로, 아무리 뒷 페이지를 가더라도 균등한 조회 속도가 보장되는 것이다.

<br>

### 11. 게시글 목록 API - 무한 스크롤 설계
___
- repository 에 무한스크롤 게시물 조회 쿼리 생성 (1번페이지일때, 2번페이지 이상일때)
```java
@Repository
public interface ArticleRepository extends JpaRepository<Article, Long> {
    // 1번 페이지
    @Query(
            value = "select article.article_id, article.title, article.content, article.board_id, article.writer_id, " +
                    "article.created_at, article.modified_at " +
                    "from article " +
                    "where board_id = :boardId " +
                    "order by article_id desc limit :limit",
            nativeQuery = true
    )
    List<Article> findAllInfiniteScroll(@Param("boardId") Long boardId, @Param("limit") Long limit);

    // 2번 페이지 이상
    @Query(
            value = "select article.article_id, article.title, article.content, article.board_id, article.writer_id, " +
                    "article.created_at, article.modified_at " +
                    "from article " +
                    "where board_id = :boardId and article_id < :lastArticleId  " +
                    "order by article_id desc limit :limit",
            nativeQuery = true
    )
    List<Article> findAllInfiniteScroll(
            @Param("boardId") Long boardId,
            @Param("limit") Long limit,
            @Param("lastArticleId") Long lastArticleId
            );
}
```
- repository Test
```java
@Slf4j
@SpringBootTest
class ArticleRepositoryTest {
    @Autowired
    ArticleRepository articleRepository;
    @Test
    void findInfiniteScrollTest() {
        List<Article> articles = articleRepository.findAllInfiniteScroll(1L, 30L);
        for (Article article : articles) {
            log.info("articleId = {}", article.getArticleId());
        }

        Long lastArticleId = articles.getLast().getArticleId();

        List<Article> articles2 = articleRepository.findAllInfiniteScroll(1L, 30L, lastArticleId);
        for (Article article : articles2) {
            log.info("articleId2 = {}", article.getArticleId());
        }
    }
}
```

- service 생성
```java
@Service
@RequiredArgsConstructor
public class ArticleService {
    private final Snowflake snowflake = new Snowflake();
    private final ArticleRepository articleRepository;

    public List<ArticleResponse> readAllInfiniteScroll(Long boardId, Long pageSize, Long lastArticleId) {
        List<Article> articles = lastArticleId == null ?
                articleRepository.findAllInfiniteScroll(boardId, pageSize) :
                articleRepository.findAllInfiniteScroll(boardId, pageSize, lastArticleId);
        return articles.stream().map(ArticleResponse::from).toList();
    }
}
```
- controller 생성
```java
@RestController
@RequiredArgsConstructor
public class ArticleController {
    private final ArticleService articleService;

    @GetMapping("/v1/articles/infinite-scroll")
    public List<ArticleResponse> readAllInfiniteScroll(
            @RequestParam("boardId") Long boardId,
            @RequestParam("pageSize") Long pageSize,
            @RequestParam(value = "lastArticleId", required = false) Long lastArticleId
    ) {
        return articleService.readAllInfiniteScroll(boardId, pageSize, lastArticleId);
    }

}
```

- test 작성
```java
public class ArticleApiTest {
    RestClient restClient = RestClient.create("http://localhost:9000");    // http 요청 할 수 있는 클래스

    @Test
    void readAllInfiniteScrollTest() {
        List<ArticleResponse> articles1 = restClient.get()
                .uri("/v1/articles/infinite-scroll?boardId=1&pageSize=5")
                .retrieve()
                .body(new ParameterizedTypeReference<List<ArticleResponse>>() {
                });// 리스트로 반환이기 때문에 ParameterizedTypeReference 사용

        System.out.println("first Page");
        for (ArticleResponse articleResponse : articles1) {
            System.out.println("articleId = " + articleResponse.getArticleId());
        }

        Long lastArticleId = articles1.getLast().getArticleId();

        List<ArticleResponse> articles2 = restClient.get()
                .uri("/v1/articles/infinite-scroll?boardId=1&pageSize=5&lastArticleId=%s".formatted(lastArticleId))
                .retrieve()
                .body(new ParameterizedTypeReference<List<ArticleResponse>>() {
                });

        System.out.println("second Page");
        for (ArticleResponse articleResponse : articles2) {
            System.out.println("articleId = " + articleResponse.getArticleId());
        }
    }
}
```

<br>

### 12. Primary Key 생성 전략
___
#### DB auto_increment
- 분산 데이터베이스 환경에서 PK가 중복될 수 있기 때문에, 식별자의 유일성이 보장되지 않는다.
  - 여러 샤드에서 동일한 PK를 가지는 상황
- 클라이언트 측에 노출하면 보안 문제
- 간단한 장점이 있어서 보안적인 문제를 크게 고려하지 않는 상황, 단일 DB를 사용하거나 애플리케이션에서 PK의 중복을 직접 구분하는 상황 사용
- 보안이 필요하면, PK는 데이터베이스 내에서의 식별자로만 사용하고, 애플리케이션에서의 식별자를 위해 별도 유니크 인덱스를 사용할 수도 있다.
  - PK = id(DB auto_increment), unique index = article_id(UUID 등), Client는 article_id만 식별자로서 노출 및 사용
  - 별도의 식별자를 사용하면 Secondary Index로 포인터 찾은 후, Clustered Index로 데이터 접근하므로, 조회 비용 증가한다.

<br> 

#### 유니크 문자열 또는 숫자 
- UUID 또는 난수를 생성하여 PK를 지정할 수 있다.
  - 키 생성 방식이 간단, 정렬데이터가 아닌 랜덤데이터를 삽입
- 랜덤 데이터로 인해 성능 저하가 발생할 수 있다.
  - Clustered Index는 정렬된 상태를 유지한다.
  - 데이터 삽입 필요한 인덱스 페이지가 가득 찼다면, B+ tree 재구성 및 페이지 분할로 디스크 I/O 증가
  - PK를 이용한 범위 조회가 필요하다면, 디스크에서 랜덤 I/O가 발생하기 때문에, 순차 I/O보다 성능 저하

<br>

#### 유니크 정렬 문자열
- 분산 환경에 대한 PK 중복 문제 해결, 보안 문제 해결, 랜덤 데이터에 의한 성능 문제 해결
- UUID v7, ULID 등의 알고리즘
  - 일반적으로 알려진 알고리즘은 128비트를 사용한다.
- 데이터 크기에 따라, 공간 및 성능 효율이 달라진다.
- Clustered Index는 PK를 기준으로 만들어진다.
- Secondary Index는 데이터에 접근할 수 있는 포인터를 가진다. => 즉, PK를 가지고 있다.
- PK가 크면 클수록, 데이터는 더 많은 공간을 차지, 비교 연산에 의한 정렬/조회에 더 많은 비용 소모

<br>

#### 
- 분산 환경에 대한 PK 중복 문제 해결, 보안 문제 해결, 랜덤 데이터에 의한 성능 문제 해결
- Snowflake, TSID 등의 알고리즘
  - 64비트를 사용한다. (BIGINT)
  - 정렬을 위해 타임스탬프를 나타내는 비트 수의 제한으로, 키 생성을 위한 시간적인 한계가 있을 수 있다. 문자열 알고리즘에서도 동일한 문제 있으나 비트 수가 많을수록 제한이 덜할 수 있다.
- 문자열 방식보다 적은 공간을 사용한다.
