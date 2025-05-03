## 고급 매핑

### 1. 상속관계 매핑
___
#### 상속관계 매핑
- 객체에는 상속관계가 있으나 관계형 데이터베이스에는 상속 관계가 없다.
- 슈퍼타입, 서브타입 관계라는 모델링 기법이 객체 상속과 유사하다.
- 상속관계 매핑 : 객체의 상속구조와 DB의 슈퍼타입 서브타입 관계를 매핑
- 슈퍼타입 서브타입 논리 모델을 실제 물리 모델로 구현하는 방법
  - 조인 전략 : 각각 테이블로 변환
  - 단일 테이블 전략 : 통합 테이블로 변환
  - 구현 클래스마다 테이블 전략 : 서브타입 테이블로 변환

#### 주요 어노테이션
- @Inheritance(strategy=InheritanceType.XXX)
  - JOINED (조인 전략), SINGLE_TABLE (단일 테이블 전략), TABLE_PER_CLASS (구현 클래스마다 테이블 전략)
  - 조인 전략 -> 단일 테이블 전략 이런식으로 전략을 바꾸려면 어노테이션만 바꿔주면 된다.
- @DiscriminatorColumn(name=“DTYPE”) : default = DTYPE
- @DiscriminatorValue(“XXX”) : 자식 테이블의 기본 명(default : 엔티티 명)

#### 조인 전략
- 장점
  - 테이블 정규화, 외래 키 참조 무결성 제약조건 활용 가능, 저장 공간 효율화
  - 주문에서 외래키 참조할때 ITEM만 보면 된다.
- 단점
  - 조회시 조인을 사용하기 떄문에 성능이 저하되고, 쿼리가 복잡하다.
  - 데이터 저장시 INSERT 2번 호출

#### 단일 테이블 전략
- @DiscriminatorColumn 어노테이션을 생략해도 자동으로 들어간다.
- 장점
  - 조인이 필요없으므로 조회 성능이 빠르고, 쿼리가 단순하다.
- 단점
  - 자식 엔티티가 매핑한 컬럼은 모두 null 허용
  - 단일 테이블에 모든 것을 저장하므로 테이블이 커질 수 있다. 상황에 따라서 조회 성능이 오히려 느려질 수 있다.

### 구현 클래스마다 테이블 전략
- 사용하지 않는다.
- - 장점
- 서브 타입을 명확하게 구분해서 처리할 때 효과적, not null 제약조건 사용 가능
- 단점
  - 여러 자식 테이블을 함께 조회할 때 성능이 느림(UNION SQL 필요)
  - 자식 테이블을 통합해서 쿼리하기 어려움

```java
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
@DiscriminatorColumn
public class Item {
    @Id @GeneratedValue
    private Long id;
    private String name;
    private int price;
}
@Entity
//@DiscriminatorValue("A") //DTYPE 값을 엔티티명이 아닌 다른것으로 바꿀 수 있다. 
public class Album extends Item {
  private String artist;
}

@Entity
public class Movie extends Item {
  private String director;
  private String actor;
}
@Entity
public class Book extends Item {
  private String author;
  private String isbn;
}
```

```java
Movie movie = new Movie();
movie.setDirector("Aaaa");
movie.setActor("bbbb");
movie.setName("네임");
movie.setPrice(10000);

em.persist(movie);
em.flush();
em.clear();

Movie findMovie = em.find(Movie.class, movie.getId());
System.out.println("findMovie = " + findMovie);
// 위와같이 movie에 등록을하면 Item, movie 에 둘다 insert가 된다. 
// 조회를 하면 알아서 조인을 해서 select 한다.   
```

<br>

### 2. Mapped Superclass - 매핑 정보 상속
___
- 상속관계 매핑X, 엔티티X, 테이블과 매핑X
- 부모 클래스를 상속 받는 자식 클래스에 매핑 정보만 제공
- 조회, 검색 불가(em.find(BaseEntity) 불가)
- 직접 생성해서 사용할 일이 없으므로 추상 클래스 권장
- 테이블과 관계 없고, 단순히 엔티티가 공통으로 사용하는 매핑정보를 모으는 역할, DB 테이블은 다르지만 객체 입장에서 속성만 상속받아 쓰고싶을 때 사용
- 주로 등록일, 수정일, 등록자, 수정자 같은 전체 엔티티에서 공통으로 적용하는 정보를 모을 때 사용
- @Entity 클래스는 엔티티나 @MappedSuperclass로 지정한 클래스만 상속 가능
- 
```java
@MappedSuperclass
public abstract class BaseEntity {
    private Long createdBy;
    private LocalDateTime createdDate;
    private Long lastModifiedBy;
    private LocalDateTime lastModifiedDate;
}

@Entity
public class Member extends BaseEntity{
}
@Entity
public class Item extends BaseEntity{
}
//member.setCreatedBy(""); 가 가능해진다. 
```
