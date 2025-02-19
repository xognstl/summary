### 10. Transactional Outbox 모듈 구현
___
- common -> outbox-message-relay 프로젝트 생성
- dependencies 추가 : jpa, redis, kafka, mysql , snowflake, event, data-serializer
- producer 데이터 보내는 config
```java
@EnableAsync    // 트랜잭션이 끝나면 kafka에 대한 이벤트 전송을 비동기로 처리
@Configuration
@ComponentScan("hello.board.common.outboxmessagerelay")
@EnableScheduling   // 전송되지 않은 이벤트를 주기적으로 가져와서 polling 해 kafka로  보낼것
public class MessageRelayConfig {
    @Value("${spring.kafka.bootstrap-servers")
    private String bootstrapServers;

    @Bean   // kafka 로 프로듀서 애플리케이션들이 이벤트를 전송
    public KafkaTemplate<String, String> messageRelayKafkaTemplate() {
        Map<String, Object> configProps = new HashMap<>();  // kafka producer 설정
        configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        // hot-article yml 을 보면 StringDeserializer
        configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        configProps.put(ProducerConfig.ACKS_CONFIG, "all");
        return new KafkaTemplate<>(new DefaultKafkaProducerFactory<>(configProps));
    }

    // 트랜잭션이 끝날 떄마다 이벤트 전송을 비동기로 전송하기 위해 처리하는 스레드 풀
    @Bean
    public Executor messageRelayPublishEventExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(20);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("mr-pub-event-");
        return executor;
    }

    // 이벤트 전송이 아직 되지 않은 것들 10초 이후에 이벤트를 주기적으로 보내주기 위한 스레드풀
    @Bean
    public Executor messageRelayPublishPendingEventExecutor() {
        // 각 애플리케이션 마다 shard 가 분할 되어 할당 되기 떄문에 싱글스레드로 미전송 이벤트 전송
        return Executors.newSingleThreadExecutor();
    }
}
```
- 상수 정의
```java
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public final class MessageRelayConstants {
    public static final int SHARD_COUNT = 4;     // 임의의 값
}
```
- Entity
```java
@Table(name = "outbox")
@Getter
@Entity
@ToString
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Outbox {
    @Id
    private Long outboxId;
    @Enumerated(EnumType.STRING)
    private EventType eventType;
    private String payload;
    private Long shardKey;  // 비즈니스 로직으로 처리된 데이터 테이블 , Outobox 테이블이 동일한 샤드에서 처리되어야 단일 트랙잭션
    private LocalDateTime createdAt;

    public static Outbox create(Long outboxId, EventType eventType, String payload, Long shardKey) {
        Outbox outbox = new Outbox();
        outbox.outboxId = outboxId;
        outbox.eventType = eventType;
        outbox.payload = payload;
        outbox.shardKey = shardKey;
        outbox.createdAt = LocalDateTime.now();
        return outbox;
    }
}
```
- outbox 이벤트 전달 메소드
```java
@Getter
@ToString
public class OutboxEvent {
    private Outbox outbox;

    public static OutboxEvent of(Outbox outbox) {
        OutboxEvent event = new OutboxEvent();
        event.outbox = outbox;
        return event;
    }
}
```
- outbox 이벤트를 만드는 이벤트 퍼블리셔
```java
@Component
@RequiredArgsConstructor
public class OutboxEventPublisher {
    private final Snowflake outboxIdSnowflake = new Snowflake();
    private final Snowflake eventIdSnowflake = new Snowflake();
    private final ApplicationEventPublisher applicationEventPublisher;

    // article Service 에서 이 모듈을 가져와서 OutboxEventPublisher를 통해 이벤트 발행
    public void publish(EventType type, EventPayload payload, Long shardKey) {
        // articleId = 10 , shardKey == articleId
        // 10 % 4 = 물리적샤드 2
        Outbox outbox = Outbox.create(
                outboxIdSnowflake.nextId(),
                type,
                Event.of(
                        eventIdSnowflake.nextId(), type, payload
                ).toJson(),
                shardKey % MessageRelayConstants.SHARD_COUNT
        );
        applicationEventPublisher.publishEvent(OutboxEvent.of(outbox)); // 메시지 발행 -> messageRelay에서 수신해서 처리
    }
}
```
- 샤드를 애플리케이션에 균등하게 할당하기 위한 클래스, 애플리케이션에 할당된 shard를 List 로 반환
```java
@Getter
public class AssignedShard {
    private List<Long> shards;

    // appId : 지금 실행된 애플리케이션 Id, appIds : 코디네이터에 의해 지금 실행되는 애플리케이션의 목록
    public static AssignedShard of(String appId, List<String> appIds, long shardCount) {
        AssignedShard assignedShard = new AssignedShard();
        assignedShard.shards = assign(appId, appIds, shardCount);   //shard 할당
        return assignedShard;
    }

    private static List<Long> assign(String appId, List<String> appIds, long shardCount) {
        int appIndex = findAppIndex(appId, appIds); // 현재 애플리케이션 index
        if(appIndex == -1) {    // 할당할 샤드 x
            return List.of();   // 빈리스트 반환
        }

        long start = appIndex * shardCount / appIds.size();
        long end = (appIndex + 1) * shardCount / appIds.size() -1;
        // 인덱스를 통해 start - end 사이의 범위가 이 애플리케이션이 할당된 shard 가 된다.
        return LongStream.rangeClosed(start, end).boxed().toList();
    }

    // 정렬된 상태에서의 앱 목록에서 현재 애플리케이션 id , 그번호 반환
    private static int findAppIndex(String appId, List<String> appIds) {
        for (int i = 0; i < appIds.size(); i++) {
            if(appIds.get(i).equals(appId)) {
                 return i;
            }
        }
        return -1;
    }
}
```
- test
```java
class AssignedShardTest {

    @Test
    void ofTest() {
        // given
        Long shardCount = 64L;
        List<String> appList = List.of("appId1", "appId2", "appId3");//코디네이터에 살아있다고 떠있는 애플리케이션 수

        // when
        AssignedShard assignedShard1 = AssignedShard.of(appList.get(0), appList, shardCount);// 각 애플리케이션에 샤드를 할당
        AssignedShard assignedShard2 = AssignedShard.of(appList.get(1), appList, shardCount);
        AssignedShard assignedShard3 = AssignedShard.of(appList.get(2), appList, shardCount);
        AssignedShard assignedShard4 = AssignedShard.of("invalid", appList, shardCount);
        // then
        //4개의 assignedShard 에서 shardList 를 꺼내서 하나의 리스트로
        List<Long> result = Stream.of(assignedShard1.getShards(), assignedShard2.getShards(),
                        assignedShard3.getShards(), assignedShard4.getShards())
                .flatMap(List::stream)
                .toList();
        System.out.println("result = " + result);
        assertThat(result).hasSize(shardCount.intValue());
        
        for(int i=0; i<shardCount.intValue(); i++) {
            assertThat(result.get(i)).isEqualTo(i);
        }

        assertThat(assignedShard4.getShards()).isEmpty();

    }
}
```
- 코디네이터 생성 : 살아있는 애플리케이션 추적, 관리
```java
@Component
@RequiredArgsConstructor
public class MessageRelayCoordinator {
    private final StringRedisTemplate redisTemplate;

    @Value("${spring.application.name")
    private String applicationName;

    private final String APP_ID = UUID.randomUUID().toString();

    private final int PING_INTERVAL_SECONDS = 3;
    private final int PING_FAILURE_THRESHOLD = 3; // 3초간격으로 3번쏴서 반응 없으면 죽었따 판단

    // 지금 애플리케이션이 할당된 shard 목록을 반환해줄수 있다.
    public AssignedShard assignedShards() {
        return AssignedShard.of(APP_ID, findAppIds(), MessageRelayConstants.SHARD_COUNT);
    }

    private List<String> findAppIds() {
        return redisTemplate.opsForZSet().reverseRange(generateKey(), 0, 1).stream()
                .sorted()
                .toList();
    }

    @Scheduled(fixedDelay = PING_INTERVAL_SECONDS, timeUnit = TimeUnit.SECONDS)
    public void ping() {
        //executePipelined 한번의 통신으로 여러개의 연산 한번에 전송 가능
        redisTemplate.executePipelined((RedisCallback<?>) action -> {
            StringRedisConnection conn = (StringRedisConnection) action;
            String key = generateKey();
            conn.zAdd(key, Instant.now().toEpochMilli(), APP_ID);
            conn.zRemRangeByScore(  // 스코어 기준으로 제거
                    key,
                    Double.NEGATIVE_INFINITY,
                    Instant.now().minusSeconds(PING_INTERVAL_SECONDS * PING_FAILURE_THRESHOLD).toEpochMilli()
            );
            return null;
        });
    }

    // 애플리케이션이 종료될떄는 자기가 죽었다고 알려 줄 수 있다. => Redis 에서 자신의 appId 제거
    @PreDestroy
    public void leave() {
        redisTemplate.opsForZSet().remove(generateKey(), APP_ID);
    }

    private String generateKey() {
        return "message-relay-coordinator::app-list::%s".formatted(applicationName);
    }
}
```
- repository
```java
@Repository
public interface OutboxRepository extends JpaRepository<Outbox, Long> {
    // 이벤트가 아직 전송되지 않은 것들을 주기적으로 폴링 그러한 이벤트를 조회하기 위한 메서드
    List<Outbox> findAllByShardKeyAndCreatedAtLessThanEqualOrderByCreatedAtAsc(
            Long shardKey,
            LocalDateTime createdAt,
            Pageable pageable
    );
}
```
- MessageRelay
```java
@Slf4j
@Component
@RequiredArgsConstructor
public class MessageRelay {
    private final OutboxRepository outboxRepository;
    private final MessageRelayCoordinator messageRelayCoordinator; // 살아있는 app 떠있는지 확인, 자기한테 할당된 샤드가 몇개인지 반환
    private final KafkaTemplate<String, String> messageRelayTemplate; // config 에서 정의한 messageRelayTemplate() 주입

    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)   // 트랜잭션에 대한 이벤트를 받을 수 있다.
    public void createOutbox(OutboxEvent outboxEvent) { // commit 전에 outboxEvent를 받아서 repository 에 저장
        log.info("[MessageRelay.createOutbox] outboxEvent={}", outboxEvent);
        // commit 되기 전이니 데이터에 대한 비즈니스 로직이 처리되고 트랜잭션에 단일하게 묶였다.
        outboxRepository.save(outboxEvent.getOutbox());
    }

    @Async("messageRelayPublishEventExecutor")
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)  // 트랜잭션이 commit 되면 비동기로 카프카 이벤트 전송
    public void publishEvent(OutboxEvent outboxEvent) {
        publishEvent(outboxEvent.getOutbox());//실제 발행을 시행, OutboxEvent에서 outbox만 꺼내 kafka 로 전송
    }

    private void publishEvent(Outbox outbox) {
        try {
            messageRelayTemplate.send(
                    outbox.getEventType().getTopic(),
                    // kafka에 전송하는 키, shard 키면 동일한 kafka 파티션으로 전송, 동일한 카프카 파티션으로 전송되면 순서 보장
                    // outbox가 동일한 shardkey에 대해서는 카프카에서 동일한 순서로 전송된다.
                    String.valueOf(outbox.getShardKey()),
                    outbox.getPayload()
            ).get(1, TimeUnit.SECONDS) ; //get 하면 결과를 기달릴 수있따.
            outboxRepository.delete(outbox);
        } catch (Exception e) {
            log.error("[MessageRelay.publishEvent] outbox={}", outbox, e);
        }
    }

    // 10초 동안 전송 안된 이벤트를 주기적으로 polling
    @Scheduled(
            fixedDelay = 10,
            initialDelay = 5,
            timeUnit = TimeUnit.SECONDS,
            scheduler = "messageRelayPublishPendingEventExecutor"
    )
    public void publishPendingEvent() {
        AssignedShard assignedShard = messageRelayCoordinator.assignedShards(); // 자기한테 할당된 샤드 목록
        log.info("[MessageRelay.publishPendingEvent] assignedShard size={}", assignedShard.getShards().size());
        for (Long shard : assignedShard.getShards()) {
            List<Outbox> outboxes = outboxRepository.findAllByShardKeyAndCreatedAtLessThanEqualOrderByCreatedAtAsc(
                    shard,
                    LocalDateTime.now().minusSeconds(10),
                    Pageable.ofSize(100)
            );
            for (Outbox outbox : outboxes) {
                publishEvent(outbox);   // 전송
            }
        }
    }
}
```
- 각 마이크로 서비스들은 OutboxEventPublisher 로 같은 트랜잭션 내에서 이벤트만 발행하면 메시지 릴레이가 자동으로 동작하게 된다.
- 자동으로 동작하려면 빈도 자동으로 등록이 되어야 한다. 
- resource - META-INF -> org.springframework.boot.autoconfigure.AutoConfiguration.imports 에  
hello.board.common.outboxmessagerelay.MessageRelayConfig
- config 위치 지정
- 다른 모듈에서 outboxmessagerelay 를 dependency로 등록하면 messageRelayConfig 를 자동으로 읽어 컴포넌트 스캔한다.

<br>

### 11. Transactional Outbox 모듈 적용
___
- article/comment/like/view 프로젝트에 redis, kafka dependency 추가
- build.gradle 에 event, outbox-message-relay 프로젝트 등록
- Outbox repository scan 할수 있게 각 프로젝트에 scan 등록
```java
@EntityScan(basePackages = "hello.board")
@SpringBootApplication
@EnableJpaRepositories(basePackages = "hello.board")
public class ViewApplication {
    public static void main(String[] args) {
        SpringApplication.run(ViewApplication.class, args);
    }
}
```
- 게시글이 생성, 삭제 될때 이벤트를 보내줘야한다.
- article Service 에 create, update, delete 할때 이벤트 발행
- comment Service 에 create, delete 할때 이벤트 발행
- like Service 에 likePessimisticLock1, unlikePessimisticLock1 할때 이벤트 발행
- View 에서는 백업하는 시점에 이벤트 발행
```java
@Service
@RequiredArgsConstructor
public class ArticleService {
    private final Snowflake snowflake = new Snowflake();
    private final ArticleRepository articleRepository;
    private final OutboxEventPublisher outboxEventPublisher;    // 이벤트 전송 -> MessageRelay 에서 event를 받아서 커밋전, 후 메소드 실행
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

        /* kafka에 event 발행 */
        outboxEventPublisher.publish(
                EventType.ARTICLE_CREATED,
                ArticleCreatedEventPayload.builder()
                        .articleId(article.getArticleId())
                        .title(article.getTitle())
                        .content(article.getContent())
                        .boardId(article.getBoardId())
                        .writerId(article.getWriterId())
                        .createdAt(article.getCreatedAt())
                        .modifiedAt(article.getModifiedAt())
                        .boardArticleCount(count(article.getBoardId()))
                        .build(),
                article.getBoardId() //동일한 단일 트랜잭션에서 동일한 shard로 처리되야한다. article의 shard key
        );

        return ArticleResponse.from(article);
    }

    // total 개수 구하기
    public Long count(Long boardId) {
        return boardArticleCountRepository.findById(boardId)
                .map(BoardArticleCount::getArticleCount)
                .orElse(0L);
    }
}
```
