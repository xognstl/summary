## JPA 시작

### 1. 프로젝트 생성
___
- 라이브러리 추가 
  - pom.xml jpa hibernate, h2 
- resource/META-INF/persistence.xml 생성
  -  db 정보 , 옵션 설정

<br>

### 2. 프로젝트 개발
___
#### JPA 구동방식
- persistence.xml 에서 설정 정보 조회
- EntityManagerFactory 생성
- EntityManager 생성

#### JPA 동작 테스트
```java
@SpringBootApplication
public class JpaMain {

    public static void main(String[] args) {
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("hello");
        EntityManager entityManager = emf.createEntityManager();
        entityManager.close();
        emf.close();

        SpringApplication.run(JpaMain.class, args);
    }
}
```

#### 테이블 생성
```text
create table Member ( 
    id bigint not null, 
    name varchar(255), 
    primary key (id) 
);
```
