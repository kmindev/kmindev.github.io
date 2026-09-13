---
title: "[DB] Keyset 페이징 vs Offset 페이징 비교"
date: 2026-09-13 00:00:00 +0900
categories: [Database]
tags: [Pagination, Keyset, Offset, DB]
---

이번 포스팅에서는 페이징 처리에서의 offset / keyset 두 가지 방식에 대해서 다뤄보겠다. 

## 1. Offset 페이징

Offset 방식은 조건에 맞는 데이터를 정렬된 순서대로 읽으면서, 앞에서부터 `OFFSET`만큼 건너뛰고 그 다음 `LIMIT` 개수만큼 데이터를 가져와 전달하는 방식이다.

```sql
SELECT *
FROM table
WHERE {condition}
ORDER BY {column}
LIMIT {size} OFFSET {건너뛸 개수}

--- 예시: id 기준 정렬, size = 10
SELECT * FROM board order by id limit 10 offset 0; -- 1-page: 1 ~ 10
SELECT * FROM board order by id limit 10 offset 10; -- 2-page: 11 ~ 20
SELECT * FROM board order by id limit 10 offset 20; -- 3-page: 21 ~ 30
```


### Offset 방식의 한계

**1. offset 깊어질수록 선형적으로 처리시간이 길어짐.**

Offset 방식은 매 요청마다 조건에 맞는 데이터를 처음부터 다시 순서대로 세어가며 `OFFSET + LIMIT`개를 읽은 뒤, 앞의 `OFFSET`개는 버리고 나머지 `LIMIT`개만 반환한다. 

예를 들어 offset=20,000이면, limit=20 이면, 앞에서부터 20,000개를 모두 읽고 마지막에 읽은 20개만 제공한다. 

그래서 Offset이 커질수록 읽어야 하는 데이터의 양도 함께 늘어나, 뒤쪽 페이지로 갈수록 조회 성능이 점점 느려진다.


**2. 데이터 누락/중복 이슈**

페이지를 순서대로 넘기는 도중 데이터가 삽입/삭제되면, 다음 페이지 조회 시 기준이 되는 순번이 밀리면서 데이터 누락이나 중복이 발생할 수 있다. 

삽입/삭제뿐만 아니라, 조회 조건에 사용된 컬럼 값이 변경되어 해당 row가 더 이상 조건을 만족하지 않게 되는 경우에도 삭제된 것과 동일한 문제가 발생한다.

예를 들어 `id DESC` 기준으로 정렬된 데이터가 `10, 9, 8, 7, 6, 5, 4, 3, 2, 1` 이고, 페이지당 5개씩 조회한다고 가정해보자.

1. 1페이지 조회 (`OFFSET 0 LIMIT 5`) → `10, 9, 8, 7, 6` 반환
2. 이 시점에 `id=9` 데이터가 삭제 → 정렬 결과는 `10, 8, 7, 6, 5, 4, 3, 2, 1`
3. 2페이지 조회 (`OFFSET 5 LIMIT 5`) → 앞에서 5개(`10, 8, 7, 6, 5`)를 건너뛰고 `4, 3, 2, 1` 반환

원래대로라면 2페이지에는 `5, 4, 3, 2, 1`이 나와야 하지만, `id=5`가 한 칸씩 당겨지면서 건너뛴 5개 안에 포함되어버려 사용자는 `id=5`를 보지 못하게 된다. 

반대로 앞쪽에 새 데이터가 삽입(또는 조건에 새로 포함)되는 경우에는, 뒤로 밀린 데이터가 다음 페이지에서 다시 등장하는 중복이 발생한다.


## 2. Keyset 페이징

keyset 방식은 마지막에 읽은 key값을 전달받아, 그 값 이후의 데이터를 limit 개수만큼 읽어온다.

`key > lastKey` 조건으로 다음 데이터를 조회하는 방식이다. 


```sql
SELECT *
FROM table
WHERE {condition} AND id > {last_seen_id}
ORDER BY id
LIMIT {size}

--- 예시: id 기준 정렬, size = 10
SELECT * FROM board WHERE id > 0  ORDER BY id LIMIT 10; -- 1page: 1 ~ 10 (처음엔 조건 없음)
SELECT * FROM board WHERE id > 10 ORDER BY id LIMIT 10; -- 2page: 11 ~ 20
SELECT * FROM board WHERE id > 20 ORDER BY id LIMIT 10; -- 3page: 21 ~ 30
```

offset 방식과 달리 페이지가 뒤로 갈수록 느려지지 않는다.

Offset처럼 앞부분을 세어가며 지나가는 게 아니라, 인덱스에서 `last_seen_id`가 위치한 지점을 바로 탐색해서 들어간다. 

그래서 몇 페이지째든 조회 비용이 거의 일정하다.

또한, 조회 도중 데이터가 삽입/삭제되어도 Offset처럼 기준점이 밀려서 생기는 누락·중복 문제가 발생하지 않는다.

### Keyset 방식의 한계

**1. 임의의 페이지로 건너뛸 수 없다**

이전 페이지의 마지막 key값을 알아야 다음 페이지를 조회할 수 있기 때문에, 1페이지 → 2페이지 → 3페이지처럼 순차적으로만 이동할 수 있다.

**2. 정렬 기준이 여러 개면 커서 조건이 복잡해진다**

정렬 기준이 `id` 하나면 `WHERE id > last_id`로 충분하지만, `ORDER BY created_at, id`처럼 정렬 기준이 여러 개면 커서 조건도 함께 늘어난다. 

단순히 `created_at > last_created_at`만 걸면, `created_at`이 같은 row들 사이에서 데이터가 누락될 수 있다.

예를 들어 `created_at`이 `10:00:01`로 동일한 `id=1, 2, 3`이 있고, 1페이지(`LIMIT 2`)에서 `id=1, 2`를 반환했다고 하자.

조건을 `created_at > '10:00:01'`로만 걸면 `id=3`도 `created_at`이 같아서 조건에 걸려 데이터가 누락된다. 

그래서 "created_at이 더 크거나, 같다면 id로 한 번 더 비교"하는 조건을 걸어야 한다.

```sql
WHERE created_at > '10:00:01'
   OR (created_at = '10:00:01' AND id > 2)
```

이렇게 하면 `id=3`부터 정확히 이어진다.

## 3. offset vs keyset 비교 테스트

MySQL에 260만 건의 데이터를 넣고, 페이지 크기(LIMIT)는 20으로 고정한 채 조회 위치(offset)만 늘려가며 각각 3회씩 조회 시간을 측정했다.

```sql
-- offset
SELECT 
    id, title 
FROM board 
ORDER BY id 
    LIMIT 20 OFFSET {건너뛸 개수};

-- keyset
SELECT 
    id, title 
FROM board 
WHERE id >= {last_seen_id} 
ORDER BY id 
LIMIT 20;
```

| 조회 위치 | Offset 방식 | Keyset 방식 |
| --- | --- | --- |
| 0 | 0.068초 | 0.067초 |
| 10,000 | 0.072초 | 0.068초 |
| 100,000 | 0.064초 | 0.054초 |
| 500,000 | 0.151초 | 0.072초 |
| 1,000,000 | 0.251초 | 0.067초 |
| 2,000,000 | 0.468초 | 0.065초 |

Offset 방식은 조회 위치가 커질수록 소요 시간이 선형적으로 늘어난다 (0.07초 → 0.47초, 약 7배 증가). 

반면 Keyset 방식은 조회 위치와 무관하게 0.06~0.07초 수준으로 거의 일정하다.

## 4. 결론

데이터 적거나, 앞쪽 위주의 페이지 조회만 일어난다면 offset 방식을 사용해도 무방하다.

무한스크롤 같은 페이지가 깊어질 수 있다면, keyset 방식을 고려해야 한다.

또한, offset 방식은 데이터 누락/중복 이슈가 있기 때문에, 배치 작업 같은 데이터를 오차없이 읽어야 한다면, keyset 방식을 사용해야 한다.