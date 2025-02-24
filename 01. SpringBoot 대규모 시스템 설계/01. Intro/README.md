## Intro

### 1. 프로젝트 요구사항
___
- 게시글 조회, 생성, 수정, 삭제, 목록 조회
- 댓글 조회, 생성, 삭제, 목록 조회 (계층형 - 최대 2depth, 무한 depth)
- 게시글 좋아요(좋아요 수)
- 게시글 조회수 (10분회 1회 집계)
- 인기글(일 단위 상위 10건)
  - 매일 오전 1시 업데이트
  - 댓글수/좋아요수/조회수 기반
- 최근 N일 인기글 내역 조회
- 게시글 조회 최적화 전략
  - 게시글 단건 조회 최적화 전략
  - 게시글 목록 조회 최적화 전략
  - 캐시 최적화 전략


- 대규모 데이터와 트래픽을 지탱할 수 있는 시스템

<br>

### 2. 인프라 구성
___
- Spring Boot 는 Client 의 요청을 처리하면 상태를 관리해야한다. 이러한 데이터는 DB에 의해 안전하게 관리 된다.
- 서비스가 활성화되면서 Client 의 요청이 많아지면 서버의 리소스 부족으로 인해 단일 애플리케이션은 더이상 감당할 수 없어진다.
- Spring Boot 애플리케이션을 여러서버에서 동시에 실행하여 처리를 분산할 수 있다.(Scale-Out)
  - Scale-out : 서버를 추가하여 성능을 향상(수평 확장)
- 트래픽을 라우팅 , 분산하기 위한 도구로 Load Balancer 를 활용할 수 있다.  
Client 는 Load Balancer 로 요청을 보내면, Load Balancer 는 요청을 적절히 분산하여 서버로 전달한다.


- Client 가 요청을 처리하는 과정이 복잡해질수록 응답은 느려질 수 있다. 빠른 성능을 위해 캐시라는 기술을 사용
- Cache : 더 느린 저장소에서 더 빠른 저장소에 데이터를 저장해두고 접근하는 기술
- 단일 애플리케이션을 기능에 맞는 여러 개의 애플리케이션으로 분리하여 관리할 수 있다.
- 분리된 웹 애플리케이션이 하나의 서비스를 이루려면 서로 네트워크 통신이 필요할 수 있다.  
API를 통해 직접통신을 할 수 도 있지만, 메시지(이벤트) 를 통해 간접적으로 통신할 수도 있다.


<br>

### 3. 시스템 아키텍처 - Monolithic Architecture
___
- 시스템 아키텍처는 시스템의 구조나 설계방식
- Monolithic Architecture : 애플리케이션의 모든 기능이 하나로 통합된 아키텍쳐
- 만약 한 기능의 트래픽이 많아지면?? 부하를 감당하기 위해 Scale-Up, 단일서버의 성능을 향상(수직확장)
- 수직 확장은 한계가 있으므로 서버를 추가하여 부하를 분산하는 Scale-Out
- 모든 기능이 단일 애플리케이션에서 함께 배포되므로, 부하가 오는 기능말고도 다른기능도 같이 배포되어 리소스를 점유
- 한 기능만 변경을 해도 모든 기능이 다시 배포되어야 한다. 애플리케이션이 무거워질수록 빌드, 배포시간은 늘어난다.


#### Monolithic Architecture
- 모든 기능이 단일 코드베이스로 결합된 아키텍쳐
- 소규모 시스템에서 개발 및 배포가 간단하기 때문에 사용
- 특정 부분만 확장하기 어렵고, 변경 사항이 시스템 전체에 영향을 미친다. 대규모 시스템에서 복잡도가 커지고 개발이 어려움

<br>

### 4. 시스템 아키텍처 - Microservice Architecture
___
- 시스템이 작고 독립적인 서비스로 구성
- 각 서비스는 단일 기능 담당, 독립적인 배포
- 서비스 단위로 유연한 확장 가능


- 각 서비스의 코드는 독립적으로 관리 및 배포, 개별 서비스의 복잡도가 낮아지고, 빠르게 빌드 및 배포를 할 수 있다.
- 한 기능의 트래픽이 증가하면 그 기능의 서버만 증설 => 해당 기능 리소스만 추가 점유
- 서비스 간 통신 비용, 트랜잭션 관리, 서비스 분리 기준, 모니터링, 개발비용, 테스트, 설계 등 고려해야할 부분이 많아진다.

<br>

### 5. Docker
___
- Docker : 애플리케이션을 컨테이너라는 격리된 환경에서 실행해주는 플랫폼
- 쉽게 배포하고 관리, 환경에 구애 받지 않는다.
- 프로그램은 이미지로 패키징되고, 컨테이너로 실행된다.
  - 이미지 : 실행파일
  - 컨테이너 : 실행된 프로세스,이미지, 독립적이고 격리된 실행 환경
 


<br>

### 6. 프로젝트 세팅
___
- java 21 , Spring boot 3.3.2 Gradle
- dependency : Spring Web, Lombok
- 멀티 모듈 구조 사용 : 프로젝트를 여러 개의 독립적인 모듈로 분리하여 개발
  - 각 모듈마다 설정들을 지정 , appprojects 로 감싸주면 모든 모듈에 똑같이 설정
```yaml
allprojects {
    repositories {
        mavenCentral()
    }

    dependencies {
        implementation 'org.springframework.boot:spring-boot-starter-web'

        compileOnly 'org.projectlombok:lombok'
        annotationProcessor 'org.projectlombok:lombok'

        //테스트에서 lombok 사용
        testCompileOnly 'org.projectlombok:lombok'
        testAnnotationProcessor 'org.projectlombok:lombok'

        testImplementation 'org.springframework.boot:spring-boot-starter-test'
        testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
    }

    tasks.named('test') {
        useJUnitPlatform()
    }
}
```
- src 제거 후 common(공통 코드 관리), service(microservice) 디렉토리 생성
- service 밑에 하위 모듈 생성, article 에만 build.gradle 웹의존성 추가
```yaml
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-web'
}
```
- setting.gradle
```yaml
include 'common'
include 'service'
include 'service:article'
```

- service 밑에 article 생성 후 src, main, test 등 프로젝트에 필요한것 생성 
- article, comment, like, view, hot-article, article-read 프로젝트 복사
