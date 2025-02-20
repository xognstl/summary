### 5. 게시글 조회 최적화 구현 - 조회수 캐시
___
- article-read 서비스에서 조회하는 시점에 viewClient 를 통하여 조회수를 가지고 있다. 
  - 트래픽이 많으면 조회수 서비스로 부하 전파
```text
redis 에서 데이터 조회 -> 데이터가 없으면 count 메소드 내부 로직이 호출되면서 viewService로 원본 데이터 요청
 -> redis 에 데이터를 넣고 응답 -> redis 에 데이터가 있으면 데이터 반환
```
- cache config
```java
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        return RedisCacheManager.builder(connectionFactory)
                .withInitialCacheConfigurations(
                        Map.of( // 캐시 설정 (캐시 이름 , 만료 시간)
                               "articleViewCount", RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofSeconds(1))
                        )
                )
                .build();
    }
}
```
- viewclient에 cache 설정 추가
```java
@Slf4j
@Component
@RequiredArgsConstructor
public class ViewClient {
    private RestClient restClient;

    @Value("${endpoints.hello-board-view-service.url}")
    private String viewServiceUrl;

    @PostConstruct
    public void initRestClient() {
        restClient = RestClient.create(viewServiceUrl);
    }

    @Cacheable(key = "#articleId", value = "articleViewCount")
    public long count(Long articleId) {
        log.info("[VieClient.count] articleId: {}", articleId);
        try {
            return restClient.get()
                    .uri("/v1/article-views/articles/{articleId}/count", articleId)
                    .retrieve()
                    .body(Long.class);
        }catch (Exception e) {
            log.error("[ViewClient.count] articleId={}", articleId, e);
            return 0;
        }
    }
}
```
- test
  - 처음엔 캐시에 데이터가 없어서 로그 출력 이후에는 캐시에 데이터가 있기때문에 미출력
  - 1초로 캐시 만료 설정 해놨기 떄문에 4번 째 호출때 로그 출력
  - 4번 호출했지만 캐시로 인해 2번 log가 출력 된다.
```java
@SpringBootTest
class ViewClientTest {
    @Autowired
    ViewClient viewClient;

    @Test
    void readCacheableTest() throws InterruptedException {
        viewClient.count(1L);   // 로그 출력
        viewClient.count(1L);   // 로그 미출력
        viewClient.count(1L);
        TimeUnit.SECONDS.sleep(3);
        viewClient.count(1L);   // 로그 출력
    }
}
```

<br>

### 6. 게시글 조회 최적화 구현 - Consumer & Controller
___
- kafka consumer
```java
@Slf4j
@Component
@RequiredArgsConstructor
public class ArticleReadEventConsumer {
    private final ArticleReadService articleReadService;

    @KafkaListener(topics = {
            EventType.Topic.BOARD_ARTICLE,
            EventType.Topic.BOARD_COMMENT,
            EventType.Topic.BOARD_LIKE
    })
    public void listen(String message, Acknowledgment ack) {
        log.info("[ArticleReadEventConsumer.listen] message: {}", message);
        Event<EventPayload> event = Event.fromJson(message);
        if (event != null) {
            articleReadService.handleEvent(event);
        }
        ack.acknowledge();
    }
}
```
- consumer config
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
- controller
```java
@RestController
@RequiredArgsConstructor
public class ArticleReadController {
    private final ArticleReadService articleReadService;

    @GetMapping("/v1/articles/{articleId}")
    public ArticleReadResponse read(@PathVariable("articleId") Long articleId) {
        return articleReadService.read(articleId);
    }
}
```

<br>

### 7. 게시글 조회 최적화 전략 구현 - 테스트
___
- data Initializer 로 article/comment/like/view 서비스에 데이터 생성 요청
- 데이터가 생성되면 ArticleReadEventConsumer.listen 에 찍어놓은 log가 출력된다. 
- 잘 처리 되었다면 EventHandler 를 통해 ArticleQueryModel 이 Redis에 만들어져 있을텐데 그것을 조회하는 테스트 
- 이벤트를 받아와서 redis에 저장되면 로그 X, fetch 라는 함수를 타면 로그가 찍힌다.
- select * from article order by article_id asc limit 1 offset 10; 옛날에 만들어진 데이터 가져오기 
- DataInitializer 실행 후 생성된 ID로 TEST 하면 로그가 찍히지 않는데 옛날에 만들어진 데이터로 조회시 fetch 메소드에 로그 찍힌다.
```java
public class ArticleReadApiTest {
    RestClient restClient = RestClient.create("http://localhost:9005");

    @Test
    void readTest() {   
        ArticleReadResponse response = restClient.get()
                .uri("/v1/articles/{articleId}", 147550439382646784L)    // DataInitializer 실행 때 생성된 ID
                .retrieve()
                .body(ArticleReadResponse.class);

        System.out.println("response = " + response);
    }
}
```
