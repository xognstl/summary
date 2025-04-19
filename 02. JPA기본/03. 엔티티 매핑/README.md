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
