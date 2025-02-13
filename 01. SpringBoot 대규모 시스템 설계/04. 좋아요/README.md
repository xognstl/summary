## 좋아요

### 1. 좋아요 설계 
___
#### 좋아요
- 게시글 좋아요 : 각 사용자는 각 게시글에 1회 좋아요
- 좋아요 수 

#### 설계
- 각 사용자는 게시글마다 좋아요를 누를 수 있다. 취소도 가능
- 이러한 좋아요는 각 게시글마다 1회만 수행 된다. ==> 1개의 고유한 데이터만 관리되어야 한다.
- MYSQL 에서 유니크 인덱스(게시글 ID + 사용자 ID)를 이용

#### 테이블 설계
```sql
create table article_like (
    article_like_id bigint not null primary key,
    article_id bigint not null,
    user_id bigint not null,
    created_at datetime not null
);
create unique index idx_article_id_user_id on article_like(article_id asc, user_id asc);
```
-  create database article_like; 생성 후 테이블 생성

<br>

### 2. 좋아요 구현
___
- dependency , application.yml 파일 설정
- entity
```java
@Table(name = "article_like")
@Getter
@Entity
@ToString
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class ArticleLike {
    @Id
    private Long articleLikeId;
    private Long articleId; // shardKey
    private Long userId;
    private LocalDateTime createdAt;
    
    public static ArticleLike create(Long articleLikeId, Long articleId, Long userId) {
        ArticleLike articleLike = new ArticleLike();
        articleLike.articleLikeId = articleLikeId;
        articleLike.articleId = articleId;
        articleLike.userId = userId;
        articleLike.createdAt = LocalDateTime.now();
        return articleLike;
    }
}
```
- repository
```java
@Repository
public interface ArticleLikeRepository extends JpaRepository<ArticleLike, Long> {
    //articleId, userId 로 데이터 조회
    Optional<ArticleLike> findByArticleIdAndUserId(Long articleId, Long userId);
}
```
- response
```java
@Getter
@ToString
public class ArticleLikeResponse {
    private Long articleLikeId;
    private Long articleId;
    private Long userId;
    private LocalDateTime createdAt;
    
    public static ArticleLikeResponse from(ArticleLike articleLike) {
        ArticleLikeResponse response = new ArticleLikeResponse();
        response.articleLikeId = articleLike.getArticleLikeId();
        response.articleId = articleLike.getArticleId();
        response.userId = articleLike.getUserId();
        response.createdAt = articleLike.getCreatedAt();
        return response;
    }
}
```

- service
```java
@Service
@RequiredArgsConstructor
public class ArticleLikeService {
    private final Snowflake snowflake = new Snowflake();
    private final ArticleLikeRepository articleLikeRepository;

    public ArticleLikeResponse read(Long articleId, Long userId) {  // 사용자가 좋아요 했는지 유무
        return articleLikeRepository.findByArticleIdAndUserId(articleId, userId)
                .map(ArticleLikeResponse::from)
                .orElseThrow();
    }

    @Transactional
    public void like(Long articleId, Long userId) { // 좋아요 수행
        articleLikeRepository.save(
                ArticleLike.create(
                        snowflake.nextId(),
                        articleId,
                        userId
                )
        );
    }   // unique index 떄문에 한건만 데이터 생성
    
    @Transactional
    public void unlike(Long articleId, Long userId) {   // 좋아요 취소
        articleLikeRepository.findByArticleIdAndUserId(articleId, userId)
                .ifPresent(articleLikeRepository::delete);
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
    @GetMapping("/v1/articles-likes/articles/{articleId}/users/{userId}")
    public ArticleLikeResponse read(
            @PathVariable("articleId") Long articleId,
            @PathVariable("userId") Long userId
    ) {
        return articleLikeService.read(articleId, userId);
    }

    // 좋아요
    @PostMapping("/v1/articles-likes/articles/{articleId}/users/{userId}")
    public void like(
            @PathVariable("articleId") Long articleId,
            @PathVariable("userId") Long userId
    ) {
        articleLikeService.like(articleId, userId);
    }

    // 좋아요 취소
    @DeleteMapping("/v1/articles-likes/articles/{articleId}/users/{userId}")
    public void unlike(
            @PathVariable("articleId") Long articleId,
            @PathVariable("userId") Long userId
    ) {
        articleLikeService.unlike(articleId, userId);
    }
}
```

- Test
```java
public class LikeApiTest {
    RestClient restClient = RestClient.create("http://localhost:9002");

    @Test
    void likeAndUnlikeTest() {
        Long articleId = 9999L;

        like(articleId, 1L);
        like(articleId, 2L);
        like(articleId, 3L);

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

    void like(Long articleId, Long userId) {
        restClient.post()
                .uri("/v1/article-likes/articles/{articleId}/users/{userId}", articleId, userId)
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
}
```
