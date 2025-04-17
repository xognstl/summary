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

#### 테이블 생성
```text
create table Member ( 
    id bigint not null, 
    name varchar(255), 
    primary key (id) 
);
```

#### Entity 생성
- Entity : JPA가 관리할 객체
```java
@Entity
public class Member {
    @Id
    private Long id;
    private String name;
    //
}
```
- EntityManagerFactory 는 애플리케이션 로딩시점에 하나만 생성
- Entity Manager : 트랜잭션을 만들때마다 생성 
- JPA의 모든 데이터 변경은 트랜잭션 안에서 실행

#### JPQL 
- 객체 지향 SQL , 테이블이 아닌 엔티티 객체를 대상으로 검색

#### JPA CRUD
```java
public class JpaMain {
    public static void main(String[] args) {
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("hello");
        EntityManager em = emf.createEntityManager();
        EntityTransaction tx = em.getTransaction();
        tx.begin(); // 트랜잭션 시작

        try {
            // 등록
            Member member = new Member();
            member.setId(1L);
            member.setName("HelloA");
            em.persist(member); // 저장

            // 조회
            Member findMember = em.find(Member.class, 1L);
            System.out.println("findMember = " + findMember.getId());

            // 삭제
            em.remove(findMember);

            // 수정
            findMember.setName("helloJPA");

            List<Member> findMembers = em.createQuery("select m from Member as m", Member.class)
//                    .setFirstResult(1).setMaxResults(10)    // 페이징
                    .getResultList();
            for (Member findMember : findMembers) {
                System.out.println("findMember.getName() = " + findMember.getName());
            }
            tx.commit();
        }catch (Exception e) {
            tx.rollback();
        }finally {
            em.close();
        }

        emf.close();
    }
}
```
