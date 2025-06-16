# 객체지향 쿼리 언어2 - 중급 문법


___
## 경로 표현식  

### 경로 표현식

경로 표현식은 .(점)을 찍어 객체 그래프를 탐색하는 것이다.

- 상태 필드(state field): 단순히 값을 저장하기 위한 필드 (ex: m.username)
- 연관 필드(association field): 연관관계를 위한 필드
- 단일 값 연관 필드 : @ManyToOne, @OneToOne, 대상이 엔티티(ex: m.team)
- 컬렉션 값 연관 필드 : @OneToMany, @ManyToMany, 대상이 컬렉션(ex: m.orders)

### 경로 표현식 특징
- 상태 필드(state field): 경로 탐색의 끝, 탐색X
- 단일 값 연관 경로: 묵시적 내부 조인(inner join) 발생, 탐색O
- 컬렉션 값 연관 경로: 묵시적 내부 조인 발생, 탐색X
  - FROM 절에서 명시적 조인을 통해 별칭을 얻으면 별칭을 통해 탐색 가능

```java
// 상태 필드 경로 탐색
String query = "select m.username from Member m";

//단일 값 연관 경로 탐색
String query = "select m.team from Member m";
// 묵시적 내부조인 상당히 위험한 방법.
// select m.* from Member m inner join Team on member_id = team_id; 

// 컬렉션 값 연관 경로
String query = "select t.members From Team t";//t.members 뒤에 .으로 무언가를 탐색할수 없다.
//select t.members.username from Team t -> 실패

// 명시적 조인
String query = "select m From Team t join t.members m";
```

### 경로 탐색을 사용한 묵시적 조인 시 주의사항
- 항상 내부 조인
- 컬렉션은 경로 탐색의 끝, 명시적 조인을 통해 별칭을 얻어야함
- 경로 탐색은 주로 SELECT, WHERE 절에서 사용하지만 묵시적 조인으로 인해 SQL의 FROM (JOIN) 절에 영향을 줌

### 결론
- 묵시적조인은 사용하지 않고 명시적 조인을 사용(튜닝 떄문에)


<br>

___
## 페치 조인(fetch join)

### 페치 조인이란.
- JPQL에서 성능 최적화를 위해 제공하는 기능
- 연관된 엔티티나 컬렉션을 SQL 한 번에 함께 조회하는 기능
- join fetch 명령어 사용
- 페치 조인 ::= [ LEFT [OUTER] | INNER ] JOIN FETCH 조인경로

### 엔티티 페치 조인(다대일, 일대일)
- 회원을 조회하면서 연관된 팀도 함께 조회(SQL 한 번에)
- SQL을 보면 회원 뿐만 아니라 팀(T.*)도 함께 SELECT

```sql
-- JPQL
select m from Member m join fetch m.team;
-- SQL 
SELECT M.*, T.* FROM MEMBER M
INNER JOIN TEAM T ON M.TEAM_ID=T.ID;
```

![화면 캡처 2023-05-22 205522](https://github.com/xognstl/JPA1/assets/48784785/988a377c-f50f-4813-9f97-893b7df45b0f)

```java
Team teamA = new Team();
teamA.setName("팀A");
em.persist(teamA);

Team teamB = new Team();
teamB.setName("팀B");
em.persist(teamB);

Member member1 = new Member();
member1.setUsername("회원1");
member1.setTeam(teamA);
em.persist(member1);

Member member2 = new Member();
member2.setUsername("회원2");
member2.setTeam(teamA);
em.persist(member2);

Member member3 = new Member();
member3.setUsername("회원3");
member3.setTeam(teamB);
em.persist(member3);



em.flush();
em.clear();


//            String query = "select m from Member m"; // 이렇게 하면 회원이 100명일때 100 + 1 개의 쿼리가 실행
String query = "select m from Member m join fetch m.team";


List<Member> resultList = em.createQuery(query, Member.class).getResultList();

for (Member member : resultList) {
    System.out.println("member = " + member.getUsername() + " : " + member.getTeam().getName());
}
//        결과
//        member = 회원1 : 팀A
//        member = 회원2 : 팀A
//        member = 회원3 : 팀B
```


### 컬렉션 페치 조인(일대다)

```sql
-- JPQL
select t
from Team t join fetch t.members
where t.name = '팀A';
-- SQL
SELECT T.*, M.*
FROM TEAM T
INNER JOIN MEMBER M ON T.ID=M.TEAM_ID
WHERE T.NAME = '팀A'; 
```

![화면 캡처 2023-05-22 205522](https://github.com/xognstl/JPA1/assets/48784785/460bbc2f-0c5b-4416-8ed9-8990b2f7cac3)

```java
String query = "select t from Team t join fetch t.members";

List<Team> resultList = em.createQuery(query, Team.class).getResultList();

for (Team team : resultList) {
    System.out.println("team = " + team.getName() + " : " + team.getMembers().size());
}
//        결과
//        team = 팀A : 2
//        team = 팀A : 2
//        team = 팀B : 1
```

** 결과가 팀A가 2개로 중복이 된다. DB에서 일대다 조인을 하면 데이터가 뻥튀기가 된다.

### 페치 조인과 DISTINCT
- JPQL의 DISTINCT는 SQL에 DISTINCT를 추가, 애플리케이션에서 엔티티 중복 제거
- select distinct t from t join fetch t.member;
- SQL에 DISTINCT를 추가하지만 데이터가 다르므로 SQL 결과에서 중복제거 실패
- DISTINCT가 추가로 애플리케이션에서 중복 제거시도
- 같은 식별자를 가진 Team 엔티티 제거
==> 하이버네이트6 부터는 distinct 명령어가 없어도 애플리케이션에서 중복 제거가 자동으로 적용

### 페치 조인과 일반 조인의 차이
- 일반 조인 실행시 연관된 엔티티를 함께 조회하지 않는다.
```java
String query = "select t from Team t join t.members m";
// team 만 select 된다. 쿼리가 N+1회 발생한다.
```
- JPQL은 결과를 반환할 때 연관관계 고려X
- 단지 SELECT 절에 지정한 엔티티만 조회할 뿐
- 여기서는 팀 엔티티만 조회하고, 회원 엔티티는 조회X
- 페치 조인을 사용할 때만 연관된 엔티티도 함께 조회(즉시 로딩)
- 페치 조인은 객체 그래프를 SQL 한번에 조회하는 개념


<br>

## 페치 조인 - 한계

### 페치 조인의 한계
- 페치 조인 대상에는 별칭을 줄 수 없다.(하이버네이트는 가능, 가급적 사용X)
ex) String query = "select t from Team t join fetch t.members m where m.username = ''";
- 둘 이상의 컬렉션은 페치 조인 할 수 없다.
  - 1:N:N이 되서 데이터의 개수가 엄청 많아진다.
- 컬렉션을 페치 조인하면 페이징 API(setFirstResult, setMaxResults)를 사용할 수 없다.
  - 일대일, 다대일 같은 단일 값 연관 필드들은 페치 조인해도 페이징 가능
  - 하이버네이트는 경고 로그를 남기고 메모리에서 페이징(매우 위험)
  - 컬렉션 페치 조인의 사진 참고. 1건만 가지고 오라고 한 경우 팀A는 회원을 한명만 가지고 있다고 나온다.

### 페이징 문제 해결 방법
#### 이러한 문제는 일대다에서 다대일로 조회한다.(페이징에 문제XX)

```java
String query = "select t from Team t join fetch t.members";
// =>> 
String query = "select m from Member m join fetch m.team t";
```

#### BatchSize 이용
```java
//Member.class
@BatchSize(size = 100)
@OneToMany(mappedBy = "team")
private List<Member> members = new ArrayList<>();

String query = "select t from Team t";
List<Team> resultList = em.createQuery(query, Team.class)
        .setFirstResult(0).setMaxResults(2).getResultList();

for (Team team : resultList) {
    System.out.println("team = " + team.getName() + " : " + team.getMembers());
    for (Member member : team.getMembers()) {
        System.out.println("member = " + member);
    }
}
```
- batchSize를 사용하지않으면 쿼리가 많이 실행된다.
- batchSize 지정 시 하나씩 조회가 아니라 where m.team_id in(?, ? ... ?) 형식으로 사이즈 만큼 한번에 조회.
- global Setting 사용가능.
```xml
<!-- persistence.xml-->
<property name="hibernate.default_batch_fetch_size" value="100" />
```

### 페치 조인의 특징
- 연관된 엔티티들을 SQL 한 번으로 조회 - 성능 최적화
- 엔티티에 직접 적용하는 글로벌 로딩 전략보다 우선함 (@OneToMany(fetch = FetchType.LAZY) //글로벌 로딩 전략)
- 실무에서 글로벌 로딩 전략은 모두 지연 로딩
- 최적화가 필요한 곳은 페치 조인 적용

### 페치 조인 - 정리
- 모든 것을 페치 조인으로 해결할 수 는 없음
- 페치 조인은 객체 그래프를 유지할 때 사용하면 효과적
- 여러 테이블을 조인해서 엔티티가 가진 모양이 아닌 전혀 다른결과를 내야 하면, 페치 조인 보다는 일반 조인을 사용하고 필요한 데이터들만 조회해서 DTO로 반환하는 것이 효과적
