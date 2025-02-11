## 게시글

### 1. Distributed Relational Database
___
- Distributed Relational Database , 분산 관계형 데이터베이스
```text
article 테이블 생성 , client(서버 어플리케이션) 은 단일 DB에서 데이터를 생성/조회/수정/삭제 한다.
서비스가 활성화되면서 단일한 DB로 처리하기엔 큰 부담이 생기게 되었다.
저장해야 할 데이터와 트래픽이 많아졌다면 어떻게 해야할까?
- Scale-Up 
- Scale-Out : 2대의 DB가 운용되고 Client의 요청은 각 DB로 분산될 수 있다.
데이터는 샤딩을 이용하여 데이터를 여러 DB로 분산할 수 있다.
```
- 샤딩(Sharding) : 데이터를 여러 데이터베이스에 분산하여 저장하는 기술
- 샤드(Shard) : 샤딩된 각각의 데이터 단위
- 수직 샤딩(Vertical Sharding) : 데이터를 수직으로 분할하는 방식(컬럼 단위)
  - 식별자를 이용하여 좌측 샤드 title/content, 우측 샤드 board_id/created_at 분산 저장 
  - DB1 : id, title, content, DB2 : id, board_id, create_at 
- 수평 샤딩(Horizontal Sharding) : 데이터를 수평으로 분할하는 방식(행 단위)
  - 좌측 샤드 article_id=1-5000, 우측 샤드 article_id=5001-10000으로 분산 저장

- 장점 : 각 샤드로 데이터 분산되므로, 성능 및 공간 이점
- 단점 : 데이터의 분리로 인해 조인 또는 트랜잭션 관리가 복잡해질 수 있다.


- Range-based Sharding(범위 기반 샤딩) : 데이터를 특정 값(Shard Key)의 특정 범위에 따라 분할하는 기법
  - 좌측 샤드 article_id = 1 ~ 5000, 우측 샤드 article_id = 5001 ~ 10000
  - 범위 데이터 조회 유리 => 100부터 30개 조회하면 한 샤드에서 모든 데이터를 찾을 수 있다.
  - 데이터 쏠림 현상 발생 가능 => 데이터가 6000개면 우측에 1000개만 저장
- Hash-based Sharding(해시 기반 샤딩) : 데이터를 특정 값(Shard Key)의 해시 함수에 따라 분할하는 기법
  - hash_function = article_id -> article_id % 2 라면? 좌측 샤드 = 1,3,5 , 우측 샤드 = 2,4,6
- 균등한 분산을 위한 Shard Key와 해시 함수가 필요하다. 균등하게 분산되지 않으면 데이터 쏠림 현상 발생 가능
- board_id 가 shard key 라면 인기 많은 게시판에 데이터가 몰릴 수 있다.
- Directory-based Sharding(디렉토리 기반 샤딩) : 디렉토리를 이용하여 데이터가 저장된 샤드를 관리하는 기법, 매핑 테이블을 이용하여 각 데이터가 저장된 샤드를 관리한다.  
  디렉토리 관리 비용 있으나, 데이터 규모에 따라 유연한 관리가 가능


- 물리적 샤드 확장 : 기존 DB 가 2대에서 4대가 되면 hash_function = article_id -> article_id % 4  
데이터의 재배치가 있어야한다. Client 에서 4개의 shard로 요청해야 하므로 애플리케이션 코드를 수정하는 등의 새로운 shard 정보를 알아야한다.  
DB의 확장으로 인해 Client도 수정해야 한다.
- 논리적 샤드 : 물리적 샤드는 2개이지만 데이터를 분할하는 가상의 샤드가 4개가 있다.  
DB 시스템 내부에 router를 만든다. => 논리적 샤드 개념을 이용하여,  물리적 샤드를 확장하더라도  
Client의 어떠한 변경도 없이 DB를 유연하게 확장할 수 있다.


- 데이터 복제의 필요성
```text
샤드중 하나가 장애가 발생하는 문제를 해결하기 위해 데이터 복제본을 관리할 수 있다.
Primary(주 데이터베이스)에 데이터를 쓰고, Replica(복제본)에 데이터를 복제한다.
Primary/Replica, Leader/Follower, Master/Slave, Main/Standby 등 유사한 개념이지만,
시스템이나 목적에 따라 다르게 사용되기도 한다.
이러한 복제는 동기적 또는 비동기적으로 처리될 수 있다.
Synchronous(동기적) : 데이터 일관성을 보장하나, 쓰기 성능 저하된다.
Asynchronous(비동기적) : 쓰기 성능 유지되나, 복제본에 최신 데이터가 즉시 반영되지 않을 수 있다.
이러한 시스템을 통해, 고가용성(Primary 재선출을 통한 서비스 지속성), 데이터 유실 방지 및 데이터 백업(Replica 복제본 관리),
부하 분산(Replica 또는 각 샤드로 요청 분산) 등의 다양한 이점을 가질 수 있다
```

<br>

### 2. MySQL 개발 환경 세팅
___
- $ docker pull mysql:8.0.38 : mysql 이미지 다운로드
- $ docker run --name board-mysql -e MYSQL_ROOT_PASSWORD=root -d -p 3306:3306 mysql:8.0.38 : 컨테이너 실행
- $ docker exec -it board-mysql bash
- $ mysql -u root –p

<br>

### 3. 게시글 CRUD API 설계
___
#### 게시글 요구사항
- 각 게시판 단위로 게시글 서비스 이용, 사용자는 게시판에 게시글을 작성하고 조회
- 게시글 조회, 생성, 수정, 삭제 API
- 게시글 목록 조회 API (게시판별 최신순)
#### 테이블 설계
- article_id : primary key, board_id : shard key
- 분산 데이터베이스를 가정하기 때문에 게시글이 N개의 샤드로 분산되는 상황을 고려
- 게시글 서비스는 게시판 단위로 서비스를 이용하고 , 게시판 단위로 게시글 목록이 조회 되므로 shard key 는 board_id
- shard key 가 article_id 이면 동일한 게시판에 속한 게시글이 별도의 샤드에 저장될 수 있다.  
각 게시판의 게시글을 모두를 조회하면 N개의 샤드를 모두 조회해야해서 복잡한 처리가 필요하다.

- create database article; : article 데이터 베이스 생성
```sql
create table article (
  article_id bigint not null primary key,
  title varchar(100) not null,
  content varchar(3000) not null,
  board_id bigint not null,
  writer_id bigint not null,
  created_at datetime not null,
  modified_at datetime not null
);
```
<br>

### 4. Snowflake 
___
#### Primary Key 선택
- 오름차순 유니크 숫자를 애플리케이션에서 직접 생성
  - AUTO_INCREMENT , UUID 사용 XX
  - 분산 시스템과 인덱스의 구조에 대한 이해가 필요하기 때문에 뒤에서 ..
- 오름차순 유니크 숫자를 만들기 위한 알고리즘 : Snowflake, TSID 등등
- Snowflake : 분산 환경에서도 중복 없이 순차적 ID 생성하기 위한 규칙, 유니크, 시간 기반 순차성, 분산 환경에서의 높은 성능


- common(공통 모듈 하위) 에 Snowflake 생성

<br>

### 5. 게시글 CRUD API 구현
___
- article 에 jpa, mysql, snowflake 모듈 dependency 추가
```yaml
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'com.mysql:mysql-connector-j'
    implementation project(':common:snowflake')
}
```
- application.yml 에 spring name, DB, jpa 설정 추가
```yaml
spring:
  application:
    name: hello-board-article-service
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://127.0.0.1:3306/article
    username: root
    password: root
  jpa:
    database-platform: org.hibernate.dialect.MySQLDialect
    open-in-view: false
    show-sql: true
    hibernate:
      ddl-auto: none
```
<br>

- Article Entity 생성, create, update 함수도 만들어놈
```java
@Table(name = "article")
@Getter
@Entity
@ToString
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Article {
    @Id
    private Long articleId;
    private String title;
    private String content;
    private Long boardId;   // shard key
    private Long writerId;
    private LocalDateTime createdAt;
    private LocalDateTime modifiedAt;

    // 팩토리 메서드
    public static Article create(Long articleId, String title, String content, Long boardId, Long writerId) {
        Article article = new Article();
        article.articleId = articleId;
        article.title = title;
        article.content = content;
        article.boardId = boardId;
        article.writerId = writerId;
        article.createdAt = LocalDateTime.now();
        article.modifiedAt = article.createdAt;
        
        return article;
    }
    
    public void update(String title, String content) {
        this.title = title;
        this.content = content;
        modifiedAt = LocalDateTime.now();
    }
}
```

<br>

- ArticleRepository.java 생성
```java
@Repository
public interface ArticleRepository extends JpaRepository<Article, Long> {
}
```

- Article 요청, 응답을 위한 request(create, update), response 객체 
```java
@Getter
@ToString
public class ArticleCreateRequest {

    private String title;
    private String content;
    private Long writerId;
    private Long boardId;
}
```
```java
@Getter
@ToString
public class ArticleUpdateRequest {
    private String title;
    private String content;
}
```
```java
@Getter
@ToString
public class ArticleResponse {
    private Long articleId;
    private String title;
    private String content;
    private Long boardId;
    private Long writerId;
    private LocalDateTime createdAt;
    private LocalDateTime modifiedAt;

    public static ArticleResponse from(Article article) {   // 팩토리 매서드
        ArticleResponse response = new ArticleResponse();
        response.articleId = article.getArticleId();
        response.title = article.getTitle();
        response.content = article.getContent();
        response.boardId = article.getBoardId();
        response.writerId = article.getWriterId();
        response.createdAt = article.getCreatedAt();
        response.modifiedAt = article.getModifiedAt();
        return response;
    }
}
```

<br>

- Service
```java
@Service
@RequiredArgsConstructor
public class ArticleService {
    private final Snowflake snowflake = new Snowflake();
    private final ArticleRepository articleRepository;

    @Transactional
    public ArticleResponse create(ArticleCreateRequest request) {
        Article article = articleRepository.save(
                Article.create(snowflake.nextId(), request.getTitle(), request.getContent(), request.getBoardId(), request.getWriterId())
        );
        return ArticleResponse.from(article);
    }

    @Transactional
    public ArticleResponse update(Long articleId, ArticleUpdateRequest request) {
        Article article = articleRepository.findById(articleId).orElseThrow();  // 데이터가 없으면 예외 발생
        article.update(request.getTitle(), request.getContent());

        return ArticleResponse.from(article);
    }

    public ArticleResponse read(Long articleId) {
        return ArticleResponse.from(articleRepository.findById(articleId).orElseThrow());
    }

    @Transactional
    public void delete(Long articleId) {
        articleRepository.deleteById(articleId);
    }
}
```

<br>

- controller
```java
@RestController
@RequiredArgsConstructor
public class ArticleController {
    private final ArticleService articleService;

    @GetMapping("/v1/articles/{articleId}")
    public ArticleResponse read(@PathVariable Long articleId) {
        return articleService.read(articleId);
    }

    @PostMapping("/v1/articles")
    public ArticleResponse create(@RequestBody ArticleCreateRequest request) {
        return articleService.create(request);
    }

    @PutMapping("/v1/articles/{articleId}")
    public ArticleResponse update(@PathVariable Long articleId, @RequestBody ArticleUpdateRequest request) {
        return articleService.update(articleId, request);
    }

    @DeleteMapping("/v1/articles/{articleId}")
    public void delete(@PathVariable Long articleId) {
        articleService.delete(articleId);
    }
}
```

<br>

- Test 코드 작성
```java
public class ArticleApiTest {
    RestClient restClient = RestClient.create("http://localhost:9000");    // http 요청 할 수 있는 클래스

    @Test
    void createTest() {
        ArticleResponse response = create(new ArticleCreateRequest(
                "h1", "my content", 1L, 1L
        ));

        System.out.println(response);
    }

    ArticleResponse create(ArticleCreateRequest request) {
        return restClient.post()
                .uri("/v1/articles")
                .body(request)
                .retrieve() // 호출
                .body(ArticleResponse.class);
    }

    @Test
    void readTest() {
        ArticleResponse response = read(147543046302195712L);   // create 에서 생성된 ID
        System.out.println("response = " + response);
    }

    ArticleResponse read(Long articleId) {
        return restClient.get()
                .uri("/v1/articles/{articleId}", articleId)
                .retrieve()
                .body(ArticleResponse.class);
    }

    @Test
    void updateTest() {
        update(147543046302195712L);
        ArticleResponse response = read(147543046302195712L);
        System.out.println("response = " + response);
    }

    void update(Long articleId) {
        restClient.put()
                .uri("/v1/articles/{articleId}", articleId)
                .body(new ArticleUpdateRequest("h2 2", "my content 2222"))
                .retrieve();
    }

    @Test
    void deleteTest() {
        restClient.delete()
                .uri("/v1/articles/147543046302195712")
                .retrieve();
    }

    @Getter
    @AllArgsConstructor
    static class ArticleCreateRequest {
        private String title;
        private String content;
        private Long writerId;
        private Long boardId;
    }

    @Getter
    @AllArgsConstructor
    static class ArticleUpdateRequest {
        private String title;
        private String content;
    }
}

```
