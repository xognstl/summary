## 값 타입

### 1. 기본값 타입
___
#### 엔티티 타입
- @Entity로 정의하는 객체, 데이터가 변해도 식별자로 지속해서 추적 가능하다.   
- ex) 회원 엔티티의 키나 나이 값을 변경해도 식별자로 인식 가능

### 값 타입
- int, Integer, String처럼 단순히 값으로 사용하는 자바 기본 타입이나 객체,식별자가 없고 값만 있으므로 변경시 추적 불가하다.
- 종류
    - 기본 값 타입
        - 자바 기본 타입(int, double), 래퍼 클래스(Integer, Long), String
        - 생명주기를 엔티티에 의존한다. ex) 회원을 삭제하면 이름, 나이 필드도 함 삭제
        - 값 타입은 공유하면 안된다. ex) 회원 이름 변경시 다른 회원의 이름도 함꼐 변경되면 안된다.
    - 임베디드 타입(embedded type, 복합 값 타입)
    - 컬렉션 값 타입(collection value type)

### 자바의 기본 타입은 절대 공유하지 않는다.
* int, double 같은 기본 타입(primitive type)은 절대 공유X
* 기본 타입은 항상 값을 복사함
* Integer같은 래퍼 클래스나 String 같은 특수한 클래스는 공유 가능한 객체이지만 변경X

<br>

### 2. 임베디드 타입
___
#### 임베디드 타입(복합 값 타입)
- 새로운 값 타입을 직접 정의할 수 있다. JPA에서는 임베디드 타입(embedded type)이라고 한다.
- 주로 기본 값 타입을 모아서 만들어서 복합 값 타입이라고도 한다. int, String 같은 값 타입.
- @Embeddable: 값 타입을 정의하는 곳에 표시
- @Embedded: 값 타입을 사용하는 곳에 표시
- 기본 생성자 필수

#### 임베디드 타입의 장점
- 재사용과 높은 응집도
- 해당 값 타입만을 사용하는 의미 있는 메소드를 만들 수 있다. ex) Period.isWork()
- 임베디드 타입을 포함한 모든 값 타입은, 값 타입을 소유한 엔티티에 생명주기를 의존함

```java
// Member.class
@Embedded
private Period workPeriod;
@Embedded
private Address homeAddress;

@Embeddable
public class Address {
    private String city;
    private String street;
    private String zipcode;

    public Address() {}
    public Address(String city, String street, String zipcode) {
        this.city = city;
        this.street = street;
        this.zipcode = zipcode;
    }
}

@Embeddable
public class Period {
    private LocalDateTime startDate;
    private LocalDateTime endDate;
}
```

- 임베디드 타입은 엔티티의 값일 뿐이다.
- 임베디드 타입을 사용하기 전과 후에 매핑하는 테이블은 같다.
- 객체와 테이블을 아주 세밀하게(find-grained) 매핑하는 것이 가능
- 잘 설계한 ORM 애플리케이션은 매핑한 테이블의 수보다 클래스의 수가 더 많음
- 임베디드 타입의 값이 null이면 매핑한 컬럼 값은 모두 null

### AttributeOverride
- 한 엔티티에서 같은 값 타입을 사용하면 컬럼 명이 중복되므로, @AttributeOverrides, @AttributeOverride를 사용해서 이를 해결(속성을 재정의 한다)
```java
@Embedded
private Address homeAddress;

@Embedded
@AttributeOverrides({
        @AttributeOverride(name = "city", column = @Column(name = "WORK_CITY")),
        @AttributeOverride(name = "street", column = @Column(name = "WORK_STREET")),
        @AttributeOverride(name = "zipcode", column = @Column(name = "WORK_ZIPCODE"))
})
private Address workAddress;
```

<br>

### 3. 값 타입과 불변 객체
___
#### 값 타입 공유 참조
- 임베디드 타입 같은 값 타입을 여러 엔티티에서 공유하면 부작용(side effect) 발생할 수 있 위험함
- 회원1, 2가 같은 주소를 보고 있으면 city의 값을 변경하면 같이 변경이 된다.
```java
Address address = new Address("city", "street", "1000");
Member member1 = new Member();
member1.setName("member1");
member1.setHomeAddress(address);
em.persist(member1);

Member member2 = new Member();
member1.setName("member1");
member1.setHomeAddress(address);
em.persist(member2);

member1.getHomeAddress().setCity("newCity");    // member1의 city 만 변경하고 싶은데 실제로 돌려보면 member1,2가 둘다 변경된다.
```

#### 값 타입 복사
- 값 타입의 실제 인스턴스 값을 공유하는 것은 위험하고, 대신 값(인스턴스)를 복사해서 사용
```java
Address address = new Address("city", "street", "1000");
Member member1 = new Member();
member1.setName("member1");
member1.setHomeAddress(address);
em.persist(member1);

Address copyAddress = new Address(address.getCity(), address.getStreet(), address.getZipcode());
Member member2 = new Member();
member1.setName("member1");
member1.setHomeAddress(copyAddress);
em.persist(member2);

member1.getHomeAddress().setCity("newCity");

tx.commit();    // 2번만 변경 된다.
```
- 항상 값을 복사해서 사용하면 공유 참조로 인해 발생하는 부작용을 피할 수 있다.
- 임베디드 타입처럼 직접 정의한 값 타입은 자바의 기본 타입이 아니라 객체 타입이다.
- 자바 기본 타입에 값을 대입하면 값을 복사한다.
- 객체 타입은 참조 값을 직접 대입하는 것을 막을 방법이 없다.
- 객체의 공유 참조는 피할 수 없다.

#### 결론
- 객체 타입을 수정할 수 없게 만들면 부작용을 원천 차단 할 수 있다. 값 타입은 불변 객체(immutable object)로 설계 해야한다.
- 생성자로만 값을 설정하고 Setter를 만들지 않으면 된다.

<br>
