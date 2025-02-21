### 8. 게시글 목록 최적화 전략 설계
___
- article Service 의 MySQL는 모든 게시글 목록 조회 트래픽을 모두 받는다.
  - index 를 이용하여 빠른 상태 이다.
- 성능을 높이기 위해서는 Redis를 이용하여 cache 를 고려해볼수 있다.

<br>

#### 게시글 목록 조회 최적화 전략
```text
- @Cacheable : 일반적으로 적용할 수 있는 캐시 전략
1. 캐시에서 key를 기반으로 데이터 조회
2. 캐시 유무에 따라 데이터가 있으면 캐시 데이터 응답, 없으면 원본 데이터 가져오고 캐시에 저장 후 응답

목록 데이터 조회 파라미터는 boardId, page, pageSize
위 파라미터가 캐시의 키, 조회된 게시글 목록이 값이 될텐데 단순히 키/값 저장 전략을 취하면, 캐시 효과??
게시글 작성/삭제되면, 해당 키로 만들어진 게시글 목록은 과거의 데이터가 된다.

게시글이 작성/삭제될 때마다, 새로운 목록을 보여주려면 캐시 만료가 필요 => 만료가 잦고 캐시의 히트율이 낮아진다.
만료를 늘리면 동기화가 안되있기 때문에 과거 데이터 노출

- 게시글 생성/삭제되면, 캐시에 미리 목록 데이터를 생성
- 게시글 조회 서비스는 이미 게시글 서비스에서 이벤트를 받고 있다.
=> Memory는 Disk에 비해 비싼 저장소

** 게시판의 사용 패턴
- 특정 게시판을 클릭하면, 게시판의 첫 페이지로 이동하여 최신글 목록이 조회
- 뒷 페이지로 명시적으로 이동하지 않는 이상 과거 데이터는 비교적 조회 횟수가 적어진다
==> 모든 데이터를 캐시할 필요는 없다.

- hot data = 자주 조회되는 데이터(최신의 글), cold data = 자주 조회되지 않는 데이터(과거의 글)
=> hot data 에만 캐시 적용, Redis는 최신 1000개의 목록만 캐시, 1000개 이후의 데이터는 게시글 서비스로 요청

- 게시글 조회 서비스는 kafka로부터 게시글 생성/삭제 이벤트 전달 받는다.
=> Redis에 게시판별 게시글 목록을 저장, sorted set 자료구조를 활용하여 최신순 정렬을 1000개까지 유지
data = article_id, score = article_id(생성 시간)
이렇게 하면 최신글 1000건 이내의 목록 데이터는 게시글 조회 서비스의 Redis에서 가져오고, 이후의 데이터는 게시글 서비스에서 가져올 수 있다.
```

<br>

### 9. 게시글 목록 최적화 전략 구현 - 리포지토리
___
- Redis에 게시글 Id만 따로 sortedSet으로 저장하는 repository
```java
@Repository
@RequiredArgsConstructor
public class ArticleIdListRepository {
    private final StringRedisTemplate redisTemplate;

    // article-read::board::{boardId}::article-list
    private static final String KEY_FORMAT = "article-read::board::%s::article-list";

    public void add(Long boardId, Long articleId, long limit) {
        redisTemplate.executePipelined((RedisCallback<?>) action -> {
            StringRedisConnection conn = (StringRedisConnection) action;
            String key = generateKey(boardId);
            // score 가 double값이라 toPaddedString 로 변경(articleId는 Long이라 데이터가 꼬일수 있따)
            // score가 고정값이면 value로 정렬상태가 만들어진다. 최신순정렬상태로 유지가능
            conn.zAdd(key, 0, toPaddedString(articleId));
            conn.zRemRange(key, 0, -limit - 1);    // 상위 1000개 유지
            return null;
        });
    }

    public void delete(Long boardId, Long articleId) {
        redisTemplate.opsForZSet().remove(generateKey(boardId), toPaddedString(articleId));
    }

    public List<Long> readAll(Long boardId, Long offset, Long limit) {
        //reverseRange 정렬된 상태로 조회
        return redisTemplate.opsForZSet()
                .reverseRange(generateKey(boardId), offset, offset + limit - 1)
                .stream().map(Long::valueOf).toList();
    }

    public List<Long> readAllInfiniteScroll(Long boardId, Long lastArticleId, Long limit) {
        // 스코어가 0으로 동일한거에대해 value 값으로 정렬이 되는데 정렬된 상태를 조회하려면 파라미터로 문자를 줘야한다. reverseRangeByLex 사용
        return redisTemplate.opsForZSet().reverseRangeByLex(
                generateKey(boardId),
                // 문자열상태로 데이터가 정렬된 상태로 들어온다. 6,5,4,3,2,1 일때 lastArticleId == null 일때 limit 개수만 가져온다.
                // ex) limit 3 => 6,5,4 이렇게되면 lastArticle = 4 , exclusive 4는 제외 => 3,2,1 이나온다.
                lastArticleId == null ?
                        Range.unbounded() : // 1페이지
                        Range.leftUnbounded(Range.Bound.exclusive(toPaddedString(lastArticleId))),
                Limit.limit().count(limit.intValue())
        ).stream().map(Long::valueOf).toList();
    }



    // long 값으로 받은 파라미터를 고정된 자릿수의 문자열로 바꿔준다.
    private String toPaddedString(Long articleId) {
        //1234 -> 0000000000000001234
        return "%019d".formatted(articleId);
    }

    private String generateKey(Long boardId) {
        return KEY_FORMAT.formatted(boardId);
    }
}
```
- 페이지 번호 방식에서는 게시글 개수도 같이 반환
```java
@Repository
@RequiredArgsConstructor
public class BoardArticleCountRepository {
    private final StringRedisTemplate redisTemplate;

    //article-read::board-article-count::board::{boardId}
    private static final String KEY_FORMAT = "article-read::board-article-count::board::%s";

    public void createOrUpdate(Long boardId, Long articleCount) {
        redisTemplate.opsForValue().set(generateKey(boardId), String.valueOf(articleCount));
    }
    
    public Long read(Long boardId) {
        String result = redisTemplate.opsForValue().get(generateKey(boardId));
        return result == null ? 0L : Long.valueOf(result);
    }
    
    private String generateKey(Long boardId) {
        return KEY_FORMAT.formatted(boardId);
    }
}
```

<br>

### 10. 게시글 목록 최적화 전략 구현 - 서비스 & 컨트롤러
___
- ArticleClient , 페이징, 무한스크롤 방식 전체 조회 , total count 
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
    public ArticlePageResponse readAll(Long boardId, Long page, Long pageSize) {
        try {
            return restClient.get()
                    .uri("/v1/articles?boardId=%s&page=%s&pageSize=%s".formatted(boardId,page,pageSize))
                    .retrieve()
                    .body(ArticlePageResponse.class);
        }catch (Exception e) {
            log.error("[ArticleClient.readAll] boardId={}, page={}, pageSize={}", boardId, page, pageSize, e);
            return ArticlePageResponse.EMPTY;
        }
    }

    public List<ArticleResponse> readAllInfiniteScroll(Long boardId, Long lastArticleId, Long pageSize) {
        try {
            return restClient.get()
                    .uri(
                            lastArticleId != null ?
                                    "/v1/articles/infinite-scroll?boardId=%s&lastArticleId=%s&pageSize=%s"
                                            .formatted(boardId, lastArticleId, pageSize) :
                                    "/v1/articles/infinite-scroll?boardId=%s&pageSize=%s"
                                            .formatted(boardId, pageSize)
                    )
                    .retrieve()
                    .body(new ParameterizedTypeReference<List<ArticleResponse>>() {});
        } catch (Exception e) {
            log.error("[ArticleClient.readAllInfiniteScroll] boardId={}, lastArticleId={}, pageSize={}", boardId, lastArticleId, pageSize, e);
            return List.of();
        }
    }

    public long count(Long boardId) {
        try {
            return restClient.get()
                    .uri("/v1/articles/boardId/{boardId}/count", boardId)
                    .retrieve()
                    .body(Long.class);
        } catch (Exception e) {
            log.error("[ArticleClient.count] boardId={}", boardId, e);
            return 0;
        }
    }

    // 게시글 목록도 Redis에 저장된게 없으면 원본 데이터 서버에서 가져옴 , 그걸위한 페이징, 카운트 메소드
    @Getter
    @NoArgsConstructor
    @AllArgsConstructor
    public static class ArticlePageResponse {
        private List<ArticleResponse> articles;
        private Long articleCount;

        public static ArticlePageResponse EMPTY = new ArticlePageResponse(List.of(), 0L);   // 빈 List 와 0으로 초기화  API호출에러시 반환

    }
}
```
- ArticleCreatedEventHandler 에 게시글 목록, 수 저장 로직 추가
```java
@Component
@RequiredArgsConstructor
public class ArticleCreatedEventHandler implements EventHandler<ArticleCreatedEventPayload> {
    private final ArticleIdListRepository articleIdListRepository;  // 게시글 목록 저장
    private final BoardArticleCountRepository boardArticleCountRepository;  // 게시글 수 저장
    private final ArticleQueryModelRepository articleQueryModelRepository;

    @Override
    public void handle(Event<ArticleCreatedEventPayload> event) {
        ArticleCreatedEventPayload payload = event.getPayload();
        articleQueryModelRepository.create(
                ArticleQueryModel.create(payload),
                Duration.ofDays(1)
        );
        articleIdListRepository.add(payload.getBoardId(), payload.getArticleId(), 1000L);
        boardArticleCountRepository.createOrUpdate(payload.getBoardId(), payload.getBoardArticleCount());
    }

    @Override
    public boolean supports(Event<ArticleCreatedEventPayload> event) {
        return EventType.ARTICLE_CREATED == event.getType();
    }
}

```
- ArticleDeletedEventHandler 에도 게시글 목록 , 수 삭제처리할때 필요한 로직 추가
```java
@Component
@RequiredArgsConstructor
public class ArticleDeletedEventHandler implements EventHandler<ArticleDeletedEventPayload> {
    private final ArticleIdListRepository articleIdListRepository;  // 게시글 목록 저장
    private final BoardArticleCountRepository boardArticleCountRepository;  // 게시글 수 저장
    private final ArticleQueryModelRepository articleQueryModelRepository;
    
    @Override
    public void handle(Event<ArticleDeletedEventPayload> event) {
        ArticleDeletedEventPayload payload = event.getPayload();
        articleIdListRepository.delete(payload.getBoardId(), payload.getArticleId());
        articleQueryModelRepository.delete(payload.getArticleId());
        boardArticleCountRepository.createOrUpdate(payload.getBoardId(), payload.getBoardArticleCount());
        // 게시글 목록 삭제 -> QueryModel 삭제 -> Count update 
        //query model 삭제되면 게시글은 삭제되었지만 목록에는 존재해서 사이에 조회를 하면 에러가 날수있다.
    }

    @Override
    public boolean supports(Event<ArticleDeletedEventPayload> event) {
        return EventType.ARTICLE_DELETED == event.getType();
    }
}
```
- page 목록 방식 response
```java
@Getter
public class ArticleReadPageResponse {
    private List<ArticleReadResponse> articles; // 게시글 목록
    private Long articleCount;
    
    public static ArticleReadPageResponse of(List<ArticleReadResponse> articles, Long articleCount) {
        ArticleReadPageResponse response = new ArticleReadPageResponse();
        response.articles = articles;
        response.articleCount = articleCount;
        return response;
    }
}
```
- ArticleQueryModelRepository 에 readAll 메서드 생성
```java
@Repository
@RequiredArgsConstructor
public class ArticleQueryModelRepository {
    private final StringRedisTemplate redisTemplate;

    private String generateKey(ArticleQueryModel articleQueryModel) {
        return generateKey(articleQueryModel.getArticleId());
    }

    private String generateKey(Long articleId) {
        return KEY_FORMAT.formatted(articleId);
    }

    public Map<Long, ArticleQueryModel> readAll(List<Long> articleIds) {
        List<String> keyList = articleIds.stream().map(this::generateKey).toList();
        // multiget : 리스트를 전달해서 여러개의 데이터 한번에 조회
        return redisTemplate.opsForValue().multiGet(keyList).stream()
                .map(json -> DataSerializer.deserialize(json, ArticleQueryModel.class))
                .collect(toMap(ArticleQueryModel::getArticleId, identity()));
    }
}
```
- Service 목록조회 메서드 생성
```java
@Slf4j
@Service
@RequiredArgsConstructor
public class ArticleReadService {
    // 데이터 없을때 command 서버로 요청 해야하니 필요 client 주입
    private final ArticleClient articleClient;
    private final ArticleQueryModelRepository articleQueryModelRepository;
    private final ArticleIdListRepository articleIdListRepository;  // 게시글 목록 저장
    private final BoardArticleCountRepository boardArticleCountRepository;  // 게시글 수 저장


    // 페이지 번호 방식
    public ArticleReadPageResponse readAll(Long boardId, Long page, Long pageSize) {
        return ArticleReadPageResponse.of(
                readAll(
                        readAllArticleIds(boardId, page, pageSize)
                ),
                count(boardId)
        );
    }

    // articleids 를 받아서 ArticleReadResponse 로 변환해서 반환
    private List<ArticleReadResponse> readAll(List<Long> articleIds) {
        Map<Long, ArticleQueryModel> articleQueryModelMap = articleQueryModelRepository.readAll(articleIds);
        return articleIds.stream()
                .map(articleId -> articleQueryModelMap.containsKey(articleId) ?
                        articleQueryModelMap.get(articleId) :
                        fetch(articleId).orElse(null))  //원본데이터
                .filter(Objects::nonNull)
                .map(articleQueryModel ->
                        ArticleReadResponse.from(
                                articleQueryModel,
                                viewClient.count(articleQueryModel.getArticleId())
                        ))
                .toList();
    }

    private List<Long> readAllArticleIds(Long boardId, Long page, Long pageSize) {
        List<Long> articleIds = articleIdListRepository.readAll(boardId, (page - 1) * pageSize, pageSize);
        if(pageSize == articleIds.size()) {   // 게시글 목록이 전부가 레디스에 저장되있다.
            log.info("[ArticleReadService.readAllArticleIds] return redis data");
            return articleIds;
        }
        log.info("[ArticleReadService.readAllArticleIds] return origin data");
        // 원본데이터
        return articleClient.readAll(boardId, page, pageSize).getArticles().stream()
                .map(ArticleClient.ArticleResponse::getArticleId)
                .toList();
    }


    private long count(Long boardId) {
        Long result = boardArticleCountRepository.read(boardId);
        if(result != null) {
            return result;
        }
        long count = articleClient.count(boardId);
        boardArticleCountRepository.createOrUpdate(boardId, count);
        return count;
    }

    private List<ArticleReadResponse> readAllInfiniteScroll(Long boardId, Long lastArticleId, Long pageSize) {
        return readAll(
                readAllInfiniteScrollArticleIds(boardId, lastArticleId, pageSize)
        );
    }

    private List<Long> readAllInfiniteScrollArticleIds(Long boardId, Long lastArticleId, Long pageSize) {
        List<Long> articleIds = articleIdListRepository.readAllInfiniteScroll(boardId, lastArticleId, pageSize);
        if(pageSize == articleIds.size()) {
            log.info("[ArticleReadService.readAllInfiniteScrollArticleIds] return redis data");
            return articleIds;
        }
        log.info("[ArticleReadService.readAllInfiniteScrollArticleIds] return origin data");
        return articleClient.readAllInfiniteScroll(boardId, lastArticleId, pageSize).stream()
                .map(ArticleClient.ArticleResponse::getArticleId)
                .toList();
    }
}
```
- controller 목록 조회 추가
```java
@RestController
@RequiredArgsConstructor
public class ArticleReadController {
    private final ArticleReadService articleReadService;

    @GetMapping("/v1/articles")
    public ArticleReadPageResponse readAll(
            @RequestParam("boardId") Long boardId,
            @RequestParam("page") Long page,
            @RequestParam("pageSize") Long pageSize
    ) {
        return articleReadService.readAll(boardId, page, pageSize);
    }

    @GetMapping("/v1/articles/infinite-scroll")
    public List<ArticleReadResponse> readAllInfiniteScroll(
            @RequestParam("boardId") Long boardId,
            @RequestParam(value = "lastArticleId", required = false) Long lastArticleId,
            @RequestParam("pageSize") Long pageSize
    ) {
        return articleReadService.readAllInfiniteScroll(boardId, lastArticleId, pageSize);
    }
}
```

<br>

### 11. 게시글 목록 최적화 전략 구현 - 테스트
___
- redis 데이터와 원본 데이터 비교
    - 1000페이지 전은 redis에 저장된 데이터 
    - 1페이지 일때는 response1은 redis, response2 는 원본데이터다, 3000페이지 일때는 둘다 원본데이터
```java
public class ArticleReadApiTest {
    RestClient articleReadRestClient = RestClient.create("http://localhost:9005");
    RestClient articleRestClient = RestClient.create("http://localhost:9000");


    @Test
    void readAllTest() {
        ArticleReadPageResponse response1 = articleReadRestClient.get()
                .uri("/v1/articles?boardId=%s&page=%s&pageSize=%s".formatted(1L, 3000L, 5))
                .retrieve()
                .body(ArticleReadPageResponse.class);
        System.out.println("response1.getArticleCount = " + response1.getArticleCount());
        for (ArticleReadResponse article : response1.getArticles()) {
            System.out.println("article.getArticleId() = " + article.getArticleId());
        }

        ArticleReadPageResponse response2 = articleRestClient.get()
                .uri("/v1/articles?boardId=%s&page=%s&pageSize=%s".formatted(1L, 3000L, 5))
                .retrieve()
                .body(ArticleReadPageResponse.class);
        System.out.println("response2.getArticleCount = " + response2.getArticleCount());
        for (ArticleReadResponse article : response2.getArticles()) {
            System.out.println("article.getArticleId() = " + article.getArticleId());
        }
    }


    @Test
    void readAllInfiniteScrollTest() {
        List<ArticleReadResponse> responses1 = articleReadRestClient.get()
//                .uri("/v1/articles/infinite-scroll?boardId=%s&pageSize=%s".formatted(1L, 5L))
                .uri("/v1/articles/infinite-scroll?boardId=%s&pageSize=%s&lastArticleId=%s".formatted(1L, 5L, 151172446771970048L))
                .retrieve()
                .body(new ParameterizedTypeReference<List<ArticleReadResponse>>() {
                });
        for (ArticleReadResponse response : responses1) {
            System.out.println("response.getArticleId() = " + response.getArticleId());
        }


        List<ArticleReadResponse> responses2 = articleRestClient.get()
//                .uri("/v1/articles/infinite-scroll?boardId=%s&pageSize=%s".formatted(1L, 5L))
                .uri("/v1/articles/infinite-scroll?boardId=%s&pageSize=%s&lastArticleId=%s".formatted(1L, 5L, 151172446771970048L))
                .retrieve()
                .body(new ParameterizedTypeReference<List<ArticleReadResponse>>() {
                });

        for (ArticleReadResponse response : responses2) {
            System.out.println("response.getArticleId() = " + response.getArticleId());
        }
    }
}
```
