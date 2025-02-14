### 6. 게시글 수&댓글 수 설계
___
- 게시판의 게시글 총 개수, 게시글의 댓글 총 개수
```sql
create table board_article_count (
    board_id bigint not null primary key,
    article_count bigint not null
);

create table article_comment_count (
    article_id bigint not null primary key,
    comment_count bigint not null
);
```

<br>

### 7. 게시글 수 구현
___
- count entity
```java
@Table(name = "board_article_count")
@Entity
@Getter
@ToString
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class BoardArticleCount {
    @Id
    private Long boardId;   // shard key
    private Long articleCount;
    
    public static BoardArticleCount init(Long boardId, Long articleCount) {
        BoardArticleCount boardArticleCount = new BoardArticleCount();
        boardArticleCount.boardId = boardId;
        boardArticleCount.articleCount = articleCount;
        return boardArticleCount;
    }
}
```
- repository
```java
@Repository
public interface BoardArticleCountRepository extends JpaRepository<BoardArticleCount, Long> {

    @Query(
            value = "update board_article_count set article_count = article_count + 1 where board_id = :boardId",
            nativeQuery = true
    )
    @Modifying
    int increase(@Param("boardId") Long boardId);

    @Query(
            value = "update board_article_count set article_count = article_count - 1 where board_id = :boardId",
            nativeQuery = true
    )
    @Modifying
    int decrease(@Param("boardId") Long boardId);
}
```
- service
```java
@Service
@RequiredArgsConstructor
public class ArticleService {
    private final Snowflake snowflake = new Snowflake();
    private final ArticleRepository articleRepository;
    private final BoardArticleCountRepository boardArticleCountRepository;

    @Transactional
    public ArticleResponse create(ArticleCreateRequest request) {
        Article article = articleRepository.save(
                Article.create(snowflake.nextId(), request.getTitle(), request.getContent(), request.getBoardId(), request.getWriterId())
        );
        /* total 게시글 수 증가 S */
        int result = boardArticleCountRepository.increase(article.getBoardId());
        if (result == 0) {
            boardArticleCountRepository.save(
                    BoardArticleCount.init(request.getBoardId(), 1L)
            );
        }
        /* total 게시글 수 증가 E */
        
        return ArticleResponse.from(article);
    }

    @Transactional
    public void delete(Long articleId) {
//        articleRepository.deleteById(articleId);
        /* total 게시글 수 감소 S */
        Article article = articleRepository.findById(articleId).orElseThrow();
        articleRepository.delete(article);
        boardArticleCountRepository.decrease(article.getBoardId());
        /* total 게시글 수 감소 E */
    }

    // total 개수 구하기
    public Long count(Long boardId) {
        return boardArticleCountRepository.findById(boardId)
                .map(BoardArticleCount::getArticleCount)
                .orElse(0L);
    }
}
```
- controller 
```java
@RestController
@RequiredArgsConstructor
public class ArticleController {
    private final ArticleService articleService;

    @GetMapping("/v1/articles/boardId/{boardId}/count")
    public Long count(@PathVariable Long boardId) {
        return articleService.count(boardId);
    }
}
```
- test
```java
public class ArticleApiTest {
    RestClient restClient = RestClient.create("http://localhost:9000");    // http 요청 할 수 있는 클래스

    ArticleResponse create(ArticleCreateRequest request) {
        return restClient.post()
                .uri("/v1/articles")
                .body(request)
                .retrieve() // 호출
                .body(ArticleResponse.class);
    }

    @Test
    void countTest() {
        ArticleResponse response = create(new ArticleCreateRequest("hi", "count", 1L, 2L));

        Long count1 = restClient.get()
                .uri("/v1/articles/boardId/{boardId}/count", 2L)
                .retrieve()
                .body(Long.class);
        System.out.println("count = " + count1);

        // 삭제
        restClient.delete()
                .uri("/v1/articles/{articleId}", response.getArticleId())
                .retrieve();

        Long count2 = restClient.get()
                .uri("/v1/articles/boardId/{boardId}/count", 2L)
                .retrieve()
                .body(Long.class);
        System.out.println("count2 = " + count2);
    }
}
```

<br>

### 8. 댓글 수 구현
___
- entity
```java
@Table(name = "article_comment_count")
@Entity
@Getter
@ToString
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class ArticleCommentCount {
    @Id
    private Long articleId; //shard key
    private Long commentCount;

    public static ArticleCommentCount init(Long articleId, Long commentCount) {
        ArticleCommentCount articleCommentCount = new ArticleCommentCount();
        articleCommentCount.articleId = articleId;
        articleCommentCount.commentCount = commentCount;
        return articleCommentCount;
    }
}
```
- repository
```java
@Repository
public interface ArticleCommentCountRepository extends JpaRepository<ArticleCommentCount, Long> {

    @Query(
            value = "update article_comment_count set comment_count = comment_count + 1 where article_id = :articleId",
            nativeQuery = true
    )
    @Modifying
    int increase(@Param("articleId") Long articleId);

    @Query(
            value = "update article_comment_count set comment_count = comment_count - 1 where article_id = :articleId",
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
public class CommentServiceV2 {
    private final Snowflake snowflake = new Snowflake();
    private final CommentRepositoryV2 commentRepository;
    private final ArticleCommentCountRepository articleCommentCountRepository;

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
        /* total 댓글 수 증가 S */
        int result = articleCommentCountRepository.increase(request.getArticleId());
        if (result == 0) {
            articleCommentCountRepository.save(
                    ArticleCommentCount.init(request.getArticleId(), 1L)
            );
        }
        /* total 댓글 수 증가 E */
        
        return CommentResponse.from(comment);
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
        /* total 게시글 수 감소 S */
        articleCommentCountRepository.decrease(comment.getArticleId());
        /* total 게시글 수 감소 E */
        if(!comment.isRoot()){  // 삭제된 상위 댓글을 찾아 지워야한다. (상위 댓글이 삭제됬어도 자식이 있어서 못 지워진 경우)
            commentRepository.findByPath(comment.getCommentPath().getParentPath())
                    .filter(CommentV2::getDeleted)    // delete true
                    .filter(not(this::hasChildren)) // 자식이 없어야 지울수 있다.
                    .ifPresent(this::delete);   // 삭제 수행
        }
    }

    public Long count(Long articleId) {
        return articleCommentCountRepository.findById(articleId)
                .map(ArticleCommentCount::getCommentCount)
                .orElse(0L);
    }
}
```
- controller
```java
@RestController
@RequiredArgsConstructor
public class CommentControllerV2 {
    private final CommentServiceV2 commentService;

    // total count
    @GetMapping("/v2/comments/articles/{articleId}/count")
    public Long count(@PathVariable Long articleId) {
        return commentService.count(articleId);
    }
}
```
- test
```java
public class CommentApiV2Test {
    RestClient restClient = RestClient.create("http://localhost:9001");

    CommentResponse create(CommentCreateRequestV2 request) {
        return restClient.post()
                .uri("/v2/comments")
                .body(request)
                .retrieve()
                .body(CommentResponse.class);
    }
 
    @Test
    void countTest() {
        CommentResponse commentResponse = create(new CommentCreateRequestV2(2L, "my comment1", null, 1L));

        Long count1 = restClient.get()
                .uri("/v2/comments/articles/{articleId}/count", 2L)
                .retrieve()
                .body(Long.class);
        System.out.println("count1 = " + count1);
        assertThat(count1).isEqualTo(1L);

        restClient.delete()
                .uri("/v2/comments/{commentId}", commentResponse.getCommentId())
                .retrieve();
        Long count2 = restClient.get()
                .uri("/v2/comments/articles/{articleId}/count", 2L)
                .retrieve()
                .body(Long.class);
        System.out.println("count1 = " + count2);
        assertThat(count2).isEqualTo(0L);
    }
}
```
