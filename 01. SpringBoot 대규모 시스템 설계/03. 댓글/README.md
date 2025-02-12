## 댓글

### 1. 댓글 최대 2 depth - 테이블 설계
___

#### 댓글
- 댓글 조회, 생성, 삭제 API
- 댓글 목록 조회 API
- 계층형 : 최대 2 depth의 대댓글, 무한 depth의 대댓글
  - 계층형 대댓글 : 하위 댓글이 모두 삭제되어야 상위 댓글을 삭제할 수 있다.
    - 하위 댓글이 없으면, 댓글은 즉시 삭제된다. 하위 댓글이 있으면, 댓글은 삭제 표시만 된다.
- 계층별 오래된순 정렬(최신 댓글이 가장 아래)
- 페이지 번호, 무한 스크롤

<br>

- 최대 2 depth : Adjacency list (인접 리스트)
- 무한 depth : Path enumeration (경로 열거)

#### 테이블 설계 
- 계층형 댓글에서는 계층 관계를 알아야하므로, 상위 댓글의 참조 키가 필요하다. 각 댓글은 상위 댓글 ID(parent_comment_id)를 가진다.
- 각 게시글에 작성된 댓글 목록을 단일 샤드에서 조회하기 위해, article_id를 기준으로 샤딩한다.

```sql
create table comment (
  comment_id bigint not null primary key,
  content varchar(3000) not null,
  article_id bigint not null,
  parent_comment_id bigint not null,
  writer_id bigint not null,
  deleted bool not null,
  created_at datetime not null
);
```

<br>

### 2. 댓글 최대 2 depth - CUD API 구현
___
- build.gradle, application.yml 파일에 설정 파일 추가(article 과 거의 동일)
- Entity 생성
```java
@Table(name = "comment")
@Getter
@Entity
@ToString
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Comment {
    @Id
    private Long commentId;
    private String content;
    private Long parentCommentId;
    private Long articleId; // shard key
    private Long writerId;
    private Boolean deleted;
    private LocalDateTime createdAt;

    public static Comment create(Long commentId, String content, Long parentCommentId, Long articleId, Long writerId) {
        Comment comment = new Comment();
        comment.commentId = commentId;
        comment.content = content;
        comment.parentCommentId = parentCommentId == null ? commentId : parentCommentId;    // 상위 댓글 없으면 자기 자신 id
        comment.articleId = articleId;
        comment.writerId = writerId;
        comment.deleted = false;    // 생성할땐 false 로
        comment.createdAt = LocalDateTime.now();
        return comment;
    }

    public boolean isRoot() {   // 1 depth 댓글인지 확인 유무
        return parentCommentId.longValue() == commentId;
    }

    public void delete() {  // 삭제시 true 로 변경
        deleted = true;
    }
}
```
- repository 생성 => JpaRepository
```java
@Repository
public interface CommentRepository extends JpaRepository<Comment, Long> {
    // 특정 게시글에서 상위 댓글 갯수를 이용해 자식개수를 구할 수 있다. 
    @Query(
            value = "select count(*) from (" +
                    "   select comment_id from comment  " +
                    "   where article_id = :articleId and parent_comment_id = :parentCommentId  " +
                    "   limit :limit" +
                    ") t", 
            nativeQuery = true
    )
    Long countBy(
            @Param("articleId") Long articleId,
            @Param("parentCommentId") Long parentCommentId,
            @Param("limit") Long limit
    );
}
```
- request, response 객체 생성
```java
@Getter
public class CommentCreateRequest {
    private Long articleId;
    private String content;
    private Long parentCommentId;
    private Long writerId;
}
```
```java
@Getter
@ToString
public class CommentResponse {
    private Long commentId;
    private String content;
    private Long parentCommentId;
    private Long articleId;
    private Long writerId;
    private Boolean deleted;
    private LocalDateTime createdAt;

    public static CommentResponse from(Comment comment) {
        CommentResponse response = new CommentResponse();
        response.commentId = comment.getCommentId();
        response.content = comment.getContent();
        response.parentCommentId = comment.getParentCommentId();
        response.articleId = comment.getArticleId();
        response.writerId = comment.getWriterId();
        response.deleted = comment.getDeleted();
        response.createdAt = comment.getCreatedAt();
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

    @Transactional
    public CommentResponse create(CommentCreateRequest request) {
        Comment parent = findParent(request);
        Comment comment = commentRepository.save(
                Comment.create(
                        snowflake.nextId(),
                        request.getContent(),
                        parent == null ? null : parent.getParentCommentId(),
                        request.getArticleId(),
                        request.getWriterId()
                )
        );
        return CommentResponse.from(comment);
    }

    private Comment findParent(CommentCreateRequest request) {
        Long parentCommentId = request.getParentCommentId();
        if(parentCommentId == null){
            return null;
        }
        return commentRepository.findById(parentCommentId)
                .filter(not(Comment::getDeleted))   // 상위 댓글이 삭제된 상태가 아니여야 댓글 작성 가능
                .filter(Comment::isRoot)    // 상위댓글은 root 댓글이여야한다. 최대 2depth 이기 때문
                .orElseThrow();

    }

    public CommentResponse read(Long commentId) {
        return CommentResponse.from(commentRepository.findById(commentId).orElseThrow());
    }

  @Transactional
    public void delete(Long commentId) {
        commentRepository.findById(commentId)
                .filter(not(Comment::getDeleted))
                .ifPresent(comment -> { // 존재 하는 경우
                    if(hasChildren(comment)){   // 대댓글 유무
                        comment.delete();   // delete true 로 변경
                    }else {
                        delete(comment);    // 삭제
                    }
                });
    }

    private boolean hasChildren(Comment comment) {
        return commentRepository.countBy(comment.getArticleId(), comment.getCommentId(), 2L) == 2;  // 2개 이면 하위 댓글이 있다.
    }


    private void delete(Comment comment) {
        commentRepository.delete(comment);
        if(!comment.isRoot()){  // 삭제된 상위 댓글을 찾아 지워야한다. (상위 댓글이 삭제됬어도 자식이 있어서 못 지워진 경우)
            commentRepository.findById(comment.getParentCommentId())
                    .filter(Comment::getDeleted)    // delete true
                    .filter(not(this::hasChildren)) // 자식이 없어야 지울수 있다.
                    .ifPresent(this::delete);   // 삭제 수행
        }
    }
}
```
- delete 에 대한 TEST
```java
@ExtendWith(MockitoExtension.class) // 단위 테스트로 작성
class CommentServiceTest {
  @InjectMocks
  CommentService commentService; // commentService 생성
  @Mock
  CommentRepository commentRepository; // mocking 객체로 commentRepository 주입

  @Test
  @DisplayName("삭제할 댓글이 자식이 있으면, 삭제 표시만 한다.")
  void deleteShouldMarkDeletedIfHasChildren() {
    //given
    Long articleId = 1L;
    Long commentId = 2L;

    Comment comment = createComment(articleId, commentId);
    given(commentRepository.findById(commentId))
            .willReturn(Optional.of(comment));  // findById 의 결과가 모킹 객체일떄
    given(commentRepository.countBy(articleId, commentId, 2L)).willReturn(2L);

    // when
    commentService.delete(commentId);

    // then
    verify(comment).delete(); // 삭제 표시만 해야함.
  }

  @Test
  @DisplayName("하위 댓글이 삭제되고, 삭제되지 않은 부모면, 하위 댓글만 삭제한다.")
  void deleteShouldDeletechildOnlyIfNotDeletedParent() {

    //given
    Long articleId = 1L;
    Long commentId = 2L;
    Long parentCommentId = 1L;

    Comment comment = createComment(articleId, commentId, parentCommentId);    // 상위 댓글이 있따.
    given(comment.isRoot()).willReturn(false); // 루트 댓글이 아니기 때문에 false 반환

    Comment parentComment = mock(Comment.class);
    given(parentComment.getDeleted()).willReturn(false);

    given(commentRepository.findById(commentId)).willReturn(Optional.of(comment));
    given(commentRepository.countBy(articleId, commentId, 2L)).willReturn(1L);
    given(commentRepository.findById(parentCommentId)).willReturn(Optional.of(parentComment));

    // when
    commentService.delete(commentId);

    // then
    verify(commentRepository).delete(comment);      // 하위 댓글 삭제
    verify(commentRepository, never()).delete(parentComment);    //상위 댓글은 삭제 내역이 없기 때문에 delete가 호출되지 않는다.
  }

  @Test
  @DisplayName("하위 댓글이 삭제되고, 삭제된 부모면, 재귀적으로 모두 삭제된다.")
  void deleteShouldDeleteAllRecursivelyIfDeletedParent() {

    //given
    Long articleId = 1L;
    Long commentId = 2L;
    Long parentCommentId = 1L;

    Comment comment = createComment(articleId, commentId, parentCommentId);
    given(comment.isRoot()).willReturn(false);

    Comment parentComment = createComment(articleId, parentCommentId);
    given(parentComment.isRoot()).willReturn(true);
    given(parentComment.getDeleted()).willReturn(true);

    given(commentRepository.findById(commentId)).willReturn(Optional.of(comment));
    given(commentRepository.countBy(articleId, commentId, 2L)).willReturn(1L);

    given(commentRepository.findById(parentCommentId)).willReturn(Optional.of(parentComment));
    given(commentRepository.countBy(articleId, parentCommentId, 2L)).willReturn(1L);

    // when
    commentService.delete(commentId);

    // then
    verify(commentRepository).delete(comment);
    verify(commentRepository).delete(parentComment);
  }

  // mock 객체 생성
  private Comment createComment(Long articleId, Long commentId) {
    Comment comment = mock(Comment.class);
    // 전달받은 파라미터를 mock 객체에 넣어준다.
    given(comment.getArticleId()).willReturn(articleId);
    given(comment.getCommentId()).willReturn(commentId);
    return comment;
  }

  private Comment createComment(Long articleId, Long commentId, Long parentCommentId) {
    Comment comment = createComment(articleId, commentId);
    given(comment.getParentCommentId()).willReturn(parentCommentId);
    return comment;
  }
}
```
- controller 생성
```java
@RestController
@RequiredArgsConstructor
public class CommentController {
    private final CommentService commentService;

    // 조회
    @GetMapping("/v1/comments/{commentId}")
    public CommentResponse read(@PathVariable Long commentId) {
        return commentService.read(commentId);
    }

    // 생성
    @PostMapping("/v1/comments")
    public CommentResponse create(@RequestBody CommentCreateRequest request) {
        return commentService.create(request);
    }

    // 삭제
    @DeleteMapping("/v1/commets/{commentId}")
    public void delete(@PathVariable("commentId") Long commentId) {
        commentService.delete(commentId);
    }
}
```
- CUD TEST
```java
public class CommentApiTest {

    RestClient restClient = RestClient.create("http://localhost:9001");

    @Test
    void create() {
        CommentResponse response1 = createComment(new CommentCreateRequest(1L, "my comment1", null, 1L));
        CommentResponse response2 = createComment(new CommentCreateRequest(1L, "my comment2", response1.getCommentId(), 1L));
        CommentResponse response3 = createComment(new CommentCreateRequest(1L, "my comment3", response1.getCommentId(), 1L));

        System.out.println("commentId=%s".formatted(response1.getCommentId()));
        System.out.println("\tcommentId=%s".formatted(response2.getCommentId()));
        System.out.println("\tcommentId=%s".formatted(response3.getCommentId()));


    }

    CommentResponse createComment(CommentCreateRequest request) {
        return restClient.post()
                .uri("/v1/comments")
                .body(request)
                .retrieve()
                .body(CommentResponse.class);
    }

    @Test
    void read() {
        CommentResponse response = restClient.get()
                .uri("/v1/comments/{commentId}", 147907627385585664L)
                .retrieve()
                .body(CommentResponse.class);

        System.out.println("response = " + response);
    }

    @Test
    void delete() {
        restClient.delete()
                .uri("/v1/comments/{commentId}", 147907628325109760L)
                .retrieve();
         /*테스트 root 댓글 1개 , 대댓글 2개 
         1. root 댓글 삭제 -> root 댓글의 delete가 1로 변경
         2. 하위 댓글 삭제 -> DB에서 바로 삭제 된다
         3. 한개남은 하위 댓글을 삭제 -> root, 하위 댓글 같이 삭제*/ 
    }


    @Getter
    @AllArgsConstructor
    static class CommentCreateRequest {
        private Long articleId;
        private String content;
        private Long parentCommentId;
        private Long writerId;
    }
}
```
