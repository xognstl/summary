## 프록시와 연관관계 관리

### 1. 프록시
___
```java
Member member = em.find(Member.class, 1L);
printMemberAndTeam(member);
printMember(member);

private static void printMember(Member member) {
    System.out.println("member = " + member.getName());
}

private static void printMemberAndTeam(Member member) {
    String username = member.getName();
    System.out.println("username = " + username);

    Team team = member.getTeam();
    System.out.println("team = " + team);
}
```
- Member와 Team을 함께 출력을 할때도 있고, 회원만 출력하려고 할때도 있다.  
  하지만 두 케이스 다 상관관계 떄문에 Team까지 출력하기 떄문에 별로이다. 이럴때 프록시나 지연로딩을 사용한다.

#### 프록시 기초
- em.find(): 데이터베이스를 통해서 실제 엔티티 객체 조회
- em.getReference(): 데이터베이스 조회를 미루는 가짜(프록시) 엔티티 객체 조회
```java
Member member = new Member();
member.setName("hello");
em.persist(member);
em.flush();
em.clear();

Member findMember = em.find(Member.class, member.getId()); // member, team join 해서 가져옴
Member findMember = em.getReference(Member.class, member.getId());    // select 쿼리 XX
// class hellojpa.Member$HibernateProxy$j5XCQgyk
System.out.println("findMember = " + findMember.getClass());    

// reference 호출하면 파라미터에 Id가 있기 때문에 select X
System.out.println("findMember.getName() = " + findMember.getId());   

// reference 호출하면 여거 조회할때 select 호출
System.out.println("findMember.getName() = " + findMember.getName());   
```
- 실제 클래스를 상속 받아서 만들어지고, 겉 모양이 같다.
- 프록시 객체는 실제 객체의 참조(target)를 보관
- 프록시 객체를 호출하면 프록시 객체는 실제 객체의 메소드 호출

#### 프록시 객체의 초기화
![화면 캡처 2023-05-10 222949](https://github.com/xognstl/JPA1/assets/48784785/eb48f970-cd0e-4d45-a76c-7a307b7a5738)

- 처음에 getName을 호출하면 Member target 에 값이 없으므로 영속성 컨텍스트에 초기화 요청을 하고  
  실제 엔티티 객체를 생성한다.  그 후 target에 진짜 객체를 연결해준다.

#### 프록시의 특징
- 프록시 객체는 처음 사용할 때 한 번만 초기화
```java
Member findMember = em.getReference(Member.class, member.getId());
System.out.println("findMember.getUsername() = " + findMember.getUsername()); // 여기에서 초기화 요청
System.out.println("findMember.getUsername() = " + findMember.getUsername());
```

<br>

- 프록시 객체를 초기화 할 때, 프록시 객체가 실제 엔티티로 바뀌는 것은 아님, 초기화되면 프록시 객체를 통해서 실제 엔티티에 접근 가능
- before, after 같다.
```java
Member findMember = em.getReference(Member.class, member.getId());
System.out.println("before findMember.getClass() = " + findMember.getClass()); // 프록시
System.out.println("findMember.getUsername() = " + findMember.getUsername());
System.out.println("after findMember.getClass() = " + findMember.getClass()); // 프록시
```

<br>

- 프록시 객체는 원본 엔티티를 상속받음, 따라서 타입 체크시 주의해야함 (== 비교 실패, 대신 instance of 사용)
```java
Member m1 = em.find(Member.class, member1.getId());
Member m2 = em.find(Member.class, member2.getId());
System.out.println("m1 == m2 : " + (m1.getClass() == m2.getClass()));   //true


Member m1 = em.find(Member.class, member1.getId());
Member m2 = em.getReference(Member.class, member2.getId());
logic(m1,m2);
            
public static void logic(Member m1, Member m2) {
    System.out.println("m1 == m2 : " + (m1.getClass() == m2.getClass()));   //false
    System.out.println("m1 == m2 : " + (m1 instanceof Member)); //true
    System.out.println("m1 == m2 : " + (m2 instanceof Member)); //true
}
```

<br>

- 영속성 컨텍스트에 찾는 엔티티가 이미 있으면 em.getReference()를 호출해도 실제 엔티티 반환
```java
Member m1 = em.find(Member.class, member1.getId());
System.out.println("m1.getClass() = " + m1.getClass()); //class hellojpa.Member
Member reference = em.getReference(Member.class, member1.getId());
System.out.println("reference.getClass( = " + reference.getClass()); //class hellojpa.Member
System.out.println("a==a : " + (m1 == reference));  //true => jpa 에서는 이부분을 항상 참으로 해준다.
```
** 반대로 getReference, find 순서로 오면 두 class 타입다 Proxy 이다.

<br>

- 영속성 컨텍스트의 도움을 받을 수 없는 준영속 상태일 때, 프록시를 초기화하면 문제 발생   
  (하이버네이트는 org.hibernate.LazyInitializationException 예외를 터트림)
```java
Member refMember = em.getReference(Member.class, member1.getId());
System.out.println("reference.getClass( = " + refMember.getClass()); //class hellojpa.Member
em.detach(refMember);   //준영속상태
//원래는 DB에 쿼리가 나가면서 Proxy 객체가 초기화됨. 하지만 준영속 상태로 만들면 에러가 난다.
System.out.println("refMember.getUsername() = " + refMember.getUsername());
```

<br>

#### 프록시 확인
- 프록시 인스턴스의 초기화 여부 확인 : PersistenceUnitUtil.isLoaded(Object entity)
```java
Member refMember = em.getReference(Member.class, member1.getId());
refMember.getUsername(); // getUsername 이 있으면 초기화 true, 없으면 false
System.out.println("isLoad = " + emf.getPersistenceUnitUtil().isLoaded(refMember));

```
<br>

- 프록시 클래스 확인 방법 : entity.getClass().getName() 출력(..javasist.. or HibernateProxy…)
- 프록시 강제 초기화 : org.hibernate.Hibernate.initialize(entity);
  ** 참고: JPA 표준은 강제 초기화 없음 강제 호출: member.getName()
```java
Hibernate.initialize(refMember);    //강제 초기화
```

<br>

### 2. 즉시 로딩과 지연 로딩
___
#### 지연 로딩
- Member를 조회할때 Team을 함께 조회할 필요가 자주 없을 때 사용
- 단순히 member 정보만 사용하는 비지니스 로직에 사용(member.getName);
```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "TEAM_ID")
private Team team;

Team team = new Team();
team.setName("teamA");
em.persist(team);

Member member1 = new Member();
member1.setName("member1");
member1.setTeam(team);
em.persist(member1);

em.flush();
em.clear();

Member m = em.find(Member.class, member1.getId());
System.out.println("m = " + m.getTeam().getClass());        // team은 프록시에서 조회

System.out.println("===");
m.getTeam().getName();      // 실제 team을 사용하는 시점에 DB 조회.(초기화)
System.out.println("===");
```
* team의  fetch를 LAZY로 설정하면 Member 만 DB에서 조회하고, member에 참조되있는 team은 프록시로 조회를 한다.

#### 즉시 로딩
- Member를 조회할때 Team을 함께 조회할 때 사용
- 즉시로딩으로 조회를 하면 member와 team을 조인해서 한번에 가져온다.

#### 프록시와 즉시로딩 주의점
- 가급적 지연 로딩만 사용
- 즉시 로딩을 적용하면 예상하지 못한 SQL이 발생
- 즉시 로딩은 JPQL에서 N+1 문제를 일으킨다.
- @ManyToOne, @OneToOne은 기본이 즉시 로딩 -> LAZY로 설정
- @OneToMany, @ManyToMany는 기본이 지연 로딩

```java
List<Member> members = em.createQuery("select m from Member m", Member.class).getResultList();
// 쿼리가 2번 날라간다.(member, team)
// 지연로딩으로 바꾸고 fetch join을 사용하여 쿼리가 2개 나가는 것에 대한 것을 해결.
```
```text
n + 1 : member1(teamA), member2(teamB) 이렇게 있으면 
select * from member => 전체 멤버 가져옴
select * from team where team_id = teamA
select * from team where team_id = teamB 
이런식으로 team을 조회하는 쿼리가 각각나간다. 
최초 쿼리를 하나를 날렸는데 그것 떄문에 추가 쿼리가 n개가 나간다.

==> N + 1 해결 방법
- 모든 연관관계를 지연로딩 후
1. JPQL 페치 조인 : 동적으로 원하는 것을 선택해서 한번에 가져옴
2. Entity Graph 어노테이션
3. 배치 사이즈
```

<br>

### 3. 영속성 전이(CASCADE)와 고아 객체
___
- 특정 엔티티를 영속 상태로 만들 때 연관된 엔티티도 함께 영속 상태로 만들도 싶을 때
- 예: 부모 엔티티를 저장할 때 자식 엔티티도 함께 저장
- 영속성 전이는 연관관계를 매핑하는 것과 아무 관련이 없음
- 엔티티를 영속화할 때 연관된 엔티티도 함께 영속화하는 편리함을 제공할 뿐
- ALL, PERSIST, REMOVE, MERGE, REFRESH, DETACH


```java
@Entity
public class Parent {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @OneToMany(mappedBy = "parent", cascade = CascadeType.ALL)
    private List<Child> childList = new ArrayList<>();

    public void addChild (Child child) {
        childList.add(child);
        child.setParent(this);
    }
}
@Entity
public class Child {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @ManyToOne
    @JoinColumn(name = "parent_id")
    private Parent parent;
}

Child child1 = new Child();
Child child2 = new Child();

Parent parent = new Parent();
parent.addChild(child1);
parent.addChild(child2);

em.persist(parent);
//em.persist(child1);
//em.persist(child2); //cascade 옵션을 주면 이부분을 써주지 않아도 persist 가 됨.
```

#### 고아 객체
- 고아 객체 제거: 부모 엔티티와 연관관계가 끊어진 자식 엔티티를 자동으로 삭제
- orphanRemoval = true

```java
Child child1 = new Child();
Child child2 = new Child();

Parent parent = new Parent();
parent.addChild(child1);
parent.addChild(child2);

em.persist(parent);
em.flush();
em.clear();

Parent findParent = em.find(Parent.class, parent.getId());
findParent.getChildList().remove(0);
```
- 참조가 제거된 엔티티는 다른 곳에서 참조하지 않는 고아 객체로 보고 삭제하는 기능
- 참조하는 곳이 하나일 때 사용해야함!
- 특정 엔티티가 개인 소유할 때 사용
- @OneToOne, @OneToMany만 가능
- 참고: 개념적으로 부모를 제거하면 자식은 고아가 된다. 따라서 고아 객체 제거 기능을 활성화 하면, 부모를 제거할 때 자식도 함께 제거된다. 이것은 CascadeType.REMOVE처럼 동작한다.

#### 영속성 전이 + 고아 객체, 생명주기
- CascadeType.ALL + orphanRemoval=true
- 스스로 생명주기를 관리하는 엔티티는 em.persist()로 영속화, em.remove()로 제거
- 두 옵션을 모두 활성화 하면 부모 엔티티를 통해서 자식의 생명주기를 관리할 수 있음
- 도메인 주도 설계(DDD)의 Aggregate Root개념을 구현할 때 유용
