## 게시글 조회 최적화

### 1. CQRS의 필요성
___
- 게시글 단건 조회 최적화
- 게시글 목록 조회 최적화
- 캐시 최적화 (조회에 최적화된 캐시)
- 게시판 서비스의 특성
  - 읽기 트래픽이 쓰기 트래픽보다 많다.
  - 사용자에게 게시글, 좋아요 수, 댓글 수, 조회 수, 작성자 정보등을 함께 보여준다.
  - 데이터가 분산되어 있는 상황에서 Client는 어떻게 조회

#### 게시글 조회 
```text
* 게시글 조회 방법
- Client는 Article Service로 게시글을 요청 -> Article Service는 Comment/Like/View Service로 데이터를 요청

* 문제점
- Article Service는 읽기 쓰기가 함께 처리된다.(트래픽 : 읽기 > 쓰기)
읽기 트래픽으로 서버를 증설해야한다면, 쓰기 작업에 대한 리소스도 함께 확장된다.
- 게시글 조회 요구사항으로 인해 각 서비스에 의존성생김
각 Service와 Article Service 간에는 양방향 참조가 이루어지고 있다. => 각 마이크로서비스 간에 순환 참조가 발생하는 구조
Comment/Like/View 참조키로 Article의 정보를 가지고 있다.(FK=articleId)
지금까지 개발을 진행하면서 데이터 검증 등에 대해 직접적으로 신경쓰진 않았지만,
각 Service는 데이터의 무결성을 검증하기 위해, 게시글 유효성 확인은 필요하다.
무결성을 검증하려면 게시글에 종속된 데이터이기 때문에 Article Service로 게시글 데이터 요청이 필요
Comment/Like/View Service는 Article Service에 의존성을 가지고 있는 것이다.
따라서, 단순히 Article Service가 각 서비스에 데이터를 요청하여 조합하면, 마이크로서비스 간에 순환 참조가 발생하게 되는 것이다.
=> 각 마이크로서비스는 독립적으로 배포 및 유지보수 될 수 없고, 서로 간에 장애가 전파될 수 있으며, 테스트에도 어려움이 생길 수 있다.

* 해결 방법
게시글 조회를 위한 서비스 생성, Client는 게시글 조회 서비스로 데이터 요청, 게시글 조회 서비스는 각 마이크로서비스에 개별 데이터 요청
양방향 의존성을 끊어냄으로써 순환참조 문제가 해결되기 때문에, 각 마이크로서비스는 다시 독립적으로 관리될 수 있다.
또, 게시글 조회를 위한 마이크로서비스이기 때문에, 읽기 트래픽에 대해서만 독립적으로 확장될 수 있다.

=> 각 데이터가 여러 서비스 또는 데이터베이스에 분산되어 있기 때문에,
데이터를 요청하기 위한 네트워크 비용, 각 서비스에 부하 전파, 데이터 조합 및 질의 비용 증가
=> CQRS 를 이용하여 해결
```

<br>

### 2. CQRS 개념 & 게시글 조회 단건 최적화 전략 설계
___
- CQRS(Command Query Responsibility Segregation, Command와 Query의 책임을 분리)
  - 데이터에 대한 변경(Command)과 조회(Query) 작업을 구분하는 패턴
  - 클래스 레벨에서, 패키지 레벨에서, 서비스 레벨에서, 데이터 저장소 레벨에서 이러한 책임 분리가 일어날 수 있다.

<br>

#### CQRS
```text
Command - CUD, Query - R
게시글/좋아요/댓글 Command는 기존의 게시글/댓글/좋아요 서비스
게시글/좋아요/댓글 Query는 신규로 만드는 게시글 조회 서비스
마이크로서비스 레벨에서 분리

* 게시글 조회 서비스는 Command로부터 게시글/댓글/좋아요 데이터를 어떻게 가져올 수 있을까?
Command 서버에서 실시간으로 가져오면, 결국 Command 서버로 부하가 전파되는 문제가 있다.
=> 게시글 조회 서비스에 자체 데이터베이스를 구축하고 데이터를 쌓는다. => 데이터 저장소 레벨에서 분리

* Query 데이터베이스는 데이터의 실시간 변경 사항을 어떻게 가져올 수 있을까?
API로 주기적으로 변경 사항을 polling 해올 수도 있고, Message Broker를 활용할 수도 있다.
이미 구축되있는 kafka를 활용, 이벤트는 각 Producer에서 이미 Kafka로 전송되고 있기 떄문에 
게시글 조회 서비스는 Consumer Group만 달리해서 그대로 활용 

* 게시글 조회 서비스는 게시글/댓글/좋아요 서비스의 데이터 모델과 동일하게 관리해야 할까?
분산된 데이터를 어차피 조합하여 질의해야 하므로,
Query에서는 조인 비용을 줄이고, 조회 최적화를 위해 필요한 데이터가 비정규화된 Query 모델을 만든다.
Query Model = 게시글 + 댓글 수 + 좋아요 수 => Query Model 단건만 조회하면, 필요한 데이터를 모두 조회할 수 있다.
=> Redis 사용

게시판 시스템 특성 상, 최신글이 조회되는 경우가 많다. => TTL(1일)을 설정하여, 24시간 이내의 최신글만 Redis에 보관

* Query 데이터베이스(Redis)에 데이터가 만료되었다면 어떻게 할까?
Command 서버로 원본 데이터를 다시 요청하여 Query Model을 만들어볼 수 있다.
만료된 데이터만 간헐적으로 요청하므로, 이러한 트래픽은 크지 않을 것이다.
또한, Query 데이터베이스의 장애 상황, 이벤트 누락, 데이터 유실 등 상황에, 원본 데이터 서버로 질의하여 가용성을 높일 수 있다.
```

<br>

#### 게시글 단건 조회 최적화 전략
```text
* 조회수는 Query Model에 왜 비정규화되지 않았을까?
조회수는 읽기 트래픽에 따라서 올라가는 데이터 특성 => 읽기 트래픽에 의해 쓰기 트래픽도 발생
조회수의 변경마다 Query Model을 만드는게 오히려 비효율
조회수는 Redis에 저장 되어있는 상태 
조회수 이벤트는 백업 시점(100개 단위)에만 전송(실시간 데이터가 아님)
=> 조회수는 조회수 서비스로 직접 요청
```

<br>

### 3. 게시글 조회 서비스 설정 & Client & QueryModel
___
- article-read 프로젝트 dependencies 추가 , redis, kafka, event, data-serializer 모듈
- application.yml : redis, kafka 설정정보 + 게시글/댓글/좋아요/조회수 서비스 호출 정보 추가
- command 서버로 데이터를 요청하기 위한 client class
- ArticleClient : 게시글 서비스에서 게시글  받아옴
```java
@Slf4j
@Component
@RequiredArgsConstructor
public class ArticleClient {
    private RestClient restClient;

    @Value("${endpoints.hello-board-article-service.url}")
    private String articleServiceUrl;

    @PostConstruct
    public void initRestClient() {
        restClient = RestClient.create(articleServiceUrl);
    }

    public Optional<ArticleResponse> read(Long articleId) {
        try {
            ArticleResponse articleResponse = restClient.get()
                    .uri("/v1/articles/{articleId}", articleId)
                    .retrieve()
                    .body(ArticleResponse.class);
            return Optional.ofNullable(articleResponse);
        }catch (Exception e) {
            log.error("[ArticleClient.read] articleId={}", articleId, e);
            return Optional.empty();
        }
    }

    @Getter
    public static class ArticleResponse {       // ArticleClient 가 게시글 서비스에서 가져온 데이터를 담아주는 클래스
        private Long articleId;
        private String title;
        private String content;
        private Long boardId;
        private Long writerId;
        private LocalDateTime createdAt;
        private LocalDateTime modifiedAt;
    }
}
```
- CommentClient : 댓글 수 만 가져옴, ViewClient, LikeClient 도 생성(CommentClient 와 거의 비슷)
```java
@Slf4j
@Component
@RequiredArgsConstructor
public class CommentClient {
    private RestClient restClient;

    @Value("${endpoints.hello-board-comment-service.url}")
    private String commentServiceUrl;

    @PostConstruct
    public void initRestClient() {
        restClient = RestClient.create(commentServiceUrl);
    }

    public long count(Long articleId) {
        try {
            return restClient.get()
                    .uri("/v2/comments/articles/{articleId}/count", articleId)
                    .retrieve()
                    .body(Long.class);
        }catch (Exception e) {
            log.error("[CommentClient.count] articleId={}", articleId, e);
            return 0;
        }
    }
}
```
- queryModel 생성 class
```java
@Getter
public class ArticleQueryModel {
    private Long articleId;
    private String title;
    private String content;
    private Long boardId;
    private Long writerId;
    private LocalDateTime createdAt;
    private LocalDateTime modifiedAt;
    
    private Long articleCommentCount;
    private Long articleLikeCount;
    
    public static ArticleQueryModel create(ArticleCreatedEventPayload payload) {
        ArticleQueryModel articleQueryModel = new ArticleQueryModel();
        articleQueryModel.articleId = payload.getArticleId();
        articleQueryModel.title = payload.getTitle();
        articleQueryModel.content = payload.getContent();
        articleQueryModel.boardId = payload.getBoardId();
        articleQueryModel.writerId = payload.getWriterId();
        articleQueryModel.createdAt = payload.getCreatedAt();
        articleQueryModel.modifiedAt = payload.getModifiedAt();
        articleQueryModel.articleCommentCount = 0L; // 처음작성시 0개 
        articleQueryModel.articleLikeCount = 0L;
        return articleQueryModel;
    }
    
    // 이벤트 뿐만아니라 Redis에 데이터가 없으면 client class로 command 서버에 데이터 요청, 
    // command 서버에서 받아온 데이터로 ArticleQuery 만드는 메소드
    public static ArticleQueryModel create(ArticleClient.ArticleResponse article, Long commentCount, Long likeCount) {
        ArticleQueryModel articleQueryModel = new ArticleQueryModel();
        articleQueryModel.articleId = article.getArticleId();
        articleQueryModel.title = article.getTitle();
        articleQueryModel.content = article.getContent();
        articleQueryModel.boardId = article.getBoardId();
        articleQueryModel.writerId = article.getWriterId();
        articleQueryModel.createdAt = article.getCreatedAt();
        articleQueryModel.modifiedAt = article.getModifiedAt();
        articleQueryModel.articleCommentCount = commentCount;
        articleQueryModel.articleLikeCount = likeCount;
        return articleQueryModel;
    }

    // update 메소드, 게시글 수정, 댓글 생성/삭제, 좋아요 생성/삭제 발생시 데이터 업데이트
    public void updateBy(CommentCreatedEventPayload payload) {
        this.articleCommentCount = payload.getArticleCommentCount();
    }

    public void updateBy(CommentDeletedEventPayload payload) {
        this.articleCommentCount = payload.getArticleCommentCount();
    }

    public void updateBy(ArticleLikedEventPayload payload) {
        this.articleLikeCount = payload.getArticleLikeCount();
    }
    
    public void updateBy(ArticleUnikedEventPayload payload) {
        this.articleLikeCount = payload.getArticleLikeCount();
    }

    public void updateBy(ArticleUpdatedEventPayload payload) {
        this.title = payload.getTitle();
        this.content = payload.getContent();
        this.boardId = payload.getBoardId();
        this.writerId = payload.getWriterId();
        this.createdAt = payload.getCreatedAt();
        this.modifiedAt = payload.getModifiedAt();
    }
}
```
- Redis 에 저장하는 repository
```java
@Repository
@RequiredArgsConstructor
public class ArticleQueryModelRepository {
    private final StringRedisTemplate redisTemplate;

    //article-read::article::{articleId}
    private static final String KEY_FORMAT = "article-read::article::%s";


    public void create(ArticleQueryModel articleQueryModel, Duration ttl) {
        redisTemplate.opsForValue()
                .set(generateKey(articleQueryModel), DataSerializer.serialize(articleQueryModel), ttl);
    }

    public void update(ArticleQueryModel articleQueryModel) {
        redisTemplate.opsForValue().setIfPresent(generateKey(articleQueryModel), DataSerializer.serialize(articleQueryModel));
    }

    public void delete(Long articleId) {
        redisTemplate.delete(generateKey(articleId));
    }

    public Optional<ArticleQueryModel> read(Long articleId) {
        return Optional.ofNullable(
                redisTemplate.opsForValue().get(generateKey(articleId))
        ).map(json -> DataSerializer.deserialize(json, ArticleQueryModel.class));
    }

    private String generateKey(ArticleQueryModel articleQueryModel) {
        return generateKey(articleQueryModel.getArticleId());
    }

    private String generateKey(Long articleId) {
        return KEY_FORMAT.formatted(articleId);
    }
}
```
