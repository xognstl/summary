### 4. 댓글 최대 2 depth - 목록 API 설계
___
```text
댓글은 오래된 댓글이 먼저 노출된다.
계층형 구조라 생성 시간(comment_id)으로는 정렬 순서가 명확하지 않다. 
계층 관계에서는 더 늦게 작성된 댓글이 먼저 노출될 수 있다. 
상위 댓글은 하위 댓글보다 반드시 먼저 생성된다. 
같은 상위 댓글을 공유하는 하위 댓글들은, 생성 시간 순으로 정렬 순서가 명확하다.
즉, 상위 댓글은 항상 하위 댓글보다 먼저 생성되고,  
하위 댓글은 상위 댓글 별 순서대로 생성된다.상위 댓글에 의해 순서가 명확하게 정해질 수 있는 것이다.

==> (parent_comment_id 오름차순, comment_id 오름차순)의 정렬 구조를 가지고 있다.

• article_id asc, parent_comment_id asc, comment_id asc;
    • (게시글 ID 오름차순, 상위 댓글 ID 오름차순, 댓글 ID 오름차순)
    • ID는 오름차순으로 생성된다. (snowflake)
    • shard key = article_id 이기 때문에, 단일 샤드에서 게시글별 댓글 목록을 조회할 수 있다.
    
```
- 인덱스 생성
```sql
create index idx_article_id_parent_comment_id_comment_id on comment (
    article_id asc, parent_comment_id asc, comment_id asc
);
```
- N번 페이지에서 M개의 댓글 조회 : 서브쿼리로 covering index 만 뽑아내서 사용
```sql
  select * from (
      select comment_id from comment
      where article_id = {article_id}
      order by parent_comment_id asc, comment_id asc
      limit {limit} offset {offset}
  ) t left join comment on t.comment_id = comment.comment_id;
```
- 댓글 개수 조회
```sql
select count(*) from (
    select comment_id from comment where article_id = {article_id} limit {limit}
) t;
```
- 무한 스크롤
- 1번 페이지
```sql
select * from comment
    where article_id = {article_id}
    order by parent_comment_id asc, comment_id asc
    limit {limit};
```
- 2번 페이지 이상 => 기준점 = last_parent_comment_id, last_comment_id
```sql
select * from comment
    where article_id = {article_id} and (
        parent_comment_id > {last_parent_comment_id} or
        (parent_comment_id = {last_parent_comment_id} and comment_id > {last_comment_id})
    )
    order by parent_comment_id asc, comment_id asc
    limit {limit};
```
