### 12. 캐시 최적화 전략 설계 - 기존 캐시 전략의 제약
___
- 조회수에 짧은 만료시간을 가지는 캐시가 적용된 상태 (@Cacheable)
  - 캐시에서 데이터를 조회
  - 캐시유무에 따라 캐시데이터를 응답하거나 원본데이터를 가져와 캐시에 저장 후 응답
- 동시에 많은 트래픽이 온다가정한 테스트
```java
@SpringBootTest
class ViewClientTest {
    @Autowired
    ViewClient viewClient;

    @Test
    void readCacheableMultiThreadTest() throws InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(5);  // 5개 동시요청

        viewClient.count(1L); // 초기화 용도 (캐시가 없기에)

        for (int i = 0; i < 5; i++) {
            CountDownLatch latch = new CountDownLatch(5);// 스레드 풀로 동시에 5개요청
            for (int j = 0; j < 5; j++) {
                executorService.execute(() -> {
                    viewClient.count(1L);
                    latch.countDown();
                });
            }
            latch.await();
            TimeUnit.SECONDS.sleep(2);
            System.out.println("==== cache expired ====");
        }
    }
}
```
```text
2025-02-21T14:17:36.057+09:00  INFO 45112 --- [hello-board-article-read-service] [pool-2-thread-3] h.board.articleRead.client.ViewClient    : [VieClient.count] articleId: 1
2025-02-21T14:17:36.058+09:00  INFO 45112 --- [hello-board-article-read-service] [pool-2-thread-2] h.board.articleRead.client.ViewClient    : [VieClient.count] articleId: 1
2025-02-21T14:17:36.057+09:00  INFO 45112 --- [hello-board-article-read-service] [pool-2-thread-1] h.board.articleRead.client.ViewClient    : [VieClient.count] articleId: 1
2025-02-21T14:17:36.058+09:00  INFO 45112 --- [hello-board-article-read-service] [pool-2-thread-5] h.board.articleRead.client.ViewClient    : [VieClient.count] articleId: 1
2025-02-21T14:17:36.058+09:00  INFO 45112 --- [hello-board-article-read-service] [pool-2-thread-4] h.board.articleRead.client.ViewClient    : [VieClient.count] articleId: 1
==== cache expired ====
2025-02-21T14:17:38.082+09:00  INFO 45112 --- [hello-board-article-read-service] [pool-2-thread-3] h.board.articleRead.client.ViewClient    : [VieClient.count] articleId: 1
2025-02-21T14:17:38.082+09:00  INFO 45112 --- [hello-board-article-read-service] [pool-2-thread-4] h.board.articleRead.client.ViewClient    : [VieClient.count] articleId: 1
2025-02-21T14:17:38.082+09:00  INFO 45112 --- [hello-board-article-read-service] [pool-2-thread-2] h.board.articleRead.client.ViewClient    : [VieClient.count] articleId: 1
2025-02-21T14:17:38.082+09:00  INFO 45112 --- [hello-board-article-read-service] [pool-2-thread-5] h.board.articleRead.client.ViewClient    : [VieClient.count] articleId: 1
2025-02-21T14:17:38.082+09:00  INFO 45112 --- [hello-board-article-read-service] [pool-2-thread-1] h.board.articleRead.client.ViewClient    : [VieClient.count] articleId: 1
==== cache expired ====
```
- cache가 만료될때마다 multiThread로 5개를 동시에 요청을 보낸다. => • 캐시가 만료될 때마다 원본 데이터 서버에 여러 번 요청되고 있다.

#### 동시 요청 때 동작
```text
client(ArticleReadService), Cache(ArticleReadService 가 관리), Origin Data(View 서비스가 관리)
Client에 의해 동시 요청이 들어오는 상황

1번 요청 시 캐시에 데이터 조회 -> 캐시에 데이터를 찾지 못한다.(cache miss) -> 원본 데이터 접근 -> 원본 데이터 획득 
2번 요청 시 캐시에 데이터 조회 -> 캐시에 데이터를 찾지 못한다.(1번 요청은 원본 데이터를 획득하고 캐시에 적재 X)
-> 원본데이터 접근 -> 원본데이터 획득
-> 1번 요청 원본 데이터 캐시 적재 -> 적재 후 데이터를 응답 
-> 2번 요청 원본 데이터 캐시 적재 -> 적재 후 데이터를 응답
 
동시 요청때 원본데이터 접근, 캐시 데이터 적재 과정이 2번 수행됬다.
=> 한번으로 충분하지만 동시요청이 많다면 원본 데이터 접근하는 과정이 무의미하게 많아질 수 있다. 

동시에 트래픽이 몰려온다면 캐시 만료 시에, 원본 데이터에 접근하여 캐시 갱신이 필요하다.
=> 원본 데이터 서버로 무수히 많은 요청이 전파될 수 밖에 없는 것이다.
하지만 캐시 갱신은 1개의 요청에 대해서만 처리되어도 충분할 수 있다.
```

<br>

### 13. 캐시 최적화 전략 설계 - Request Collapsing
___
#### 캐시 최적화 전략
```text
- 분산 시스템에서 캐시 갱신에 대해서 Lock을 획득 => 분산락(Distributed Lock)을 활용하여 1개의 요청만 처리할 수 있다.
- 다른 요청들은 캐시 갱신될 때까지 대기? => 캐시 갱신이 길어지면 다른 요청은 기다려야한다.
    => 리소스 점유 및 낭비
-  캐시 갱신을 기다리지 않고, 즉시 응답
=> 데이터를 실제 만료로 설정된 시간보다 더 늦게 만료 시킨다
갱신을 위한 만료 시간(Logical TTL)과 실제 만료 시간(Physical TTL)을 다르게 가져가는 전략, Logical TTL < Physical TTL
Logical TTL 때문에 갱신을 수행하더라도, Physical TTL로 인해 데이터는 남아있으므로 즉시 응답할 수 있다

client(ArticleReadService), Cache(ArticleReadService 가 관리), Origin Data(View 서비스가 관리)
Distributed Lock Provider(동시요청 방지 위해 분산락 제공)
*** Client에 의해 동시 요청이 들어오는 상황
1번 요청 시 캐시에 데이터 조회 -> 
Logical TTL은 초과되어서 logical cache miss, Physical TTL은 아직 지나지 않아서 캐시에서 hit 된 데이터는 남아있다.
-> 1번 요청은 캐시 갱신을 위해 Distributed Lock Provider에 락을 요청
(캐시에 데이터는 있었지만, Logical TTL에 의해 아무튼 logical cache miss이므로, 원본 데이터에 접근하여 캐시를 갱신해야 한다.)
-> 1번 요청 락 획득 (캐시를 갱신할 책임이 있다) ->
1번 요청 시 캐시에 데이터 조회 -> 
Logical TTL은 초과되어서 cache miss지만, Physical TTL은 여전히 지나지 않아서 캐시에 데이터는 남아있다
-> 2번 요청도 Distributed Lock Provider에 락을 요청 
-> 2번은 락획득 실패(1번이 획득) 캐시에서 Logical TTL은 초과했지만, Physical TTL은 아직 초과되지 않아서 조회된 데이터를 그대로 응답한다.
-> 1번 요청은 락을 획득했으므로 캐시를 갱신해야하니 원본 데이터에 접근
-> 원본데이터 획득 -> 캐시에 적재 -> 1번 요청은 데이터 응답

원본 데이터 접근이 1번만 수행되었다. 캐시 갱신에 대해 분산 락을 획득함으로써, 중복 수행을 방지한 것이다.
Distributed Lock Provider로 락을 요청하는 과정이 추가되었지만, 원본 데이터를 다시 캐시에 적재하는 과정보다 훨씬 빠르게 수행될 수 있다.
```
- Distributed Lock Provider : Redis의 setIfAbsent 연산으로 빠르게 처리
- Logical TTL과 Physical TTL이 다르므로, 갱신이 처리 되기 전까지 과거 데이터가 일시적으로 노출될 수 있다.
- 조회수와 같이 캐시 만료 시간이 아주 짧으면서 실시간 데이터 일관성이 반드시 중요하지 않은 작업, 또는 원본 데이터를 처리하는 게 아주 무거운 작업이면,
  - 무의미할 수 있는 중복된 요청 트래픽을 줄임으로써 많은 이점을 가져올 수 있다.
  - 원본 데이터 서버에 부하를 줄이면서도, 캐시를 적극 활용하며 조회 성능을 최적화하는데 유리해질 수 있는 것
- Request Collapsing :  여러 개의 동일하거나 유사한 요청을 하나의 요청으로 병합하여 처리하는 기법


<br>

### 14. 캐시 최적화 전략 구현 - Request Collapsing
___
- OptimizedCache : OptimizedCache 데이터를 Redis에 저장해준다.
```java
@Getter
@ToString
public class OptimizedCache {
    private String data;
    private LocalDateTime expiredAt;   // Logical TTL

    public static OptimizedCache create(Object data, Duration ttl) {
        OptimizedCache optimizedCache = new OptimizedCache();
        optimizedCache.data = DataSerializer.serialize(data);
        optimizedCache.expiredAt = LocalDateTime.now().plus(ttl);
        return optimizedCache;
    }

    // logical TTL 만료 확인
    @JsonIgnore
    public boolean isExpired() {
        return LocalDateTime.now().isAfter(expiredAt);
    }

    // data -> 실제 객체 타입 변경
    public <T> T parseData(Class<T> dataType) {
        return DataSerializer.deserialize(data, dataType);
    }
}
```
- Test
```java
class OptimizedCacheTest {

    @Test
    void parseDataTest() {
        parseDataTest("data", 10);
        parseDataTest(3L, 10);
        parseDataTest(3, 10);
        parseDataTest(new TestClass("Hi"), 10);
    }

    void parseDataTest(Object data, long ttlSeconds) {
        //given
        OptimizedCache optimizedCache = OptimizedCache.of(data, Duration.ofSeconds(ttlSeconds));
        System.out.println("optimizedCache = " + optimizedCache);

        //when
        Object resolvedData = optimizedCache.parseData(data.getClass());

        //then
        System.out.println("resolvedData = " + resolvedData);
        assertThat(resolvedData).isEqualTo(data);
    }

    @Test
    void isExpiredTest() {
        assertThat(OptimizedCache.of("data", Duration.ofDays(-30)).isExpired()).isTrue();
        assertThat(OptimizedCache.of("data", Duration.ofDays(30)).isExpired()).isFalse();   // 만료 X
    }

    @Getter
    @NoArgsConstructor
    @ToString
    @EqualsAndHashCode
    @AllArgsConstructor
    static class TestClass {
        String testData;
    }
}
```
- TTL 을 받아서 Logical TTL , Physical TTL 계산
```java
@Getter
public class OptimizedCacheTTL {
    private Duration logicalTTL;
    private Duration physicalTTL;
    
    public static final long PHYSICAL_TTL_DELAY_SECONDS = 5;

    // PhysicalTTL 은 Logical TTL 보다 + 5초 커야한다.
    public static OptimizedCacheTTL of(long ttlSeconds) {
        OptimizedCacheTTL optimizedCacheTTL = new OptimizedCacheTTL();
        optimizedCacheTTL.logicalTTL = Duration.ofSeconds(ttlSeconds);
        optimizedCacheTTL.physicalTTL = optimizedCacheTTL.logicalTTL.plusSeconds(PHYSICAL_TTL_DELAY_SECONDS);
        return optimizedCacheTTL;
    }
}
```
- Test
```java
class OptimizedCacheTTLTest {

    @Test
    void ofTest() {
        // given
        long ttlSeconds = 10;

        // when
        OptimizedCacheTTL optimizedCacheTTL = OptimizedCacheTTL.of(ttlSeconds);

        // then
        Assertions.assertThat(optimizedCacheTTL.getLogicalTTL()).isEqualTo(Duration.ofSeconds(ttlSeconds));
        Assertions.assertThat(optimizedCacheTTL.getPhysicalTTL()).isEqualTo(
                Duration.ofSeconds(ttlSeconds).plusSeconds(OptimizedCacheTTL.PHYSICAL_TTL_DELAY_SECONDS));
    }
}
```
- OptimizedCacheLockProvider : 캐시 갱신 요청에 대해 분산락을 잡아 한건의 요청만 처리되도록 한다.
```java
@Component
@RequiredArgsConstructor
public class OptimizedCacheLockProvider {
    private final StringRedisTemplate redisTemplate;

    private static final String KEY_PREFIX = "optimized-cache-lock::";
    private static final Duration LOCK_TTL = Duration.ofSeconds(3);

    //lock 을 잡는 메소드
    public boolean lock(String key) {
        return redisTemplate.opsForValue().setIfAbsent(
                generateLockKey(key),
                "",
                LOCK_TTL
        );
    }

    //lock 해제, 캐시 갱신이 끝났을 때 호출 락을 해제 해준다.
    public void unlock(String key) {
        redisTemplate.delete(generateLockKey(key));
    }

    private String generateLockKey(String key) {
        return KEY_PREFIX + key;
    }
}
```
