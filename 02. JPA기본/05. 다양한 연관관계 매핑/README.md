## 다양한 연관관계 매핑

### 1. 연관관계 매핑시 고려사항 
___
#### 다중성
- 다대일 : @ManyToOne
- 일대다 : @OneToMany
- 일대일 : @OneToOne
- 다대다 : @ManyToMany (거의 사용 X)

#### 단방향, 양방향
- 테이블
    - 외래키 하나로 양쪽 조인 가능(방향이라는 개념이 없음)
    
- 객체
    - 참조용 필드가 있는 쪽으로만 참조 가능
    - 한쪽만 참조하면 단방향, 양쪽은 양방향
  
#### 연관관계의 주인
- 객체 양방향 관계는 A -> B, B -> A 참조가 두군데, 테이블은 외래키 하나로 연관관계를 맺음
- 객체는 둘중 테이블의 외래 키를 관리할 곳을 지정해야 한다.
- 연관관계의 주인은 외래 키를 관리하는 참조, 반대편은 외래키에 영향X, 단순 조회만 가능

<br>

### 2. 다대일 [N:1]
___
- MEMBER N : TEAM 1 , N쪽에 외래키가 있어야 한다.
- 다대일 단방향은 Team 에서는 Member를 참조 하지 않는다.
- 다대일 양방향 : 외래키가 있는 쪽이 연관관계의 주인 
```java
//Member.class
@Id @GeneratedValue
@Column(name = "MEMBER_ID")
private Long id;

@Column(name = "USERNAME")
private String username;

@ManyToOne
@JoinColumn(name = "TEAM_ID")
private Team team;

//Team.class
@Id @GeneratedValue
@Column(name = "TEAM_ID")
private Long id;
private String name;

// 다대일 양방향, 테이블에 연관이 없고, 조회만 가능 
//    @OneToMany(mappedBy = "team")
//    private List<Member> members = new ArrayList<>();
```

<br>

### 3. 일대다 [1:N]
___
#### 일대다 단방향
- 거의 사용 X
- 일대다 단방향은 1이 연관관계 주인, 테이블에서는 항상 N쪽에 외래 키가 있음.
- @JoinColumn을 꼭 사용해야 함. 그렇지 않으면 조인 테이블 방식을 사용함(중간에 테이블을 하나 추가함)
```java
//Member.class
@Id @GeneratedValue
@Column(name = "MEMBER_ID")
private Long id;
@Column(name = "USERNAME")
private String username;

//Team.class
@Id @GeneratedValue
@Column(name = "TEAM_ID")
private Long id;
private String name;

@OneToMany
@JoinColumn(name = "TEAM_ID")
private List<Member> members = new ArrayList<>();
```
```java
Member member = new Member();
member.setUsername("member1");
em.persist(member);

Team team = new Team();
team.setName("teamA");
team.getMembers().add(member);  // 이부분
em.persist(team);
```
#### 일대다 단방향 매핑의 단점
- 위와 같은 코드 작성시 insert, insert, update 쿼리가 발생한다. 쿼리가 한번더 발생한다.
- 엔티티가 관리하는 외래 키가 다른 테이블에 있다.
- 일대다 단방향 매핑보다는 다대일 양방향 매핑 사용

#### 일대다 양방향
- 읽기 전용 필드를 사용해서 양방향 처럼 사용 ==> 이런게 있다만 알고 사용하지말자.
```java
@ManyToOne
@JoinColumn(name="TEAM_ID", insertable=false, updatable=false)
private Team team;
```

<br>

### 4. 일대일 [1:1]
___
- 주 테이블이나 대상 테이블 중 외래 키 선택 가능
- 외래 키에 데이터베이스 유니크 제약조건(다대일에 유니크 제약조건이 추가된것과 같다.)
- 양방향 매핑 시 주 테이블의 외래 키가 있는 곳이 연관관계의 주인, 반대편은 mappedBy 적용 
```java
//Member.class
@Id @GeneratedValue
@Column(name = "MEMBER_ID")
private Long id;
@Column(name = "USERNAME")
private String name;

@OneToOne
@JoinColumn(name = "LOCKER_ID")
private Locker locker;

//Locker.class
@Id @GeneratedValue
private Long id;
private String name;

@OneToOne(mappedBy = "locker")
private Member member;  // 일대일 양방향
```
- 주 테이블에 외래 키(Member)
    - 장점: 주 테이블만 조회해도 대상 테이블에 데이터가 있는지 확인 가능
    - 단점: 값이 없으면 외래 키에 null 허용

- 대상 테이블에 외래 키(Locker)
    - 장점: 주 테이블과 대상 테이블을 일대일에서 일대다 관계로 변경할 때 테이블 구조 유지
    - 단점: 프록시 기능의 한계로 지연 로딩으로 설정해도 항상 즉시 로딩됨

