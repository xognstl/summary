### 4. 댓글 최대 2 depth - 목록 API 설계
___
```text
댓글은 오래된 댓글이 먼저 노출된다.
계층형 구조라 생성 시간(comment_id)으로는 정렬 순서가 명확하지 않다. 
계층 관계에서는 더 늦게 작성된 댓글이 먼저 노출될 수 있다. 
상위 댓글은 하위 댓글보다 반드시 먼저 생성된다. 
같은 상위 댓글을 공유하는 하위 댓글들은, 생성 시간 순으로 정렬 순서가 명확하다.
즉, 상위 댓글은 항상 하위 댓글보다 먼저 생성되고,  
하위 댓글은 상위 댓글 별 순서대로 생성된다.상위 댓글에 의해 순서가 명확하게 정해질 수 있는 것이다.

==> (parent_comment_id 오름차순, comment_id 오름차순)의 정렬 구조를 가지고 있다.

• article_id asc, parent_comment_id asc, comment_id asc;
    • (게시글 ID 오름차순, 상위 댓글 ID 오름차순, 댓글 ID 오름차순)
    • ID는 오름차순으로 생성된다. (snowflake)
    • shard key = article_id 이기 때문에, 단일 샤드에서 게시글별 댓글 목록을 조회할 수 있다.
    
```
- 인덱스 생성
```sql
create index idx_article_id_parent_comment_id_comment_id on comment (
    article_id asc, parent_comment_id asc, comment_id asc
);
```
- N번 페이지에서 M개의 댓글 조회 : 서브쿼리로 covering index 만 뽑아내서 사용
```sql
  select * from (
      select comment_id from comment
      where article_id = {article_id}
      order by parent_comment_id asc, comment_id asc
      limit {limit} offset {offset}
  ) t left join comment on t.comment_id = comment.comment_id;
```
- 댓글 개수 조회
```sql
select count(*) from (
    select comment_id from comment where article_id = {article_id} limit {limit}
) t;
```
- 무한 스크롤
- 1번 페이지
```sql
select * from comment
    where article_id = {article_id}
    order by parent_comment_id asc, comment_id asc
    limit {limit};
```
- 2번 페이지 이상 => 기준점 = last_parent_comment_id, last_comment_id
```sql
select * from comment
    where article_id = {article_id} and (
        parent_comment_id > {last_parent_comment_id} or
        (parent_comment_id = {last_parent_comment_id} and comment_id > {last_comment_id})
    )
    order by parent_comment_id asc, comment_id asc
    limit {limit};
```

<br>

### 5. 댓글 최대 2 depth - 목록 API 구현
___
- repository N번페이지 M개 글 조회, 댓글 개수 조회 , 무한스크롤 1페이지, 2페이지 이상에 대한 쿼리 작성
```java
@Repository
public interface CommentRepository extends JpaRepository<Comment, Long> {
    // N번 페이지에서 M개의 댓글 조회
    @Query(
            value = "select comment.comment_id, comment.content, comment.parent_commentId, comment.article_id, " +
                    "comment.writer_id, comment.deleted, comment.created_at " +
                    "from (" +
                    "   select comment_id from comment where article_id :articleId  " +
                    "   order by parent_comment_id asc, comment_id asc  " +
                    "   limit :limit offset :offset " +
                    ") t left join comment on t.comment_id = comment.comment_id" ,
            nativeQuery = true
    )
    List<Comment> findAll(
            @Param("articleId") Long articleId,
            @Param("offset") Long offset,
            @Param("limit") Long limit
    );

    //댓글 개수 조회
    @Query(
            value = "select count(*) from (" +
                    "   select comment_id from comment where article_id = :articleId limit :limit" +
                    ") t",
            nativeQuery = true
    )
    Long count(
        @Param("articleId") Long articleId,
        @Param("limit") Long limit
    );

    // 무한 스크롤 1번 페이지
    @Query(
            value = "select comment.comment_id, comment.content, comment.parent_comment_id, comment.article_id, " +
                    "comment.writer_id, comment.deleted, comment.created_at " +
                    "from comment   " +
                    "where article_id = :articleId " +
                    "order by parent_comment_id asc, comment_id asc  " +
                    "limit :limit",
            nativeQuery = true
    )
    List<Comment> findAllInfiniteScroll(
            @Param("articleId") Long articleId,
            @Param("limit") Long limit
    );

    // 무한 스크롤 2번 페이지 이상
    @Query(
            value = "select comment.comment_id, comment.content, comment.parent_comment_id, comment.article_id, " +
                    "comment.writer_id, comment.deleted, comment.created_at " +
                    "from comment   " +
                    "where article_id = :articleId and (" +
                    "   parent_comment_id > :lastParentCommentId or" +
                    "   (parent_comment_id = :lastParentCommentId and comment_id > :lastCommentId)" +
                    ")" +
                    "order by parent_comment_id asc, comment_id asc  " +
                    "limit :limit",
            nativeQuery = true
    )
    List<Comment> findAllInfiniteScroll(
            @Param("articleId") Long articleId,
            @Param("lastParentCommentId") Long lastParentCommentId,
            @Param("lastCommentId") Long lastCommentId,
            @Param("limit") Long limit
    );
}
```

- CommentPageResponse 페이지 생성, article 에서 사용했던 PageLimitCalculator.java 가저오기
```java
@Getter
public class CommentPageResponse {
    private List<Comment> comments;
    private Long commentCount;

    private static CommentPageResponse of(List<Comment> comments, Long commentCount) {
        CommentPageResponse response = new CommentPageResponse();
        response.comments = comments;
        response.commentCount = commentCount;
        return response;
    }
}
```

- service 생성
```java
@Service
@RequiredArgsConstructor
public class CommentService {
    private final Snowflake snowflake = new Snowflake();
    private final CommentRepository commentRepository;

    public CommentPageResponse readAll(Long articleId, Long page, Long pageSize) {
        return CommentPageResponse.of(
                commentRepository.findAll(articleId, (page - 1) * pageSize, pageSize).stream()
                        .map(CommentResponse::from)
                        .toList(),
                commentRepository.count(articleId, PageLimitCalculator.calculatePageLimit(page, pageSize, 10L))
        );
    }

    // 무한 스크롤
    public List<CommentResponse> readAll(Long articleId, Long lastParentCommentId, Long lastCommentId, Long limit) {
        List<Comment> comments = lastParentCommentId == null || lastCommentId == null ?
                commentRepository.findAllInfiniteScroll(articleId, limit) :
                commentRepository.findAllInfiniteScroll(articleId, lastParentCommentId, lastCommentId, limit);
        return comments.stream().map(CommentResponse::from).toList();
    }
}
```
- controller
```java
@RestController
@RequiredArgsConstructor
public class CommentController {
    private final CommentService commentService;

    @GetMapping("/v1/comments")
    public CommentPageResponse readAll(
            @RequestParam("articleId") Long articleId,
            @RequestParam("page") Long page,
            @RequestParam("pageSize") Long pageSize
    ) {
        return commentService.readAll(articleId, page, pageSize);
    }

    @GetMapping("/v1/comments/infinite-scroll")
    public List<CommentResponse> readAll(
            @RequestParam("articleId") Long articleId,
            @RequestParam(value = "lastParentCommentId", required = false) Long lastParentCommentId,
            @RequestParam(value = "lastCommentId", required = false) Long lastCommentId,
            @RequestParam("pageSize") Long pageSize
    ) {
        return commentService.readAll(articleId, lastParentCommentId, lastCommentId, pageSize);
    }
}
```

- test 1페이지 readAll 과 readAllInfiniteScroll의 결과 값은 같다.
```java
public class CommentApiTest {

    RestClient restClient = RestClient.create("http://localhost:9001");

    @Test
    void readAll() {
        CommentPageResponse response = restClient.get()
                .uri("/v1/comments?articleId=1&page=1&pageSize=10")
                .retrieve()
                .body(CommentPageResponse.class);

        System.out.println("response.getCommentCount() = " + response.getCommentCount());
        for (CommentResponse comment : response.getComments()) {
            if (!comment.getCommentId().equals(comment.getParentCommentId())) {
                System.out.print("\t");
            }
            System.out.println("comment.getCommentId() = " + comment.getCommentId());
        }

    }


    @Test
    void readAllInfiniteScroll() {
        List<CommentResponse> response1 = restClient.get()
                .uri("v1/comments/infinite-scroll?articleId=1&pageSize=5")
                .retrieve()
                .body(new ParameterizedTypeReference<List<CommentResponse>>() {
                });
        System.out.println("first page");
        for (CommentResponse comment : response1) {
            if (!comment.getCommentId().equals(comment.getParentCommentId())) {
                System.out.print("\t");
            }
            System.out.println("comment.getCommentId() = " + comment.getCommentId());
        }

        Long lastParentCommentId = response1.getLast().getParentCommentId();
        Long lastCommentId = response1.getLast().getCommentId();

        List<CommentResponse> response2 = restClient.get()
                .uri("v1/comments/infinite-scroll?articleId=1&pageSize=5&lastParentCommentId%s&lastCommentId=%s"
                        .formatted(lastParentCommentId, lastCommentId))
                .retrieve()
                .body(new ParameterizedTypeReference<List<CommentResponse>>() {
                });
        System.out.println("second page");
        for (CommentResponse comment : response1) {
            if (!comment.getCommentId().equals(comment.getParentCommentId())) {
                System.out.print("\t");
            }
            System.out.println("comment.getCommentId() = " + comment.getCommentId());
        }
    }

}
```
