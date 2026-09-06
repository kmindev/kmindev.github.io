---
title: "[JPA] 주인이 아닌 OneToOne 관계에서 Lazy로딩이 동작하지 않는 이슈"
date: 2026-09-06 00:00:00 +0900
categories: [Language/Programming]
tags: [JPA, "@OneToOne", Lazy로딩, 트러블슈팅]
---

## 1. 문제 상황

게시글(board)과 게시글통계정보(boardStatistic) 1:1 양방향 연관관계로 설계된 상황이다.

연관관계의 주인은 `boardStatistic`이며, `board`는 `mappedBy`를 통해 참조만 하는 구조이다.

로딩 전략은 모두 LAZY로 설정되어 있다.

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Board extends BaseEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne(mappedBy = "board", fetch = FetchType.LAZY, cascade = CascadeType.ALL)
    private BoardStatistic boardStatistic;

  	// ..
}

@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class BoardStatistic {
   @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
   private Long id;

   @OneToOne(fetch = FetchType.LAZY)
   @JoinColumn(name = "board_id", foreignKey = @ForeignKey(ConstraintMode.NO_CONSTRAINT))
   private Board board;

   // ..
}
```

### 예상했던 동작
`Board`를 페이징 조회할 때, `BoardStatistic`은 LAZY로 설정되어 있기에, 
실제 `board.getBoardStatistic().getXXX()`(id 제외)을 호출하기 전까지 `boardStatistic`을 조회하는 쿼리가 발생하지 않을 것으로 예상했다.

```java
Page<Board> findAll(Pageable pageable);
```

### 실제 동작(N+1 발생)
쿼리 로그를 확인해보니, Lazy 로딩은 무시되고, Board를 한 건마다 BoardStatistic을 조회하는 쿼리가 발생하는 것이었다.

board가 20개라면, boardStatistic 조회하는 쿼리가 20개 발생할 수 있는 상황이다.

```sql
-- 1. Board 페이징 조회
select * from board b1_0 order by b1_0.created_at desc limit ?

-- 2. 각 Board마다 BoardStatistic을 개별 조회 (N+1 발생!)
select * from board_statistic bs1_0 where bs1_0.board_id = ?
select * from board_statistic bs1_0 where bs1_0.board_id = ?
select * from board_statistic bs1_0 where bs1_0.board_id = ?
```

## 2. 원인 파악

내가 평소 알고 있던, JPA 지식과 실제 동작에서 차이가 나는 지점은 2가지이다.

1. Lazy로 설정했음에도, 지연로딩을 하지 않는 것일까?
2. `default_batch_fetch_size`를 설정했음에도, 왜 IN 쿼리가 아닌 단건 조회가 나가는 것일까?

### 1. 왜 Lazy 설정을 무시하고 즉시로딩을 하는 것일까?

이유는 JPA가 사용하는 프록시 객체의 한계 때문이다.

연관관계의 주인인 `boardStatistic`은 `board_id`(FK)를 직접 가지기에 `Board`의 존재 여부를 즉시 알 수 있다.

하지만, `board` 입장에서는 `boardStatistic` 존재 여부를 알 수 없다.

그래서 JPA 입장에서는 `board`를 조회할 때 `boardStatistic` 필드에 프록시를 넣어야 할지, null을 넣어야 할지 결정해야 하는데, board 테이블만 봐서 알 방법이 없다.

결국 JPA는 존재 여부를 확인하기 위해 `boardStatistic`을 직접 조회하게 되고, 그로 인해 LAZY 설정이 무시된 것이다.

### 2. 그럼 왜 IN 쿼리는 안 나가고, 단건 조회로 N번만큼 조회하는걸까?

`default_batch_fetch_size`는 이미 생성된 프록시들을 한꺼번에 초기화할 때 사용되는 설정이다.

현재 상황은 프록시 자체를 생성할 수 있는지(null 여부)를 확인하려고 쿼리를 날리는 단계로, 해당 설정과 무관한 것이었다.

## 3. 해결 방안
내가 생각한 방안은 5가지이다.

### 1. `Fetch Join` 또는 `@EntityGraph` 사용
- `board`를 조회할 때 `boardStatistic`을 함께 조회하도록 설정하는 방법이다. 
- 1번의 쿼리로 데이터를 모두 조회할 수 있다.
- 하지만, 구조적인 문제를 고치지 않으면, board를 조회하는 모든 쿼리 메서드에서 해당 설정을 추가해줘야 하므로 매우 번거롭다.
```java
@EntityGraph(attributePaths = {"boardStatistic"})
Page<Board> findAll(Pageable pageable);
```

```sql
select
    b1_0.id,
    -- ... Board의 다른 컬럼들,
    bs1_0.id,
    -- ... BoardStatistic의 컬럼들
from board b1_0
left join board_statistic bs1_0 on b1_0.id = bs1_0.board_id
order by b1_0.created_at desc limit ?
```

### 2. 연관관계 주인(FK) 위치 수정
비즈니스 로직상 `board`에서 `boardStatistic`을 조회하는 빈도가 훨씬 높다면, `board` 테이블이 `board_statistic_id`(FK)를 갖도록 DB 설계를 변경하는 방법이다.

이렇게 하면, board 테이블에 FK가 존재하기에, JPA는 DB 조회 시점에 해당 컬럼이 null인지 아닌지 바로 알 수 있으므로 불필요한 쿼리가 발생하지 않는다.

하지만 게시글통계가 게시글의 종속적인 구조이기에 DB 관점에서 `boardStatistic`이 `board_id`(FK)를 갖는 것이 더 자연스럽다.

DBA가 별로 좋아하지 않을 것 같다..

### 3. 양방향 관계 고민

양방향 매핑을 포기하는 것이다.

양방향 매핑을 사용하면 객체 그래프 탐색이 가능해 로직이 깔끔하고 관계가 명확해보이는 장점은 있지만, 이번 케이스처럼 쿼리 예측이 어렵다는 치명적인 단점이 있다.

단점이 매우 치명적이기에 양방향 매핑은 매우 신중하게 접근해야 한다.

지금 같은 게시글 - 게시글통계 구조에서 굳이 양방향 매핑을 유지해 예상치 못한 쿼리가 발생하여 성능이 저하되거나, 매번 fetch join 등을 고민하기 보다는, 단방향으로 끊어내고 필요한 시점에만 따로 조회하는 것이 훨씬 안전하고 깔끔한 설계라고 생각한다.

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Board extends BaseEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    // 연관관계 제거!
    // @OneToOne(mappedBy = "board", fetch = FetchType.LAZY, cascade = CascadeType.ALL)
    // private BoardStatistic boardStatistic;

}
```

### 4. `@MapsId`로 PK 공유

이것도 DB 설계를 바꾸는 방법인데, `@MapsId`로 `BoardStatistic`의 PK를 `Board`의 PK와 공유시키는 것이다. 

즉, `Board.id`를 `BoardStatistic`의 PK이자 FK로 그대로 쓰는 것이다.

이렇게 하면 `BoardStatistic.id`가 곧 `Board.id`이기 때문에, 존재 여부를 미리 확인할 필요가 없어지니 일반 프록시처럼 다룰 수 있게 되고, 
지연 로딩은 물론 `default_batch_fetch_size`를 통한 배치 조회(IN 쿼리)까지 정상 동작한다.

다만 이 방법도 DB 설계를 변경해야 하는 비용이 따르고, 이미 운영 중인 테이블이라면 PK/FK를 손봐야 하는 마이그레이션 작업이 필요하다.

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Board extends BaseEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne(mappedBy = "board", fetch = FetchType.LAZY, cascade = CascadeType.ALL)
    private BoardStatistic boardStatistic;

  	// ..
}

@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class BoardStatistic {
   @Id
   private Long id;

   @MapsId
   @OneToOne(fetch = FetchType.LAZY)
   @JoinColumn(name = "board_id", foreignKey = @ForeignKey(ConstraintMode.NO_CONSTRAINT))
   private Board board;

   // ..
}
```

```sql
select
    b1_0.id,
    b1_0.title 
from board b1_0

select
    bs1_0.board_id,
    bs1_0.view_count
from
    board_statistic bs1_0
where
    bs1_0.board_id in (?, ... )
```

결과를 보면 board 조회 1건, board_statistic 배치 조회(IN 쿼리) 1회만 발생하는 것을 확인할 수 있다.

### 5. `OneToMany` - `ManyToOne` 관계로 변경

`@OneToOne` 대신 `@OneToMany` - `@ManyToOne` 관계로 바꾸고, 연관관계의 주인(`boardStatistic`)이 FK를 `unique`로 명시하는 방법이다.

`board`가 `boardStatistic`을 단일 값이 아니라 컬렉션으로 가지도록 바꾸면, null인지 아닌지 미리 확인할 필요 없이 그냥 지연 로딩 컬렉션 프록시를 만들면 그만이다.

다만 실제로는 1:1인 관계를 코드 상에서 컬렉션으로 다뤄야 해서 모델링이 의도와 어긋난다는 점이 아쉽다.

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Board {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToMany(mappedBy = "board", fetch = FetchType.LAZY, cascade = CascadeType.ALL)
    private List<BoardStatistic> boardStatistics = new ArrayList<>();

    // 실제로는 1:1이므로 편의 메서드로 감싸서 사용
    public BoardStatistic getBoardStatistic() {
        return boardStatistics.isEmpty() ? null : boardStatistics.get(0);
    }
}

@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class BoardStatistic {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // DB 유니크 제약으로 사실상 1:1을 강제
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "board_id", unique = true)
    private Board board;
}
```

컬렉션의 지연 로딩/배치 페치는 DB 설계를 바꾸지 않고도 `default_batch_fetch_size`만으로 IN 쿼리 배치가 그대로 동작한다.

```sql
select b1_0.id, b1_0.title from board b1_0

select
    bs1_0.board_id,
    bs1_0.id,
    bs1_0.view_count
from board_statistic bs1_0
where bs1_0.board_id in (?, ... )
```

해결방법은 5가지 정도로 보이며, 정답은 없으니 프로젝트 상황에 따라 유연하게 적용하면 될 것 같다.