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
