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
