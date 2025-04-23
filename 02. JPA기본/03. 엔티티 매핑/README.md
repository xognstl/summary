## 엔티티 매핑

### 엔티티 매핑 종류

- 객체와 테이블 매핑 : @Entity, @Table
- 필드와 컬럼 매핑 : @Column
- 기본키 매핑 : @Id
- 연관관계 매핑 : @ManyToOne, @JoinColumn

<br>

### 1. 객체와 테이블 매핑
___

#### Entity
- @Entity가 붙은 클래스는 JPA가 관리
- final 클래스, enum, interface, inner 클래스 사용X, 저장할 필드에 final 사용 X
- @Entity(name = "") 와 같이 엔티티 이름을 지정할 수 있지만 기본값(클래스 이름 그대로 사용)을 사용한다.

#### Table
- @Table은 엔티티와 매핑할 테이블 지정
- name : 매핑할 테이블 이름
- catalog : DB catalog 매핑
- schema : DB schema 매핑
- uniqueConstraints : DDL 생성 시에 유니크 제약 조건 생성

<br>

### 2. 데이터베이스 스키마 자동 생성
___
- DDL을 애플리케이션 실행 시점에 자동 생성
- 테이블 중심 -> 객체 중심 (환경이 다를때 DB 테이블 생성 하고 이런 작업을 안해도 된)
- 데이터베이스 방언을 활용해서 데이터베이스에 맞는 적절한 DDL 생성(ex. varchar2, varchar)

#### 속성
```xml
<property name="hibernate.hbm2ddl.auto" value="create" />
```
- create : 기존테이블 삭제 후 다시 생성 (DROP + CREATE)
- create-drop : create와 같으나 종료시점에 테이블 DROP
- update : 변경분만 반영
- validate : 엔티티와 테이블이 정상 매핑되었는지만 확인
- none : 사용하지 않음

#### DDL 생성 기능
- 제약조건 추가 : @Column(nullable = false)
- 유니크 제약 조건 추가
- DDL 생성 기능은 DDL 생성 시 사용되고 JPA 실행 로직에는 영향 X

<br>

### 3. 필드와 컬럼 매핑
___

#### @Column (컬럼 매핑)
- name : 필드와 매핑할 테이블의 컬럼 이름 (default : 객체의 필드 이름)
- insertable, updatable : 등록, 변경 가능 여부 (default : true)
- nullable : null 값 허용 여부 설정, false로 설정시 not null 제약조건이 붙는다.
- unique : @Table의 uniqueConstraints와 같지만 한 컬럼에 간단히 유니크 제약조건을 걸 때 사용
- columnDefinition : DB 컬럼 정보를 직접 줄 수 있다. ex) varchar(100) default ‘EMPTY'
- length : 문자 길이 제약 조건 (default : 255)
- precision, scale : BigDecimal 타입에서의 소수점 관련  

#### @Enumerated (enum 타입 매핑)
- EnumType.ORDINAL : enum 의 순서를 DB 에 저장
- EnumType.STRING : enum 이름을 DB 에 저장
- ORDINAL 사용 X => USER, ADMIN 에서 GUEST, USER, ADMIN 로 guest를 앞에 추가하면 guest가 0번이 되는데 기존 데이터는 USER 가 0 이기 떄문에 섞인다. 
```java
public enum RoleType {
    USER, ADMIN
}

@Enumerated(EnumType.STRING) 
private RoleType roleType;

Member member = new Member();
member.setId(1L);
member.setUsername("A");
member.setRoleType(RoleType.USER);
// EnumType.ORDINAL: enum 순서를 데이터베이스에 저장, 위의 코드에서 데이터에 0이 저장된다.
// EnumType.STRING: enum 이름을 데이터베이스에 저장, 위의 코드에서 데이터에 USER가 저장된다.
em.persist(member);
```

#### @Temporal (날짜 타입 매핑)
- 날짜 타입(java.util.Date, java.util.Calendar)을 매핑할 때 사용
- LocalDate, LocalDateTime을 사용할 때는 생략 가능
- TemporalType.DATE : 2023-05-03
- TemporalType.TIME : 20:16:00
- TemporalType.TIMESTAMP : 2023-05-03 20:16:00

#### @Lob (BLOB, CLOB 매핑)
- 지정할 수 있는 속성 X, 매핑하는 필드 타입이 문자면 CLOB, 나머지는 BLOB 매핑

#### @Transient
- 필드 매핑 X, DB에 저장 조회 X, 메모리상에서 임시로 어떤 값 보관할 때 정도로 사용

<br>

### 4. 기본 키 매핑
___
#### 직접 할당
- @Id 만 사용

#### 자동 생성(@GeneratedValue)
##### Auto
- 각 DB 문법에 따라 자동 지정 된다. (기본값)
```java
@Id @GeneratedValue(strategy = GenerationType.AUTO)
private Long id;
```

##### IDENTITY
- 기본 키 생성을 데이터베이스에 위임
- 주로 MySQL, PostgreSQL, SQL Server, DB2에서 사용 (ex : MySQL의 AUTO_ INCREMENT)
```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```
- 특징
```text
id에 값을 넣지 않은 상태에서 DB에 insert 를 해야한다. 
DB에서 null로 insert 쿼리가 날라오면 그때 값을 세팅해준다.   
id값을 알 수 있는 시점은 DB에 값이 들어간 후다.   
identity 에서만 em.persist(member); 를 호출 하자마자 DB에 insert 쿼리를 날린다.(원래는 commit)   
insert 후 jpa 내부적으로 영속성 컨텍스트의 pk값으로 쓰게 된다.   
Batch 처럼 insert를 모아서 쿼리를 날리는 것을 할 수 없는 것이 단점이다.   
```

##### SEQUENCE
- 데이터베이스 시퀀스는 유일한 값을 순서대로 생성하는 특별한 데이터베이스 오브젝트(예: 오라클 시퀀스)
- 오라클, PostgreSQL, DB2, H2에서 사용
```java
@Entity
@SequenceGenerator(
 name = “MEMBER_SEQ_GENERATOR",
 sequenceName = “MEMBER_SEQ", //매핑할 데이터베이스 시퀀스 이름
 initialValue = 1, allocationSize = 1)
public class Member {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE,
    generator = "MEMBER_SEQ_GENERATOR")
    private Long id;
```
- SEQUENCE -- @SequenceGenerator
  - name : 식별자 생성기 이름(필수 값)
  - sequenceName : 데이터베이스에 등록되어 있는 시퀀스 이름 (default : hibernate_sequence)
  - initialValue : DDL 생성 시에만 사용됨, 시퀀스 DDL을 생성할 때 처음 1 시작하는 수를 지정한다.
  - allocationSize : 시퀀스 한 번 호출에 증가하는 수(성능 최적화에 사용됨) 데이터베이스 시퀀스 값이 하나씩 증가하도록 설정되어 있으면 이 값을 반드시 1로 설정해야 한다.
  - catalog, schema : 데이터베이스 catalog, schema 이름

- 특징
```text
persist를 하려면 영속성 컨텍스트에 일단 넣어야하고, 그러면 PK가 필요하다.   
PK값을 알려면 sequence 값이 필요하고 DB에서 sequence를 가져와야한다.   
JPA 에서는 call next value for SEQ_NAME 을 통해 식별자 값을 가져온다.   
initialValue = 1, allocationSize = 50 으로 설정을 해놓으면 (create sequence MEMBER_SEQ start with 1 increment by 50)   
원래는 persist 할때마다 next call을 하는데 성능 저하가 있을 수 있다. allocationSize = 50 이면 메모리에 미리 50개를 할당 받아 놓는다.   

```

##### TABLE
- 키 생성 전용 테이블을 하나 만들어서 데이터베이스 시퀀스를 흉내내는 전략
- 장점은 모든 DB에 적용 가능하지만 단점은 성능이 좋지 않다.
- @TableGenerator
    - name : 식별자 생성기 이름(필수값)
    - table 키생성 테이블명 (default : hibernate_sequences)
    - pkColumnName 시퀀스 컬럼명 (default : sequence_name)
    - valueColumnName : 시퀀스 값 컬럼명 (default : next_val)
    - pkColumnValue : 키로 사용할 값 이름 엔티티 이름
    - initialValue 초기 값, 마지막으로 생성된 값이 기준이다. (default : 0)
    - allocationSize 시퀀스 한 번 호출에 증가하는 수(성능 최적화에 사용됨) (default : 50)
    - catalog, schema 데이터베이스 catalog, schema 이름
    - uniqueConstraints : 유니크 제약 조건을 지정할 수 있다

