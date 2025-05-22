## 객체지향 쿼리 언어1 - 기본 문법

### 1. JPQL
___
#### JPQL
- JPA는 SQL을 추상화한 JPQL이라는 객체 지향 쿼리 언어 제공한다.
- SQL과 문법 유사, SELECT, FROM, WHERE, GROUP BY, HAVING, JOIN 지원
- JPQL은 엔티티 객체 대상, SQL은 DB 테이블을 대상으로 쿼리
- SQL을 추상화해서 특정 DB에 의존하지 않는다. JPQL은 객체지향 SQL
- 동적 쿼리를 만들기 어렵다는 단점이 있다. (queryDSL을 이용하여 해결)

```java
String jpql = "select m from Member m where m.name like '%hello%'";
List<Member> result = em.createQuery(jpql, Member.class).getResultList();
```

#### Criteria
- 문자가 아닌 자바코드로 JPQL을 작성 가능
- 동적 쿼리 작성하는데 장점이 있지만 너무 복잡하고 실용성이 없다. QueryDSL 사용 권장.
- JPQL 빌더역할
- 이런게 있다 정도만 알고있자.
- 
```java
//Criteria 사용 준비
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Member> query = cb.createQuery(Member.class);

//루트 클래스 (조회를 시작할 클래스)
Root<Member> m = query.from(Member.class);

CriteriaQuery<Member> cq = query.select(m).where(cb.equal(m.get("username"), "kim"));
List<Member> resultList = em.createQuery(cq).getResultList();
```

#### QueryDSL
- 문자가 아닌 자바코드로 JPQL을 작성할 수 있다
- JPQL 빌더 역할을 하고, 컴파일 시점에 문법오류를 찾을 수 있다.
```java
JPAFactoryQuery query = new JPAQueryFactory(em);
QMember m = QMember.member;
List<Member> list = query.selectFrom(m)
                      .where(m.age.gt(18))
                      .orderBy(m.name.desc())
                      .fetch();
```

#### 네이티브 SQL
- JPA가 제공하는 SQL을 직접 사용하는 기능
- JPQL로 해결할 수 없는 특정 데이터베이스에 의존적인 기능 ex) 오라클 CONNECT BY, 특정 DB만 사용하는 SQL 힌트
```java
String sql = "SELECT ID, AGE, TEAM_ID, NAME FROM MEMBER WHERE NAME = ‘kim’";
List<Member> resultList = em.createNativeQuery(sql, Member.class).getResultList();
```


#### JDBC 직접 사용, SpringJdbcTemplate
- JPA를 사용하면서 JDBC 커넥션을 직접 사용하거나, 스프링 JdbcTemplate, 마이바티스등을 함께 사용 가능
- 단 영속성 컨텍스트를 적절한 시점에 강제로 플러시 필요 ex) JPA를 우회해서 SQL을 실행하기 직전에 영속성 컨텍스트 수동 플러시
- flush 는 commit, query 를 호출할때만 된다.
