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

<br>

### 3. 양방향 연관관계와 연관관계의 주인
___

#### 양방향 매핑
- Member 객체에서 Team 을 조회 할 수 있지만 Team 에서 Member 를 볼 수 없다.
```java
@Entity
public class Member {
    @Id @GeneratedValue
    @Column(name = "MEMBER_ID")
    private Long id;
    @Column(name = "USERNAME")
    private String name;
    @ManyToOne
    @JoinColumn(name = "TEAM_ID")
    private Team team;
}
```
```java
@Entity
public class Team {
    @Id
    @GeneratedValue
    @Column(name = "TEAM_ID")
    private Long id;
    private String name;
    @OneToMany(mappedBy = "team")
    private List<Member> members = new ArrayList<>();
}
```
- Member 객체에서 Team을 조회할 수 있고, Team에서 Member를 조회할 수 있다.
```java
Member findMember = em.find(Member.class, member.getId());
List<Member> members = findMember.getTeam().getMembers();

for (Member member1 : members) {
    System.out.println("member1.getName() = " + member1.getName());
}
```

#### mappedBy
- 객체는 양방향이라기보단 회원-> 팀 , 팀 -> 회원 형태의 단방향 2개 이고, 테이블은 회원 <-> 팀 처럼 양방향이다.
- 객체는 둘 중 하나로 외래 키를 관리 해야하는데 두 관계중 하나를 연관관계의 주인으로 지정한다.
- 연관관계 주인만이 외래 키를 관리(등록, 수정) 할 수 있고, 나머지는 읽기만 가능하다.
- 주인은 mappedBy 속성을 사용 하지않고 주인이 아닐때만 mappedBy 속성으로 주인을 지정한다.
- 외래 키가 있는 곳을 주인으로 정해라.(N 쪽이 무조건 주인이 된다.)


<br>

### 4. 양방향 연관관계와 연관관계의 주인 - 주의점
___
```java
Member member = new Member();
member.setName("member1");
em.persist(member);

Team team = new Team();
team.setName("Team A");
team.getMembers().add(member);  // 연관관계 주인에 값을 입력하지 않음
em.persist(team);
```
- member의 team_id 값은 null 이다. (주인이 아닌 곳에 값을 입력 members 는 읽기 전용)
- 주인에 값을 입력 해주어야 team_id가 null 이아니다.
```java
Team team = new Team();
team.setName("Team A");
em.persist(team);

Member member = new Member();
member.setName("member1");
member.setTeam(team);
em.persist(member);
```
- 순수 객체 상태를 고려하여 항상 양쪽에 값을 설정하는게 좋다.
- 매 번 getMembers().add(team) 이런 코드를 쓰지말고 연관관계 편의 메소드를 만들어준다.
```java
public void setTeam(Team team) {
        this.team = team;
        team.getMembers().add(this);
    }
```
- 무한 루프 조심(toString(), lombok, JSON 생성 라이브러리)
  - team, members 둘아 toString 함수 호출하면 무한 루프가 돈다.

#### 양방향 매핑 정리
- 단방향 매핑만으로도 이미 연관관계 매핑은 완료
- 양방향 매핑은 반대 방향으로 조회(객체 그래프 탐색) 기능이 추가된 것 뿐
  - JPQL에서 역방향으로 탐색할 일이 많음
- 단방향 매핑을 잘 하고 양방향은 필요할 때 추가해도 됨 (테이블에 영향을 주지 않음)
- 비즈니스 로직을 기준으로 연관관계의 주인을 선택하면 안됨 => 외래키의 위치를 기준으로 정해야한다.
