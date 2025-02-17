### 4. Event&직렬화 모듈 구현
___
- common - data-serializer 프로젝트 생성
  - 이벤트 주고 받는 데이터를 직렬화, 역직렬화
- objectmapper 사용을 위한 의존성 추가
```gradle
dependencies {
    implementation 'com.fasterxml.jackson.core:jackson-databind'
    implementation 'com.fasterxml.jackson.datatype:jackson-datatype-jsr310'
}
```
- 직렬화 UtilClass
```java
@Slf4j
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public final class dataSerializer {
    private static final ObjectMapper objectMapper = initialize();

    private static ObjectMapper initialize() {
        return new ObjectMapper()
                .registerModule(new JavaTimeModule())   // 시간관련해서 직렬화 처리하려면 Java time 모듈 등록해야함
                .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);    // 역직렬화 할때 없는 필드가 있으면 에러가날 수 있는 설정, false는 에러안남
    }

    // String 타입으로 데이터를 받아서 클래스 타입으로 역직렬화 해주는 메서드
    public static <T> T deserialize(String data, Class<T> clazz) {
        try {
            return objectMapper.readValue(data, clazz);
        } catch (JsonProcessingException e) {
            log.error("[DataSerializer.deserialize] data={}, clazz={}", data, clazz, e);
            return null;
        }
    }

    // Object 로 데이터를 받아서 타입 변환
    public static <T> T deserialize(Object data, Class<T> clazz) {
        return objectMapper.convertValue(data, clazz);
    }

    // object 받아서 json 으로 변환
    public static String serialize(Object object) {
        try {
            return objectMapper.writeValueAsString(object);
        } catch (JsonProcessingException e) {
            log.error("[DataSerializer.serialize] object={}", object, e);
            return null;
        }
    }
}
```

<br>

- common - event 프로젝트 생성
  - Producer , Consumer 는 이벤트로 통신, 이벤트에 대한 정의 
- event.java
```java
@Getter
public class Event <T extends EventPayload> {    // event 통신을 위한 class

    private Long eventId;   // 이벤트에 대하여 고유한 아이디를 가져서 이벤트를 식별할 수 있도록 한다.
    private EventType type;
    private T payload; // 어떤 데이터를 가지고 있는지

    public static Event<EventPayload> of(Long eventId, EventType type, EventPayload payload) {
        Event<EventPayload> event = new Event<>();
        event.eventId = eventId;
        event.type = type;
        event.payload = payload;
        return event;
    }

    // 이벤트를 kafka 로 전달할 때 json 으로 변환하고 다시 역직렬화
    // event 클래스를 json 문자열로 변경해주는 메서드
    public String toJson() {
        return DataSerializer.serialize(this);
    }

    // json 을 받아서 이벤트로 반환해주는 메서드
    public static Event<EventPayload> fromJson(String json) {
        EventRaw eventRaw = DataSerializer.deserialize(json, EventRaw.class);
        if(eventRaw == null) {
            return null;
        }
        Event<EventPayload> event = new Event<>();
        event.eventId = eventRaw.getEventId();
        event.type = EventType.from(eventRaw.getType());
        event.payload = DataSerializer.deserialize(eventRaw.getPayload(), event.type.getPayloadClass());
        return event;
    }

    // type 에 따라서 payload 가 어떤 클래스 타입인지 다다르다. 그걸 알기위해 처음에 일단 스트링이랑 오브젝트 타입으로 받아서 처리
    @Getter
    private static class EventRaw{
        private Long eventId;
        private String type;
        private Object payload;
    }
}
```
- eventType
```java
@Slf4j
@Getter
@RequiredArgsConstructor
public enum EventType {
    ARTICLE_CREATED(ArticleCreatedEventPayload.class, Topic.BOARD_ARTICLE),
    ARTICLE_UPDATED(ArticleUpdatedEventPayload.class, Topic.BOARD_ARTICLE),
    ARTICLE_DELETED(ArticleDeletedEventPayload.class, Topic.BOARD_ARTICLE),
    COMMENT_CREATED(CommentCreatedEventPayload.class, Topic.BOARD_COMMENT),
    COMMENT_DELETED(CommentDeletedEventPayload.class, Topic.BOARD_COMMENT),
    ARTICLE_LIKED(ArticleLikedEventPayload.class, Topic.BOARD_LIKE),
    ARTICLE_UNLIKED(ArticleUnikedEventPayload.class, Topic.BOARD_LIKE),
    ARTICLE_VIEWED(ArticleViewedEventPayload.class, Topic.BOARD_VIEW);

    // 이이벤트들이 어떤 payload 타입인지 그 타입을 가질수 있다.
    private final Class<? extends EventPayload> payloadClass;
    private final String topic;

    // 이벤트 타입 문자열로 받아서 enum 타입으로 변환해주는 메소드
    public static EventType from(String type) {
        try {
            return valueOf(type);
        }catch (Exception e) {
            log.error("[EventType.from] type={}", type, e);
            return null;
        }
    }

    // kafka topic
    public static class Topic {
        public static final String BOARD_ARTICLE = "board-article";
        public static final String BOARD_COMMENT = "board-comment";
        public static final String BOARD_LIKE = "board-like";
        public static final String BOARD_VIEW = "board-view";
    }
}
```
- eventPayload.java
```java
public interface EventPayload {
}
```
- eventType 에 정의했던 enum 데이터 class들 정의 
```java
@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ArticleCreatedEventPayload implements EventPayload {
    private Long articleId;
    private String title;
    private String content;
    private Long boardId;
    private Long writerId;
    private LocalDateTime createdAt;
    private LocalDateTime modifiedAt;
    private Long boardArticleCount;
}
```
```text
class EventTest {
    @Test
    void serde() {  // 직렬화 역직렬화 테스트
        //given
        //payload 생성
        ArticleCreatedEventPayload payload = ArticleCreatedEventPayload.builder()
                .articleId(1L)
                .title("title")
                .content("content")
                .boardId(1L)
                .writerId(1L)
                .createdAt(LocalDateTime.now())
                .modifiedAt(LocalDateTime.now())
                .boardArticleCount(25L)
                .build();

        // 이벤트 생성
        Event<EventPayload> event = Event.of(
                1234L,
                EventType.ARTICLE_CREATED,
                payload
        );

        //event -> json
        String json = event.toJson();
        System.out.println("json = " + json);

        //when
        //json -> 객체
        Event<EventPayload> result = Event.fromJson(json);

        //then
        assertThat(result.getEventId()).isEqualTo(event.getEventId());
        assertThat(result.getType()).isEqualTo(event.getType());
        assertThat(result.getPayload()).isInstanceOf(payload.getClass());
        ArticleCreatedEventPayload resultPayload = (ArticleCreatedEventPayload) result.getPayload();

        assertThat(resultPayload.getArticleId()).isEqualTo(payload.getArticleId());
        assertThat(resultPayload.getTitle()).isEqualTo(payload.getTitle());
        assertThat(resultPayload.getCreatedAt()).isEqualTo(payload.getCreatedAt());
    }
}
```

<br>

### 5. 인기글 Consumer 구현 - 설정 및 인기글 리포지토리
___
- 인기글 서비스 구현
- dependency : redis, kafka, event 모듈 추가
- yml : redis, kafka, 게시글 서비스 호출정보 추가
- KafkaConfig.java : Kafka 소비자(Consumer)를 설정
```java
@Configuration
public class KafkaConfig {
    @Bean
    ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory(
            ConsumerFactory<String, String> consumerFactory
    ) {
        ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL);
        return factory;
    }
}
```
- ArticleClient.java :  게시글 서비스로 게시글 원본 정보 호출
```java

@Slf4j
@Component
@RequiredArgsConstructor
public class ArticleClient {
    private RestClient restClient;
    @Value("${endpoints.hello-board-article-service.url}")
    private String articleServiceUrl;

    @PostConstruct  // 빈이 생성될때 restClient 초기화
    void initRestClient() {
        restClient = RestClient.create(articleServiceUrl);
    }

    // 게시글 조회
    public ArticleResponse read(Long articleId) {
        try {
            return restClient.get()
                    .uri("/v1/articles/{articleId}", articleId)
                    .retrieve()
                    .body(ArticleResponse.class);
        }catch (Exception e) {
            log.error("[ArticleClient.read] articleId={}", articleId, e);
            return null;
        }
    }

    // 응답 클래스
    @Getter
    public static class ArticleResponse {
        private Long articleId;
        private String title;
        private LocalDateTime createdAt;
    }

}
```
- repository : 인기글 데이터 redis 저장
```java
@Slf4j
@Repository
@RequiredArgsConstructor
public class HotArticleListRepository {
    private final StringRedisTemplate redisTemplate;

    // SortedSet 으로 하나의 키에 여러개의 게시글 아이디와 스코어를 저장
    // hot-article::list::{yyyyMMdd}
    private static final String KEY_FORMAT = "hot-article-list:%s";

    private static final DateTimeFormatter TIME_FORMATTER = DateTimeFormatter.ofPattern("yyyyMMdd");

    public void add(Long articleId, LocalDateTime time, Long score, Long limit, Duration ttl) {
        // executePipelined : redis쪽으로 네트워크 통신 한번만 연결하면 여러개의 연산을 한번에 수행
        redisTemplate.executePipelined((RedisCallback<?>) action -> {
            StringRedisConnection conn = (StringRedisConnection) action;
            String key = generateKey(time); // 키생성
            conn.zAdd(key, score, String.valueOf(articleId));   // sortedSet 사용
            conn.zRemRange(key, 0, - limit - 1); // 상위 limit 갯수만큼 sortedSet 데이터를 유지할 수 있다.
            conn.expire(key, ttl.toSeconds());
            return null;
        });
    }

    // 게시글 삭제시 인기글에서도 사라져야하는것 처리
    public void remove(Long articleId, LocalDateTime time) {
      redisTemplate.opsForZSet().remove(generateKey(time), String.valueOf(articleId));
    }
  
    private String generateKey(LocalDateTime time) {
        return generateKey(TIME_FORMATTER.format(time));
    }

    private String generateKey(String dateStr) {
        return KEY_FORMAT.formatted(dateStr);
    }

  public List<Long> readAll(String dateStr) {
    return redisTemplate.opsForZSet()
            .reverseRangeWithScores(generateKey(dateStr), 0, -1).stream()
            .peek(tuple ->
                    log.info("[HotArticleListRepository.readAll] articleId={}, score={}", tuple.getValue(), tuple.getScore()))
            .map(ZSetOperations.TypedTuple::getValue)// TypedTuple<String>에서 articleId 만 가져옴
            .map(Long::valueOf) // String -> Long
            .toList();
  }
}
```
- test
```java
@SpringBootTest
class HotArticleListRepositoryTest {
    @Autowired
    HotArticleListRepository hotArticleListRepository;

    @Test
    void addTest() throws InterruptedException {
        // given
        LocalDateTime time = LocalDateTime.of(2024, 7, 23, 0, 0);
        long limit = 3; // 인기글 3건 제한

        // when
        hotArticleListRepository.add(1L, time, 2L, limit, Duration.ofSeconds(3));
        hotArticleListRepository.add(2L, time, 3L, limit, Duration.ofSeconds(3));
        hotArticleListRepository.add(3L, time, 1L, limit, Duration.ofSeconds(3));
        hotArticleListRepository.add(4L, time, 5L, limit, Duration.ofSeconds(3));
        hotArticleListRepository.add(5L, time, 4L, limit, Duration.ofSeconds(3));

        // then
        List<Long> articleIds = hotArticleListRepository.readAll("20240723");

        assertThat(articleIds).hasSize(Long.valueOf(limit).intValue());
        assertThat(articleIds.get(0)).isEqualTo(4L);
        assertThat(articleIds.get(1)).isEqualTo(5L);
        assertThat(articleIds.get(2)).isEqualTo(2L);

        TimeUnit.SECONDS.sleep(5);  // TTL 3초라 5초후에는 다지워진다.

        assertThat(hotArticleListRepository.readAll("20240723")).isEmpty();
    }
}
```

<br>

### 6. 인기글 Consumer 구현 - 이벤트 데이터 리포지토리
___
- 댓글 수 저장 리포지토리
```java
/**
 * 데이터들을 그 인기글 서비스의 책임으로 보고
 * 자체적으로 인기글이 만들어지는동안 여러개의 데이터들을 가지고 있는다.
 * 가지고 있지 않으면 댓글서비스로 조회하기 위해서 계속 호출이 필요
 * */
@Repository
@RequiredArgsConstructor
public class ArticleCommentCountRepository {
    private final StringRedisTemplate redisTemplate;

    //hot-article::article::{articleId}::comment-count
    private static final String KEY_FORMAT = "hot-article::article::%s::comment-count";

    private void createOrUpdate(Long articleId, String commentCount, Duration ttl) {
        //인기글이 선정될때까지만 가지고 있으면 되니까 시간만료
        redisTemplate.opsForValue().set(generateKey(articleId), commentCount, ttl);// set 데이터가 있으면 update 없으면 create
    }

    //조회
    public Long read(Long articleId) {
        String result = redisTemplate.opsForValue().get(generateKey(articleId));
        return result == null ? 0L : Long.valueOf(result);
    }

    private String generateKey(Long articleId) {
        return KEY_FORMAT.formatted(articleId);
    }
}
```
- 좋아요 수 저장 리포지토리
```java
@Repository
@RequiredArgsConstructor
public class ArticleLikeCountRepository {
    private final StringRedisTemplate redisTemplate;

    //hot-article::article::{articleId}::like-count
    private static final String KEY_FORMAT = "hot-article::article::%s::like-count";

    private void createOrUpdate(Long articleId, Long likeCount, Duration ttl) {
        redisTemplate.opsForValue().set(generateKey(articleId), String.valueOf(likeCount), ttl);
    }

    public Long read(Long articleId) {
        String result = redisTemplate.opsForValue().get(generateKey(articleId));
        return result == null ? 0L : Long.valueOf(result);
    }

    private String generateKey(Long articleId) {
        return KEY_FORMAT.formatted(articleId);
    }
}
```
- 조회 수 저장 리포지토리
```java
@Repository
@RequiredArgsConstructor
public class ArticleViewCountRepository {
    private final StringRedisTemplate redisTemplate;

    // hot-article::article::{articleId}::view-count
    private static final String KEY_FORMAT = "hot-article::article::%s::view-count";

    public void createOrUpdate(Long articleId, Long viewCount, Duration ttl) {
        redisTemplate.opsForValue().set(generateKey(articleId), String.valueOf(viewCount), ttl);
    }

    public Long read(Long articleId) {
        String result = redisTemplate.opsForValue().get(generateKey(articleId));
        return result == null ? 0L : Long.valueOf(result);
    }

    private String generateKey(Long articleId) {
        return KEY_FORMAT.formatted(articleId);
    }
}
```
- article 생성시간 저장 => 좋아요, 인기글 선정하는게 오늘 생성된 게시글이여야 한다.
  - 좋아요 이벤트가 왔는데, 이 이벤트에 대한 게시글이 오늘 게시글인지 확인하려면, 게시글 서비스에 조회가 필요
  - 하지만 게시글 생성 시간을 저장하고 있으면, 오늘 게시글인지 게시글 서비스를 호출하지않아도 바로 알 수 있다.
```java
@Repository
@RequiredArgsConstructor
public class ArticleCreatedTimeRepository {
    private final StringRedisTemplate redisTemplate;

    //hot-article::article::{articleId}::created-time
    private static final String KEY_FORMAT = "hot-article::article::%s::created-time";

    public void createOrUpdate(Long articleId, LocalDateTime createdAt, Duration ttl) {
        redisTemplate.opsForValue().set(
                generateKey(articleId),
                String.valueOf(createdAt.toInstant(ZoneOffset.UTC).toEpochMilli()),
                ttl
        );
    }
    
    public void delete(Long articleId) {
        redisTemplate.delete(generateKey(articleId));
    }
    
    public LocalDateTime read(Long articleId) {
        String result = redisTemplate.opsForValue().get(generateKey(articleId));
        if (result == null) {
            return null;
        }
        return LocalDateTime.ofInstant(
                Instant.ofEpochMilli(Long.valueOf(result)), ZoneOffset.UTC);
    }
    
    private String generateKey(Long articleId) {
        return KEY_FORMAT.formatted(articleId);
    }
}
```
