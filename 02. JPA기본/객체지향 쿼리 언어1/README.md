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

<br>


### 2. JQPL 기본 문법과 쿼리 API
___

#### JPQL 소개
- JPQL은 객체지향 쿼리 언어다.따라서 테이블을 대상으로 쿼리하는 것이 아니라 엔티티 객체를 대상으로 쿼리한다.
- JPQL은 SQL을 추상화해서 특정데이터베이스 SQL에 의존하지 않는다.
- JPQL은 결국 SQL로 변환된다.

#### JPQL 문법
- select m from Member as m where m.age > 18
- 엔티티와 속성은 대소문자 구분O (Member, age)
- JPQL 키워드는 대소문자 구분X (SELECT, FROM, where)
- 엔티티 이름 사용, 테이블 이름이 아님(Member)
- 별칭은 필수(m) (as는 생략가능)
- 집합과 정렬도 사용가능
    - ex) COUNT(m), //회원수, SUM(m.age), //나이 합, AVG(m.age), //평균 나이, MAX(m.age), //최대 나이, MIN(m.age) //최소 나이
    - ex) GROUP BY, HAVING, ORDER BY

#### TypeQuery, Query
- TypeQuery: 반환 타입이 명확할 때 사용
- Query: 반환 타입이 명확하지 않을 때 사용
```java
TypedQuery<Member> query = em.createQuery("select m from Member m", Member.class);
TypedQuery<String> query2 = em.createQuery("select m.username, m.age from Member m", String.class);
Query query3 = em.createQuery("select m.username, m.age from Member m");
```

#### 결과 조회 API
- query.getResultList(): 결과가 하나 이상일 때, 리스트 반환
- query.getSingleResult(): 결과가 정확히 하나, 단일 객체 반환
  - 결과가 없으면: javax.persistence.NoResultException
  - 둘 이상이면: javax.persistence.NonUniqueResultException

#### 파라미터 바인딩
```java
//이름 기준
Member result = em.createQuery("select m from Member m where m.username = :username", Member.class)
    .setParameter("username", "member1")
    .getSingleResult();

System.out.println("singleResult = " + result.getUsername()); // member1

//위치 기준
Member result = em.createQuery("select m from Member m where m.username = ?1", Member.class)
  .setParameter(1, "member1")
  .getSingleResult();

System.out.println("singleResult = " + result.getUsername());
```

<br>

### 3. 프로젝션
___
- SELECT 절에 조회할 대상을 지정하는 것. 
- 대상은 엔티티, 임베디드 타입, 스칼라 타입(숫자, 문자등 기본 데이터 타입)이 있다. 
- DISTINCT로 중복 제거로 가능
```java
Member member = new Member();
member.setUsername("member1");
member.setAge(10);
em.persist(member);

em.flush();
em.clear();

// 엔티티 프로젝션
// 결과가 영속성 컨텍스트에서 관리됨.
List<Member> result = em.createQuery("select m from Member m", Member.class).getResultList();
Member findMember = result.get(0);
findMember.setAge(20);  // 엔티티 프로젝션은 영속성 컨텍스트에서 관리 되므로 값 변경 가능.

// 엔티티 프로젝션
List<Team> result = em.createQuery("select m.team from Member m", Team.class).getResultList();
//위의 쿼리는 inner join 이 실행된다. 유지보수 측면에서 좋지 않으니 아래와 같이 쿼리를 짠다.
List<Team> result = em.createQuery("select t from Member m join m.team t", Team.class).getResultList();

// 임베디드 타입 프로젝션
List<Address> result = em.createQuery("select o.address from Order o ", Address.class).getResultList();
// 임베디드 타입은 엔티티가 아니라서 select o.address from Address o 같이 사용할 수 없다.

// 스칼라 타입 프로젝션
List<Member> result = em.createQuery("select distinct m.username, m.age from Member m ").getResultList();
```

#### 프로젝션 - 여러 값 조회
- SELECT m.username, m.age FROM Member m
- Query 타입 조회, Object[] 타입으로 조회, new 명령어로 조회
```java
//Object[] 타입으로 조회
List resultList = em.createQuery("select distinct m.username, m.age from Member m ").getResultList();

Object o = resultList.get(0);
Object[] result = (Object[]) o;
System.out.println("username = " + result[0]);
System.out.println("age = " + result[1]);

//Object[] 타입으로 조회
List<Object[]> resultList = em.createQuery("select distinct m.username, m.age from Member m ").getResultList();
Object[] result = resultList.get(0);

// new 명령어로 조회
List<MemberDTO> result
        = em.createQuery("select distinct new jpql.MemberDTO(m.username, m.age) from Member m ", MemberDTO.class).getResultList();

MemberDTO memberDTO = result.get(0);
System.out.println("memberDTO.getUsername() = " + memberDTO.getUsername());
System.out.println("memberDTO.getUsername() = " + memberDTO.getAge());

//MemberDTO.class
public class MemberDTO {

    private String username;
    private int age;

    public MemberDTO(String username, int age) {
        this.username = username;
        this.age = age;
    }
}    
```

<br>

### 4. 페이징
___
- setFirstResult(int startPosition) : 조회 시작 위치(0부터 시작)
- setMaxResults(int maxResult) : 조회할 데이터 수
```java
List<Member> result = em.createQuery("select m from Member m order by m.age desc", Member.class)
                    .setFirstResult(0)
                    .setMaxResults(10)
                    .getResultList();
```

<br>

### 5. 조인
___
#### 조인 종류

```sql
--내부 조인:
SELECT m FROM Member m [INNER] JOIN m.team t
-- 외부 조인:
SELECT m FROM Member m LEFT [OUTER] JOIN m.team t
-- 세타 조인: 
select count(m) from Member m, Team t where m.username = t.name
```

#### 조인 - ON절
- 조인 대상 필터링
- 연관관계 없는 엔티티 외부 조인

##### 1. 조인 대상 필터링
- EX) 회원과 팀을 조인하면서, 팀 이름이 A인 팀만 조인
```sql
-- JPQL : 
SELECT m, t FROM Member m LEFT JOIN m.team t on t.name = 'A'; 
-- SQL:
SELECT m.*, t.* FROM 
Member m LEFT JOIN Team t ON m.TEAM_ID=t.id and t.name='A';
```

##### 2. 연관관계 없는 엔티티 외부 조인
- EX) 회원의 이름과 팀의 이름이 같은 대상 외부 조인
```sql
-- JPQL:
SELECT m, t FROM Member m LEFT JOIN Team t on m.username = t.name;
-- SQL:
SELECT m.*, t.* FROM Member m LEFT JOIN Team t ON m.username = t.name;
```

<br>
