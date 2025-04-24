## 연관관계 매핑 기초

### 1. 연관관계가 필요한 이유
___
- 예제 시나리오 : 회원과 팀이 있고, 회원은 하나의 팀에만 소속될 수 있다. 회원과 팀은 다대일 관계

##### 객체를 테이블에 맞추어 모델링
- 참조 대신에 외래 키를 그대로 사용
```java
@Entity
public class Member {

    @Id @GeneratedValue
    @Column(name = "MEMBER_ID")
    private Long id;
    @Column(name = "USERNAME")
    private String name;
    @Column(name = "TEAM_ID")
    private Long teamId;
}
@Entity
public class Team {

    @Id
    @GeneratedValue
    @Column(name = "TEAM_ID")
    private Long id;
    private String name;
}
```

- 외래 키 식별자를 직접 다룸
```java
Team team = new Team();
team.setName("Team A");
em.persist(team);

Member member = new Member();
member.setName("member1");
member.setTeamId(team.getId());
em.persist(member);
```

- 식별자로 다시 조회
```java
Member findMember = em.find(Member.class, member.getId());
Long findTeamId = findMember.getTeamId();
Team findTeam = em.find(Team.class, findTeamId);
```

- 객체를 테이블에 맞추어 데이터 중심으로 모델링하면, 협력 관계를 만들 수 없다.
- 테이블은 외래 키로 조인을 사용해서 연관된 테이블을 찾고, 객체는 참조를 사용해 찾는다.

<br>

### 2. 단방향 연관관계
___

#### 객체 지향 모델링
- 객체의 참조와 테이블의 외래키를 매핑
```java
@Entity
public class Member {

    @Id @GeneratedValue
    @Column(name = "MEMBER_ID")
    private Long id;
    @Column(name = "USERNAME")
    private String name;
//    @Column(name = "TEAM_ID")
//    private Long teamId;

    @ManyToOne
    @JoinColumn(name = "TEAM_ID")
    private Team team;
}
```
- ORM 매핑이 된다. Team team(객체 연관관계) -> TEAM_ID(FK)(테이블 연관관계)

- 연관관계 저장, 조회(객체 그래프 탐색)
```java
// 연관관계 저장
Team team = new Team();
team.setName("Team A");
em.persist(team);

Member member = new Member();
member.setName("member1");
member.setTeam(team);   // 단방향 연관관계 설정, 참조 저장
em.persist(member);

// 참조로 연관관계 조회
Member findMember = em.find(Member.class, member.getId());
Team findTeam = findMember.getTeam();

// 연관관계 수정
// 새로운 팀B
Team teamB = new Team();
teamB.setName("TeamB");
em.persist(teamB);
// 회원1에 새로운 팀B 설정
member.setTeam(teamB);
```
