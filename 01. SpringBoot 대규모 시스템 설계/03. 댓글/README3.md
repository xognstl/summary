### 6. 댓글 무한 depth - 테이블&구현 설계
___
- 댓글 목록 조회 – 무한 depth
```text
상위 댓글은 항상 하위 댓글보다 먼저 생성되고, 하위 댓글은 상위 댓글 별 순서대로 생성된다.
위 규칙은, 무한 depth 에서는 상하위 댓글이 재귀적으로 무한할 수 있으므로, 정렬 순을 나타내기 위해 모든 상위 댓글의 정보가 필요할 수 있다.

parent_comment_id = 1 > 2 > 6 이러한 형식으로 상위 댓글의 정보를 표시
댓글 구조를 트리로 본다면, root node(1 depth 최상위 댓글) 부터 각 node(각 depth 댓글)에 도달하기 위한 경로로 이해해볼 수 있다.
각 depth 만큼 컬럼을 만들어서, 각 depth별 댓글 ID를 관리해줄 수도 있지만, 테이블 구조가 복잡해지고 무한 depth라 그만큼 컬럼 개수를 생성 X

그래서 각 depth에서의 순서를 문자열로 나타내고, 이러한 문자열을 순서대로 결합하여 경로를 나타내는 것이다. => Path Enumeration(경로 열거) 방식

위와 같이 각 depth 별로 5개의 문자열로 경로 정보를 저장한다.
1 depth는 5개의 문자로, 2 depth는 10개의 문자로, 3 depth는 15개의 문자로, N depth는 (N * 5)개의 문자로, 문자열로 모든 상위 댓글에서 각 댓글까지의 경로를 저장하는 것이다.
그리고, 각 경로는 상위 경로를 상속하며 독립적이고 순차적인 경로를 가지도록 한다.

1 depth : 00000 , 2 depth 00000 00000 , 3 depth 00000 00000 00000
각 경로는 상위 댓글의 경로를 상속하며, 각 댓글마다 독립적이고 순차적인(문자열 순서) 경로가 생성된다.

표현 범위가 제한되므로 표현할 수 있는 경로의 범위가 각 경로 별로 10^5 = 100,000개로 제한된다.
문자열 순서 = 0~9 < A-Z < a-z 이렇게 섞어서 쓰면 더 많이 사용 할 수 있다. => 9억개
우리는 문자열에 대해 순서를 정의하고, 인덱스를 사용하여 정렬된 데이터를 관리해야한다.
```
#### 데이터베이스 collation
- 문자열을 정렬하고 비교하는 방법을 정의하는 설정 : 이는 대소문자 구분, 악센트 구분, 언어별 정렬 순서 등을 포함, 데이터베이스/테이블/컬럼별 설정
- 디폴트는 utf8mb4_0900_ai_ci => utf8mb4 : 각문자 최대 4바이트 utf8 지원, 0900 : 정렬 방식 버전, ai : 악센트 비구분, ci : 대소문자 비구분 
- 0~9 < A-Z < a-z 을 비교하는 것을 사용 할 것이므로 utf8mb4_bin 을 사용하여 대소문자 구분도 추가해준다.

#### 댓글 테이블 설계
- 기존 parent_comment_id는 제거 , 댓글 경로를 나타내기 위한 path 컬럼 추가
- 설계 구조 상 무한이지만 일단은 5 depth 로제한 varchar(25)
```sql
create table comment_v2 (
    comment_id bigint not null primary key,
    content varchar(3000) not null,
    article_id bigint not null,
    writer_id bigint not null,
    path varchar(25) character set utf8mb4 collate utf8mb4_bin not null,
    deleted bool not null,
    created_at datetime not null
);
```
- 인덱스 설정
  - path에 인덱스를 생성하여 정렬 데이터를 관리한다. => 페이징에 사용될 수 있다.
  - path는 독립적인 경로를 가지므로, unique index로 생성한다. => 애플리케이션에서의 동시성 문제를 막아줄 수 있을 것이다.
```sql
create unique index idx_article_id_path on comment_v2(
    article_id asc, path asc
);
```

#### path 생성 구현 방법
```text
00a0z -> 00a0z 00000, 00a0z 00001, 00a0z 00002 -> 00a0z 00002 00000
이렇게 있을 때 00a0z 의 하위에 신규 댓글 작성이 되면 00a0z의 하위 댓글 중 가장 큰 path(childrenTopPath) 가 00a0z 00002 를 찾고 +1 한 문자열이 path 가 된다.

childrenTopPath(00a0z 00002)을 찾으려면
각 댓글의 모든 자손 댓글은 prefix가 상위 댓글의 path(parentPath, 00a0z)로 시작한다는 특성을 이용해볼 수 있다.
00a0z의 모든 자손 댓글은 prefix가 00a0z로 시작한다.

00a0z의 prefix(parentPath)를 가지는 모든 자손 댓글에서, 가장 큰 path(descendantsTopPath)를 찾아보자.
descendantsTopPath는 신규 댓글의 depth와 다를 수 있지만, childrenTopPath를 포함한다.
descendantsTopPath에서 (신규 댓글의 depth * 5)까지만 남기고 잘라내는 것이다.
00a0z 00002 00000에서 (신규 댓글의 depth 2) * 5 = 10개의 문자만 남기고 잘라낸다.
그러면 childrenTopPath = 00a0z 00002를 찾을 수 있다.

descendantsTopPath만 빠르게 잘 찾아내면, childrenTopPath는 애플리케이션 로직으로 어렵지 않게 구할 수 있을 것이다.
```
- childrenTopPath 구하는 쿼리
```sql
select path from comment_v2
where article_id = {article_id}
    and path > {parentPath} // parent 본인은 미포함 검색 조건
    and path like {parentPath}% // parentPath를 prefix로 하는 모든 자손 검색 조건
order by path desc limit 1; // 조회 결과에서 가장 큰 path
```
```text
article_id asc, path asc로 인덱스를 생성했으므로 where 범위 필터링은 정상적으로 처리될 수 있을 듯하다.
그런데 path는 오름차순으로 생성되어 있다. 
가장 큰 path 찾기 위한 정렬 조건이 path desc limit 1 때문에, 조회 시점에 내림차순 정렬을 하는 건 아닐지 염려해볼 수 있다.

explain 을 확인해보면 idx_article_id_path 사용되었다. 
Using where; Backward index scan; Using index
Extras=Using index를 통해 커버링 인덱스로 동작했음을 알 수 있다.

Backward index scan 란 index 를 역순 스캔 하는 것이다. • 인덱스 트리 leaf node 간에 연결된 양방향 포인터를 활용
결국 descendantsTopPath는 인덱스 덕분에 빠르게 찾아질 수 있다는 것이다.

신규 댓글을 만들때 숫자가 아니라 문자열 기반으로 덧셈 연산을 수행해야한다. 0-9 < A-Z < a-z
- 문자열 덧셈으로 구하기 : 0부터 z까지의 대소 관계를 정의하고, 오른쪽 문자부터 다음 문자로 바꿔준다.
    - carry(올림수)가 없으면(=0~y가 1~z로 바뀌면), 처리 종료
    - carry(올림수)가 있으면(=z가 0으로 바뀌면), 다음 문자도 처리
    - 이미 zzzzz 일땐 overflow

- 숫자 덧셈으로 구하기 : 62진수 문자열을 10진수 숫자로 바꿔서 +1 한 다음에, 숫자를 대응하는 문자열로 다시 바꿔준다


**** 예외 케이스
-  00a0z의 하위 댓글이 없어서 최초 생성일때 
    => descendantsTopPath가 없으므로, 00a0z 00000가 신규 댓글의 path가 된다. => parentPath + 00000
- 해당 경로에서 childrenTopPath = zzzzz까지 댓글이 생성되어 있을때
    => 값을 표현할 수 있는 범위(62^5개)를 벗어났기 때문에 더 이상 생성될 수 없다. , depth 별 문자개수를 늘려서 해결 가능
```

#### 페이지 번호 , 무한스크롤 일때 조회 쿼리
```sql
# 목록 조회
select * from (
  select comment_id
  from comment_v2
  where article_id = {article_id}
  order by path asc
    limit {limit} offset {offset} 
) t left join comment_v2 on t.comment_id = comment_v2.comment_id;  

# 카운트 조회
select count(*) from (
    select comment_id from comment_v2 where article_id = {article_id} limit {limit}
) t;

# 무한스크롤 1번 페이지 일때
select * from comment_v2
  where article_id = {article_id}
  order by path asc limit {limit};
  
# 무한스크롤 2페이지 이상 , 기준점 last_path
select * from comment_v2
  where article_id = {article_id} and path > {last_path}
  order by path asc
    limit {limit};
```
