### 6. 댓글 무한 depth - 테이블&구현 설계
___
- 댓글 목록 조회 – 무한 depth
```text
상위 댓글은 항상 하위 댓글보다 먼저 생성되고, 하위 댓글은 상위 댓글 별 순서대로 생성된다.
위 규칙은, 무한 depth 에서는 상하위 댓글이 재귀적으로 무한할 수 있으므로, 정렬 순을 나타내기 위해 모든 상위 댓글의 정보가 필요할 수 있다.

parent_comment_id = 1 > 2 > 6 이러한 형식으로 상위 댓글의 정보를 표시
댓글 구조를 트리로 본다면, root node(1 depth 최상위 댓글) 부터 각 node(각 depth 댓글)에 도달하기 위한 경로로 이해해볼 수 있다.
각 depth 만큼 컬럼을 만들어서, 각 depth별 댓글 ID를 관리해줄 수도 있지만, 테이블 구조가 복잡해지고 무한 depth라 그만큼 컬럼 개수를 생성 X

그래서 각 depth에서의 순서를 문자열로 나타내고, 이러한 문자열을 순서대로 결합하여 경로를 나타내는 것이다. => Path Enumeration(경로 열거) 방식

위와 같이 각 depth 별로 5개의 문자열로 경로 정보를 저장한다.
1 depth는 5개의 문자로, 2 depth는 10개의 문자로, 3 depth는 15개의 문자로, N depth는 (N * 5)개의 문자로, 문자열로 모든 상위 댓글에서 각 댓글까지의 경로를 저장하는 것이다.
그리고, 각 경로는 상위 경로를 상속하며 독립적이고 순차적인 경로를 가지도록 한다.

1 depth : 00000 , 2 depth 00000 00000 , 3 depth 00000 00000 00000
각 경로는 상위 댓글의 경로를 상속하며, 각 댓글마다 독립적이고 순차적인(문자열 순서) 경로가 생성된다.

표현 범위가 제한되므로 표현할 수 있는 경로의 범위가 각 경로 별로 10^5 = 100,000개로 제한된다.
문자열 순서 = 0~9 < A-Z < a-z 이렇게 섞어서 쓰면 더 많이 사용 할 수 있다. => 9억개
우리는 문자열에 대해 순서를 정의하고, 인덱스를 사용하여 정렬된 데이터를 관리해야한다.
```
#### 데이터베이스 collation
- 문자열을 정렬하고 비교하는 방법을 정의하는 설정 : 이는 대소문자 구분, 악센트 구분, 언어별 정렬 순서 등을 포함, 데이터베이스/테이블/컬럼별 설정
- 디폴트는 utf8mb4_0900_ai_ci => utf8mb4 : 각문자 최대 4바이트 utf8 지원, 0900 : 정렬 방식 버전, ai : 악센트 비구분, ci : 대소문자 비구분 
- 0~9 < A-Z < a-z 을 비교하는 것을 사용 할 것이므로 utf8mb4_bin 을 사용하여 대소문자 구분도 추가해준다.

#### 댓글 테이블 설계
- 기존 parent_comment_id는 제거 , 댓글 경로를 나타내기 위한 path 컬럼 추가
- 설계 구조 상 무한이지만 일단은 5 depth 로제한 varchar(25)
```sql
create table comment_v2 (
    comment_id bigint not null primary key,
    content varchar(3000) not null,
    article_id bigint not null,
    writer_id bigint not null,
    path varchar(25) character set utf8mb4 collate utf8mb4_bin not null,
    deleted bool not null,
    created_at datetime not null
);
```
- 인덱스 설정
  - path에 인덱스를 생성하여 정렬 데이터를 관리한다. => 페이징에 사용될 수 있다.
  - path는 독립적인 경로를 가지므로, unique index로 생성한다. => 애플리케이션에서의 동시성 문제를 막아줄 수 있을 것이다.
```sql
create unique index idx_article_id_path on comment_v2(
    article_id asc, path asc
);
```

#### path 생성 구현 방법
```text
00a0z -> 00a0z 00000, 00a0z 00001, 00a0z 00002 -> 00a0z 00002 00000
이렇게 있을 때 00a0z 의 하위에 신규 댓글 작성이 되면 00a0z의 하위 댓글 중 가장 큰 path(childrenTopPath) 가 00a0z 00002 를 찾고 +1 한 문자열이 path 가 된다.

childrenTopPath(00a0z 00002)을 찾으려면
각 댓글의 모든 자손 댓글은 prefix가 상위 댓글의 path(parentPath, 00a0z)로 시작한다는 특성을 이용해볼 수 있다.
00a0z의 모든 자손 댓글은 prefix가 00a0z로 시작한다.

00a0z의 prefix(parentPath)를 가지는 모든 자손 댓글에서, 가장 큰 path(descendantsTopPath)를 찾아보자.
descendantsTopPath는 신규 댓글의 depth와 다를 수 있지만, childrenTopPath를 포함한다.
descendantsTopPath에서 (신규 댓글의 depth * 5)까지만 남기고 잘라내는 것이다.
00a0z 00002 00000에서 (신규 댓글의 depth 2) * 5 = 10개의 문자만 남기고 잘라낸다.
그러면 childrenTopPath = 00a0z 00002를 찾을 수 있다.

descendantsTopPath만 빠르게 잘 찾아내면, childrenTopPath는 애플리케이션 로직으로 어렵지 않게 구할 수 있을 것이다.
```
- childrenTopPath 구하는 쿼리
```sql
select path from comment_v2
where article_id = {article_id}
    and path > {parentPath} // parent 본인은 미포함 검색 조건
    and path like {parentPath}% // parentPath를 prefix로 하는 모든 자손 검색 조건
order by path desc limit 1; // 조회 결과에서 가장 큰 path
```
```text
article_id asc, path asc로 인덱스를 생성했으므로 where 범위 필터링은 정상적으로 처리될 수 있을 듯하다.
그런데 path는 오름차순으로 생성되어 있다. 
가장 큰 path 찾기 위한 정렬 조건이 path desc limit 1 때문에, 조회 시점에 내림차순 정렬을 하는 건 아닐지 염려해볼 수 있다.

explain 을 확인해보면 idx_article_id_path 사용되었다. 
Using where; Backward index scan; Using index
Extras=Using index를 통해 커버링 인덱스로 동작했음을 알 수 있다.

Backward index scan 란 index 를 역순 스캔 하는 것이다. • 인덱스 트리 leaf node 간에 연결된 양방향 포인터를 활용
결국 descendantsTopPath는 인덱스 덕분에 빠르게 찾아질 수 있다는 것이다.

신규 댓글을 만들때 숫자가 아니라 문자열 기반으로 덧셈 연산을 수행해야한다. 0-9 < A-Z < a-z
- 문자열 덧셈으로 구하기 : 0부터 z까지의 대소 관계를 정의하고, 오른쪽 문자부터 다음 문자로 바꿔준다.
    - carry(올림수)가 없으면(=0~y가 1~z로 바뀌면), 처리 종료
    - carry(올림수)가 있으면(=z가 0으로 바뀌면), 다음 문자도 처리
    - 이미 zzzzz 일땐 overflow

- 숫자 덧셈으로 구하기 : 62진수 문자열을 10진수 숫자로 바꿔서 +1 한 다음에, 숫자를 대응하는 문자열로 다시 바꿔준다


**** 예외 케이스
-  00a0z의 하위 댓글이 없어서 최초 생성일때 
    => descendantsTopPath가 없으므로, 00a0z 00000가 신규 댓글의 path가 된다. => parentPath + 00000
- 해당 경로에서 childrenTopPath = zzzzz까지 댓글이 생성되어 있을때
    => 값을 표현할 수 있는 범위(62^5개)를 벗어났기 때문에 더 이상 생성될 수 없다. , depth 별 문자개수를 늘려서 해결 가능
```

#### 페이지 번호 , 무한스크롤 일때 조회 쿼리
```sql
# 목록 조회
select * from (
  select comment_id
  from comment_v2
  where article_id = {article_id}
  order by path asc
    limit {limit} offset {offset} 
) t left join comment_v2 on t.comment_id = comment_v2.comment_id;  

# 카운트 조회
select count(*) from (
    select comment_id from comment_v2 where article_id = {article_id} limit {limit}
) t;

# 무한스크롤 1번 페이지 일때
select * from comment_v2
  where article_id = {article_id}
  order by path asc limit {limit};
  
# 무한스크롤 2페이지 이상 , 기준점 last_path
select * from comment_v2
  where article_id = {article_id} and path > {last_path}
  order by path asc
    limit {limit};
```

### 7. 댓글 무한 depth - CUD API 구현 & 테스트 데이터 삽입
___
- 경로 path 컬럼 값 관련 Embeddable, 여러개의 필드를 하나의 객체로 만들고 싶을때 @Embeddable, @Embedded 사용
```java
@Getter
@ToString
@Embeddable
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class CommentPath {
    private String path;

    // path 가 가질 수 있는 characterSet
    private static final String CHARSET = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";

    private static final int DEPTH_CHUNK_SIZE = 5;  // 경로정보 depth 당 5개
    private static final int MAX_DEPTH = 5; // 최대 depth

    // MIN_CHUNK = 00000, MAX_CHUNK = zzzzz
    private static final String MIN_CHUNK = String.valueOf(CHARSET.charAt(0)).repeat(DEPTH_CHUNK_SIZE);
    private static final String MAX_CHUNK = String.valueOf(CHARSET.charAt(CHARSET.length() -1)).repeat(DEPTH_CHUNK_SIZE);

    public static CommentPath create(String path) {
        if (isDepthOverflowed(path)) {
            throw new IllegalStateException("Depth overflowed");
        }
        CommentPath commentPath = new CommentPath();
        commentPath.path = path;
        return commentPath;
    }

    private static boolean isDepthOverflowed(String path) {     // 최대 5 depth 가 넘어가면 예외
        return callDepth(path) > MAX_DEPTH; // 5  depth 가 넘어가면 true 반환
    }

    private static int callDepth(String path) {     // depth 정보 계산 = path 길이 / 5
        return path.length() / DEPTH_CHUNK_SIZE;
    }

    public int getDepth() {     // path 의 depth 를 구하는 함수
        return callDepth(path);
    }

    public boolean isRoot() {   // 제일 상위인지
        return callDepth(path) == 1;
    }

    public String getParentPath() { // 현재 path 의 parentPath 를 구하는 함수, 끝에 5개만 잘라내면 된다.
        return path.substring(0, path.length() - DEPTH_CHUNK_SIZE);
    }

    // 현재 path 에 하위 댓글의 path 만드는 메소드
    public CommentPath createChildCommentPath(String descendantsTopPath) {
        if(descendantsTopPath == null) {    // 하위 댓글이 처음 생성 되는 상황
            return CommentPath.create(path + MIN_CHUNK);
        }
        String childrenTopPath = findChildrenTopPath(descendantsTopPath);
        return CommentPath.create(increase(childrenTopPath)); // childrenTopPath + 1
    }

    private String findChildrenTopPath(String descendantsTopPath) {
        return descendantsTopPath.substring(0, (getDepth() + 1) * DEPTH_CHUNK_SIZE);
    }

    private String increase(String path) {
        String lastChunk = path.substring(path.length() - DEPTH_CHUNK_SIZE);// path 의 가장 마지막 5개의 문자열
        if (isChunkOverflowed(lastChunk)) {
            throw new IllegalStateException("Chunk overflowed");
        }
        // overflow 가 나지 않으면 path 에서 + 1
        int charsetLength = CHARSET.length();
        int value = 0;  // lastChunk 를 10 진수로 변환하기 위한 값
        for (char ch : lastChunk.toCharArray()) {
            value = value * charsetLength + CHARSET.indexOf(ch);
        }

        value = value + 1;

        String result = ""; // value 를 62 진수로 변환
        for (int i=0; i < DEPTH_CHUNK_SIZE; i++) {
            result = CHARSET.charAt(value % charsetLength) + result;
            value /= charsetLength;
        }

        return path.substring(0, path.length() - DEPTH_CHUNK_SIZE) + result;    // 상위 댓글 경로 정보 + result

    }

    private boolean isChunkOverflowed(String lastChunk) {   // zzzzz 인지 확인
        return MAX_CHUNK.equals(lastChunk);
    }
}
```
- CommentPathTest
```java
class CommentPathTest {

    @Test
    void createChildCommentTest() {
        createChildCommentTest(CommentPath.create(""), null, "00000");
        createChildCommentTest(CommentPath.create("00000"), null, "0000000000");
        createChildCommentTest(CommentPath.create(""), "00000", "00001");
        createChildCommentTest(CommentPath.create("0000z"), "0000zabcdzzzzzzzzzzz", "0000zabce0");
    }

    void createChildCommentTest(CommentPath commentPath, String descendantsTopPath, String expectedChildPath) {
        CommentPath childCommentPath = commentPath.createChildCommentPath(descendantsTopPath);
        assertThat(childCommentPath.getPath()).isEqualTo(expectedChildPath);
    }

    @Test
    void createChildCommentPathIfMaxDepthTest() {
        assertThatThrownBy(() ->
                CommentPath.create("zzzzz".repeat(5)).createChildCommentPath(null)
        ).isInstanceOf(IllegalStateException.class);
    }

    @Test
    void createChildCommentPathIfChunkOverflowTest() {
        CommentPath commentPath = CommentPath.create("");
        assertThatThrownBy(() -> commentPath.createChildCommentPath("zzzzz"))
                .isInstanceOf(IllegalStateException.class);
    }
}
```
- entity
```java
@Table(name = "comment_v2")
@Getter
@Entity
@ToString
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class CommentV2 {
    @Id
    private Long commentId;
    private String content;
    private Long articleId; // shard key
    private Long writerId;
    @Embedded
    private CommentPath commentPath;
    private Boolean deleted;
    private LocalDateTime createdAt;

    public static CommentV2 create(Long commentId, String content, Long articleId, Long writerId, CommentPath commentPath) {
        CommentV2 comment = new CommentV2();
        comment.commentId = commentId;
        comment.content = content;
        comment.articleId = articleId;
        comment.writerId = writerId;
        comment.commentPath = commentPath;
        comment.deleted = false;
        comment.createdAt = LocalDateTime.now();
        return comment;
    }

    public boolean isRoot() {
        return commentPath.isRoot();
    }
    
    public void delete() {
        deleted = true;
    }
}
```
- repository
```java
@Repository
public interface CommentRepositoryV2 extends JpaRepository<CommentV2, Long> {

    // path 가 unique 한 index 로 만들어져있기 때문에 path 로 댓글을 찾을 수 있다.
    @Query("select c from CommentV2 c where c.commentPath.path = :path")    //jqpl
    Optional<CommentV2> findByPath(@Param("path") String path);
    
    // descendantsTopPath
    @Query(
            value = "select path from comment_v2 " +
                    "where articleId = :articleId " +
                    " and path > :pathPrefix " +    //  자기 자신 제외
                    " and path like :pathPrefix% " +    // parentPath를 prefix로 하는 모든 자손 검색 조건
                    " order by path desc limit 1",
            nativeQuery = true
    )
    Optional<String> findDescendantsTopPath(
            @Param("articleId") Long articleId,
            @Param("pathPrefix") String pathPrefix  // parent Path
    );
}
```
- service
```java
@Getter
public class CommentCreateRequestV2 {
    private Long articleId;
    private String content;
    private String parentPath;
    private Long writerId;
}
```
- response 에 V2 용 팩토리 메서드도 생성
```java
@Service
@RequiredArgsConstructor
public class CommentServiceV2 {
    private final Snowflake snowflake = new Snowflake();
    private final CommentRepositoryV2 commentRepository;

    @Transactional
    public CommentResponse create(CommentCreateRequestV2 request) {
        CommentV2 parent = findParent(request); // 상위 댓글 찾기
        CommentPath parentCommentPath = parent == null ? CommentPath.create("") : parent.getCommentPath();
        CommentV2 comment = commentRepository.save(
            CommentV2.create(
                snowflake.nextId(),
                request.getContent(),
                request.getArticleId(),
                request.getWriterId(),
                parentCommentPath.createChildCommentPath(
                        commentRepository.findDescendantsTopPath(request.getArticleId(), parentCommentPath.getPath())
                                .orElse(null)
                )
            )
        );
        return CommentResponse.from(comment);
    }

    private CommentV2 findParent(CommentCreateRequestV2 request) {
        String parentPath = request.getParentPath();
        if(parentPath == null) {
            return null;
        }
        return commentRepository.findByPath(parentPath)
                .filter(not(CommentV2::getDeleted)) // 삭제 확인
                .orElseThrow();
    }
    
    // 읽기 
    public CommentResponse read(Long commentId) {
        return CommentResponse.from(commentRepository.findById(commentId).orElseThrow());
    }
    
    // 삭제
    @Transactional
    public void delete(Long commentId) {
        commentRepository.findById(commentId)
                .filter(not(CommentV2::getDeleted))
                .ifPresent(comment -> {
                    if (hasChildren(comment)) {
                        comment.delete();
                    } else {
                        delete(comment);
                    } 
                });
        
    }

    // descendantsTopPath 를 조회해서 없으면 자손 댓글이 X
    private boolean hasChildren(CommentV2 comment) {
        return commentRepository.findDescendantsTopPath(
                comment.getArticleId(),
                comment.getCommentPath().getPath()
        ).isPresent(); 
    }

    private void delete(CommentV2 comment) {
        commentRepository.delete(comment);
        if(!comment.isRoot()){  // 삭제된 상위 댓글을 찾아 지워야한다. (상위 댓글이 삭제됬어도 자식이 있어서 못 지워진 경우)
            commentRepository.findByPath(comment.getCommentPath().getParentPath())
                    .filter(CommentV2::getDeleted)    // delete true
                    .filter(not(this::hasChildren)) // 자식이 없어야 지울수 있다.
                    .ifPresent(this::delete);   // 삭제 수행
        }
    }
}
```
- controller
```java
@RestController
@RequiredArgsConstructor
public class CommentControllerV2 {
    private final CommentServiceV2 commentService;

    // 조회
    @GetMapping("/v2/comments/{commentId}")
    public CommentResponse read(@PathVariable Long commentId) {
        return commentService.read(commentId);
    }

    // 생성
    @PostMapping("/v2/comments")
    public CommentResponse create(@RequestBody CommentCreateRequestV2 request) {
        return commentService.create(request);
    }

    // 삭제
    @DeleteMapping("/v2/comments/{commentId}")
    public void delete(@PathVariable("commentId") Long commentId) {
        commentService.delete(commentId);
    }
}
```
- api test
```java
public class CommentApiV2Test {
    RestClient restClient = RestClient.create("http://localhost:9001");

    @Test
    void create() {
        CommentResponse response1 = create(new CommentCreateRequestV2(1L, "my comment1", null, 1L));
        CommentResponse response2 = create(new CommentCreateRequestV2(1L, "my comment2", response1.getPath(), 1L));
        CommentResponse response3 = create(new CommentCreateRequestV2(1L, "my comment3", response2.getPath(), 1L));

        System.out.println("response1.getPath() = " + response1.getPath());
        System.out.println("response1.getCommentId() = " + response1.getCommentId());
        System.out.println("\tresponse2.getPath() = " + response2.getPath());
        System.out.println("\tresponse2.getCommentId() = " + response2.getCommentId());
        System.out.println("\t\tresponse3.getPath() = " + response3.getPath());
        System.out.println("\t\tresponse3.getCommentId() = " + response3.getCommentId());
    }

    CommentResponse create(CommentCreateRequestV2 request) {
        return restClient.post()
                .uri("/v2/comments")
                .body(request)
                .retrieve()
                .body(CommentResponse.class);
    }
 
    @Test
    void read() {
        CommentResponse response = restClient.get()
                .uri("/v2/comments/{commentId}", 148277722815254528L)
                .retrieve()
                .body(CommentResponse.class);
        System.out.println("response = " + response);
    }

    @Test
    void delete() {
        restClient.delete()
                .uri("/v2/comments/{commentId}", 148277722815254528L)
                .retrieve();
    }

    @Getter
    @AllArgsConstructor
    static class CommentCreateRequestV2 {
        private Long articleId;
        private String content;
        private String parentPath;
        private Long writerId;
    }
}
```
- data 생성
- 멀티스레드로 동시에 생성 하므로 path 가 겹칠수도 있는데   
path 는 unique index 로 만들어져서 독립적인 경로다. start 와 end 로 중복이안되게 범위를 파라미터로 전달

```java
@SpringBootTest
public class DataInitializerV2 {

    @PersistenceContext
    EntityManager entityManager;
    @Autowired
    TransactionTemplate transactionTemplate;
    Snowflake snowflake = new Snowflake();
    CountDownLatch latch = new CountDownLatch(EXECUTE_COUNT);

    static final int BULK_INSERT_SIZE = 2000;
    static final int EXECUTE_COUNT = 6000;

    @Test
    void initialize() throws InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(10);
        for(int i = 0; i < EXECUTE_COUNT; i++) {
            int start = i * BULK_INSERT_SIZE;
            int end = (i + 1) * BULK_INSERT_SIZE;
            executorService.submit(() -> {
                insert(start, end);
                latch.countDown();
                System.out.println("latch.getCount() = " + latch.getCount());
            });
        }
        latch.await();
        executorService.shutdown();
    }

    void insert(int start, int end) {
        transactionTemplate.executeWithoutResult(status -> {
            for(int i = start; i < end; i++) {
                CommentV2 comment = CommentV2.create(
                        snowflake.nextId(),
                        "content",
                        1L,
                        1L,
                        toPath(i)
                );
                entityManager.persist(comment);
            }
        });
    }

    private static final String CHARSET = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";
    private static final int DEPTH_CHUNK_SIZE = 5;  // 경로정보 depth 당 5개

    CommentPath toPath(int value) {
        String path = "";
        for (int i=0; i < DEPTH_CHUNK_SIZE; i++) {
            path = CHARSET.charAt(value % CHARSET.length()) + path;
            value /= CHARSET.length();
        }
        return CommentPath.create(path);
    }
}
```

<br>

### 8. 댓글 무한 depth - 목록 API 구현
___
- repository
```java
@Repository
public interface CommentRepositoryV2 extends JpaRepository<CommentV2, Long> {
    // 게시글 목록 조회
    @Query(
            value = "select comment_v2.comment_id, comment_v2.content, comment_v2.path, comment_v2.article_id   " +
                    "comment_v2.writer_id, comment_v2.deleted, comment_v2 created_at    " +
                    "from (" +
                    "   select comment_id from comment_v2 where article_id = :articleId " +
                    "   order by path asc   " +
                    "   limit :limit offset :offset " +
                    ") t left join comment_v2 on t.comment_id = comment_v2.comment_id",
            nativeQuery = true
    )
    List<CommentV2> findAll(
            @Param("articleId") Long articleId,
            @Param("offset") Long offset,
            @Param("limit") Long limit
    );

    @Query(
            value = "select count(*) from (" +
                    "   select comment_id from comment_v2 where article_id :articleId limit :limit" +
                    ") t",
            nativeQuery = true
    )
    Long count(@Param("articleId") Long articleId, @Param("limit") Long limit);

    // 무한 스크롤 
    @Query(
            value = "select comment_v2.comment_id, comment_v2.content, comment_v2.path, comment_v2.article_id   " +
                    "comment_v2.writer_id, comment_v2.deleted, comment_v2 created_at    " +
                    "from comment_v2 " +
                    "where article_id = :articleId  " +
                    "order by path asc  " +
                    "limit :limit",
            nativeQuery = true
    )
    List<CommentV2> findAllInfiniteScroll(
            @Param("articleId") Long articleId,
            @Param("limit") Long limit
    );

    
    @Query(
            value = "select comment_v2.comment_id, comment_v2.content, comment_v2.path, comment_v2.article_id   " +
                    "comment_v2.writer_id, comment_v2.deleted, comment_v2 created_at    " +
                    "from comment_v2 " +
                    "where article_id = :articleId and path > :lastPath " +
                    "order by path asc  " +
                    "limit :limit",
            nativeQuery = true
    )
    List<CommentV2> findAllInfiniteScroll(
            @Param("articleId") Long articleId,
            @Param("lastPath") String lastPath,
            @Param("limit") Long limit
    );
}
```
- service
```java
@Service
@RequiredArgsConstructor
public class CommentServiceV2 {
    private final Snowflake snowflake = new Snowflake();
    private final CommentRepositoryV2 commentRepository;

    // 페이지 번호 방식 메소드
    public CommentPageResponse readAll(Long articleId, Long page, Long pageSize) {
        return CommentPageResponse.of(
                commentRepository.findAll(articleId, (page - 1) * pageSize, pageSize).stream()
                        .map(CommentResponse::from)
                        .toList(),
                commentRepository.count(articleId, PageLimitCalculator.calculatePageLimit(page, pageSize, 10L))
        );
    }

    public List<CommentResponse> readAllInfiniteScroll(Long articleId, String lastPath, Long pageSize) {
        List<CommentV2> comments = lastPath == null ?
                commentRepository.findAllInfiniteScroll(articleId, pageSize) :
                commentRepository.findAllInfiniteScroll(articleId, lastPath, pageSize);
        return comments.stream().map(CommentResponse::from).toList();
    }
}
```

- controller
```java
@RestController
@RequiredArgsConstructor
public class CommentControllerV2 {
    private final CommentServiceV2 commentService;

    // 페이지 방식 목록 조회
    @GetMapping("/v2/comments")
    public CommentPageResponse readAll(
            @RequestParam("articleId") Long articleId,
            @RequestParam("page") Long page,
            @RequestParam("pageSize") Long pageSize
    ) {
        return commentService.readAll(articleId, page, pageSize);
    }

    // 무한스크롤 방식 목록 조회
    @GetMapping("/v2/comments/infinite-scroll")
    public List<CommentResponse> readAllInfiniteScroll(
            @RequestParam("articleId") Long articleId,
            @RequestParam(value = "lastPath", required = false) String lastPath,
            @RequestParam("pageSize") Long pageSize
    ) {
        return commentService.readAllInfiniteScroll(articleId, lastPath, pageSize);
    }
}
```
- test
```java
public class CommentApiV2Test {
    RestClient restClient = RestClient.create("http://localhost:9001");

    @Test
    void readAll() {
        CommentPageResponse response = restClient.get()
                .uri("/v2/comments?articleId=1&pageSize=10&page=50000")
                .retrieve()
                .body(CommentPageResponse.class);

        System.out.println("response = " + response.getCommentCount());
        for (CommentResponse comment : response.getComments()) {
            System.out.println("comment.getCommentId() = " + comment.getCommentId());
        }
    }

    @Test
    void readAllInfiniteScroll() {
        List<CommentResponse> response1 = restClient.get()
                .uri("/v2/comments/infinite-scroll?articleId=1&pageSize=5")
                .retrieve()
                .body(new ParameterizedTypeReference<List<CommentResponse>>() {
                });

        System.out.println("first page");
        for (CommentResponse response : response1) {
            System.out.println("response.getCommentId() = " + response.getCommentId());
        }

        String lastPath = response1.getLast().getPath();

        List<CommentResponse> response2 = restClient.get()
                .uri("/v2/comments/infinite-scroll?articleId=1&pageSize=5&lastPath=%s".formatted(lastPath))
                .retrieve()
                .body(new ParameterizedTypeReference<List<CommentResponse>>() {
                });

        System.out.println("second page");
        for (CommentResponse response : response2) {
            System.out.println("response.getCommentId() = " + response.getCommentId());
        }
    }
}
```
