# Index Basic

### Contents

- [Query 1: 전체 회원 조회](#query-1-전체-회원-조회)
- [Query 2-1: 전체 회원 조회 - Index Column](#query-2-1-전체-회원-조회---index-column)
- [Query 2-2: 전체 회원 조회 - Index Column + NO_INDEX_FFS Hint](#query-2-2-전체-회원-조회---index-column--no_index_ffs-hint)
- [Query 3: 전체 회원 조회 - Index Column + ORDER BY DESC](#query-3-전체-회원-조회---index-column--order-by-desc)
- [Query 4: 전체 회원 조회 - Index Column & Non-Index Column](#query-4-전체-회원-조회---index-column--non-index-column)
- [Query 5-1: 주문 상품 조회 - USE_CONCAT Hint vs. UNION ALL](#query-5-1-주문-상품-조회---use_concat-hint-vs-union-all)
    - [Conclusion](#query_5_1_conclusion)
- [Query 5-2: 주문 상품 조회 - USE_CONCAT Hint vs. UNION ALL vs. INDEX_COMBINE Hint](#query-5-2-주문-상품-조회---use_concat-hint-vs-union-all-vs-index_combine-hint)
    - [Conclusion](#query_5_2_conclusion)
- [Query 6: 주문 조회 - IN-List Iterator](#query-6-주문-조회---in-list-iterator)

<br/>
<br/>

## Query 1: 전체 회원 조회

```sql
SELECT *
FROM MEMBER;
```

<br/>

### Execution Plan

```
Plan hash value: 3441279308
 
----------------------------------------------------------------------------
| Id  | Operation         | Name   | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |        | 99733 |    10M|   204   (0)| 00:00:01 |
|   1 |  TABLE ACCESS FULL| MEMBER | 99733 |    10M|   204   (0)| 00:00:01 |
----------------------------------------------------------------------------
```

`TABLE ACCESS FULL` : Table full scan 방식으로 결과 집합을 가져왔다.

**즉, Sequential access & Multiblock I/O가 발생하였다.**

<br/>

[참고] `Plan hash value` : Numeric representation of the current SQL plan.  
→ 각 실행 계획을 고유하게 식별하는 값이다.

<br/>
<br/>

## Query 2-1: 전체 회원 조회 - Index Column

```sql
SELECT MEMBER_ID
FROM MEMBER;
```

<br/>

### Execution Plan

```
Plan hash value: 3279811250
 
--------------------------------------------------------------------------------------
| Id  | Operation            | Name          | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT     |               | 99733 |  1266K|    68   (0)| 00:00:01 |
|   1 |  INDEX FAST FULL SCAN| MEMBER_ID_IDX | 99733 |  1266K|    68   (0)| 00:00:01 |
--------------------------------------------------------------------------------------
```

`INDEX FAST FULL SCAN` → Index scan을 통해서 결과 집합을 가져오는 것으로 선택되었다.

<br/>

Oracle에서는 어떠한 table의 PK에 대해서, 자동으로 Unique index를 생성해준다.

> [[참고] Oracle 공식 문서](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/managing-indexes.html#GUID-BF04684D-A857-4046-8749-0F57D3B19113)  
> Oracle Database enforces a `UNIQUE` key or `PRIMARY KEY` integrity constraint on a table  
> by creating a unique index on the unique key or primary key.  
> This index is automatically created by the database when the constraint is enabled.

<br/>

따라서 MEMBER_ID(= MEMBER table’s PK)에 대한 Unique index가 존재하고

<p align="center"><img width="339" alt="unique_idx" src="https://github.com/user-attachments/assets/27335f14-60e2-49b9-8987-a035e1826a31">

현재 query에서 조회를 하는 것은 MEMBER_ID 밖에 없으므로,  
Optimizer는 해당 index를 scan 하여 결과 집합을 가져오는 방향을 선택한 것이다.

(Index의 이름은 `ALTER INDEX current_name TO name_to_change` query를 통해서 변경할 수 있음)

<br/>

이때 Index fast full scan은 **Index segment 전체를 Multiblock I/O 방식으로 scan** 하는 방식이다.

즉, 물리적으로 disk에 저장된 순서대로 Index leaf block들을 읽어들이는 것이다.

<br/>
<br/>

## Query 2-2: 전체 회원 조회 - Index Column + NO_INDEX_FFS Hint

```sql
SELECT /*+ NO_INDEX_FFS(MEMBER MEMBER_ID_IDX) */
    MEMBER_ID
FROM MEMBER;
```

`NO_INDEX_FFS` hint는 **Optimizer가 Index fast full scan 방식을 선택하지 않도록 유도한다.**

↔︎ `INDEX_FFS` hint

<br/>

### Execution Plan

```
Plan hash value: 3441279308
 
----------------------------------------------------------------------------
| Id  | Operation         | Name   | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |        | 99733 |  1266K|   204   (0)| 00:00:01 |
|   1 |  TABLE ACCESS FULL| MEMBER | 99733 |  1266K|   204   (0)| 00:00:01 |
----------------------------------------------------------------------------
```

Table full scan 방식이 선택되었다.

Query 2-1에서의 실행 계획과 비교해보면, 읽어들인 row의 개수는 동일하지만    
**table block I/O가 증가하였기 때문에 예상 cost도 같이 증가한 것을 볼 수 있다. (68 → 204)**

<br/>
<br/>

## Query 3: 전체 회원 조회 - Index Column + ORDER BY DESC

```sql
SELECT MEMBER_ID
FROM MEMBER
ORDER BY MEMBER_ID DESC;
```

`DESC` : 내림차순 정렬

<br/>

### Execution Plan

```
Plan hash value: 927876221
 
--------------------------------------------------------------------------------------------
| Id  | Operation                  | Name          | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT           |               | 99733 |  1266K|   245   (0)| 00:00:01 |
|   1 |  INDEX FULL SCAN DESCENDING| MEMBER_ID_IDX | 99733 |  1266K|   245   (0)| 00:00:01 |
--------------------------------------------------------------------------------------------
```

`INDEX FULL SCAN DESCENDING` → Index를 내림차순으로 scan 한다.

<br/>
<br/>

## Query 4: 전체 회원 조회 - Index Column & Non-Index Column

```sql
SELECT MEMBER_ID,
       NAME
FROM MEMBER;
```

<br/>

### Execution Plan

```
Plan hash value: 3441279308
 
----------------------------------------------------------------------------
| Id  | Operation         | Name   | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |        | 99733 |  3895K|   204   (0)| 00:00:01 |
|   1 |  TABLE ACCESS FULL| MEMBER | 99733 |  3895K|   204   (0)| 00:00:01 |
----------------------------------------------------------------------------
```

Query 2-1에서는 MEMBER_ID column 단 하나만을 조회하는 것이라서 MEMBER_ID_IDX를 타는 것이 더 효율적이었는데,  
**지금 query에서는 NAME까지 가져와야 하므로 Table full scan 방식이 선택되었다.**

<br/>

### CREATE INDEX

MEMBER_ID와 NAME을 column으로 가지는 index를 새로 생성해보자.

```sql
CREATE UNIQUE INDEX MEMBER_ID_NAME_IDX
    ON MEMBER (MEMBER_ID, NAME);
```

<br/>

#### Execution Plan

```
Plan hash value: 2860970536
 
-------------------------------------------------------------------------------------------
| Id  | Operation            | Name               | Rows  | Bytes | Cost (%CPU)| Time     |
-------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT     |                    | 99733 |  3895K|   111   (0)| 00:00:01 |
|   1 |  INDEX FAST FULL SCAN| MEMBER_ID_NAME_IDX | 99733 |  3895K|   111   (0)| 00:00:01 |
-------------------------------------------------------------------------------------------
```

방금 생성한 놈을 fast full scan 하는 방향을 선택하였다. (Cost : 204 → 111)


<br/>
<br/>

## Query 5-1: 주문 상품 조회 - USE_CONCAT Hint vs. UNION ALL

```sql
SELECT ORDERED_ITEM_ID
FROM ORDERED_ITEM
WHERE (PRICE = 40
    OR QUANTITY = 2);
```

<br/>

### Execution Plan

```
Plan hash value: 1308414751
 
----------------------------------------------------------------------------------
| Id  | Operation         | Name         | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |              |   208K|  7954K|  1920   (1)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| ORDERED_ITEM |   208K|  7954K|  1920   (1)| 00:00:01 |
----------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
"   1 - filter(""PRICE""=40 OR ""QUANTITY""=2)"
```

Table full scan 방식으로 결과 집합을 가져왔다.

<br/>

### CREATE INDEX + USE_CONCAT Hint + INDEX Hint

PRICE와 QUANTITY 각각만을 column으로 가지는 두 index를 새로 생성한 후, 그 둘을 타도록 강제해보자.

```sql
CREATE INDEX PRICE_IDX
    ON ORDERED_ITEM (PRICE);

CREATE INDEX QUANTITY_IDX
    ON ORDERED_ITEM (QUANTITY);
```

```sql
SELECT /*+
         USE_CONCAT
         INDEX(ORDERED_ITEM PRICE_IDX)
                       QUANTITY_IDX)
         */
    ORDERED_ITEM_ID
FROM ORDERED_ITEM
WHERE (PRICE = 40
    OR QUANTITY = 2);
```

- `INDEX ( table [index [index]...] )` hint : 해당 index를 사용하도록 유도한다.


- `USE_CONCAT` hint : `WHERE` 절에 존재하는 `OR` 조건을 `UNION ALL` 방식을 사용하는 것으로 변환하여 처리한다.
  > [[참고] USE_CONCAT](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Comments.html#GUID-5A12BDC8-4CD1-448F-80BF-9F02653A3F94)  
  > The USE_CONCAT hint forces combined OR conditions in the WHERE clause of a query  
  > to be transformed into a compound query using the UNION ALL set operator.

  즉, 위 query를 아래 query로 변환하는 것이다. _(라고 이해했었다 ..)_

  ```sql
  SELECT /*+ INDEX(ORDERED_ITEM PRICE_IDX) */
      ORDERED_ITEM_ID
  FROM ORDERED_ITEM
  WHERE PRICE = 40
  
  UNION ALL
  
  SELECT /*+ INDEX(ORDERED_ITEM QUANTITY_IDX) */
      ORDERED_ITEM_ID
  FROM ORDERED_ITEM
  WHERE QUANTITY = 2;
  ```
  **IN-List Iterator 방식**(IN 조건 만큼 Index range scan을 반복)을 유도하게 된다.

<br/>

#### Execution Plan

```
Plan hash value: 2684606241
 
-----------------------------------------------------------------------------------------------------
| Id  | Operation                            | Name         | Rows  | Bytes | Cost (%CPU)| Time     |
-----------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                     |              |   208K|  7954K|  2581   (0)| 00:00:01 |
|   1 |  CONCATENATION                       |              |       |       |            |          |
|   2 |   TABLE ACCESS BY INDEX ROWID BATCHED| ORDERED_ITEM |   188K|  7183K|   672   (0)| 00:00:01 |
|*  3 |    INDEX RANGE SCAN                  | QUANTITY_IDX |  2303 |       |   392   (0)| 00:00:01 |
|*  4 |   TABLE ACCESS BY INDEX ROWID BATCHED| ORDERED_ITEM | 20249 |   771K|  1909   (0)| 00:00:01 |
|*  5 |    INDEX RANGE SCAN                  | PRICE_IDX    |  2303 |       |    45   (0)| 00:00:01 |
-----------------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
"   3 - access(""QUANTITY""=2)"
"   4 - filter(LNNVL(""QUANTITY""=2))"
"   5 - access(""PRICE""=40)"
```

- `INDEX RANGE SCAN` : Index를 **필요한 범위(range)만** scan 한다.


- `TABLE ACCESS BY INDEX ROWID BATCHED`
  > [[참고] Table Access by Rowid](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgsql/optimizer-access-paths.html#GUID-4180BA97-3E2C-41F9-B282-4FB3FF9532CB)  
  > The BATCHED access shown in Step 1 means that the database **retrieves a few rowids from the index,**  
  > and then **attempts to access rows in block order** to improve the clustering and reduce the number of times  
  > that the database must access a block.


- `LNNVL`
  > [[참고] LNNVL(condition)](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/LNNVL.html#GUID-FBCCE9B1-614E-45FA-8EE1-DFAA4F936867)  
  > **returns TRUE if the condition is FALSE or UNKNOWN** and FALSE if the condition is TRUE.

  `LNNVL`이 PRICE와 QUANTITY column에 `NULL` 값이 존재할 수 있어서 발생하는 것으로 예상하고  
  [NOT NULL constraint를 거는 것으로 변경했는데,](../ERD/README.md#DDL-with-NOT-NULL-constraints) `LNNVL`이 나타나는 것은 그대로였다.  
  → 이것이 나타나는 원인은 `NULL` 값이 아닌 것 같다.

<br/>

**각 Index에는 ORDERED_ITEM_ID column이 없기 때문에, Table access가 발생하는 것은 불가피하다.**

<br/>

### CREAT_INDEX with ORDERED_ITEM_ID

ORDERED_ITEM_ID column도 가지는 새로운 Index 2개를 생성하고, 그들을 타도록 유도하였다.

Index에서 조건에 부합하는 column들을 찾으면, ORDERED_ITEM_ID 정보를 바로 반환하면 되므로  
Table access가 발생하지 않을 것으로 예상했다.

```sql
CREATE INDEX PRICE_ORDERED_ITEM_ID_IDX
    ON ORDERED_ITEM (PRICE, ORDERED_ITEM_ID);

CREATE INDEX QUANTITY_ORDERED_ITEM_ID_IDX
    ON ORDERED_ITEM (QUANTITY, ORDERED_ITEM_ID);
```

```sql
SELECT /*+
         USE_CONCAT
         INDEX(ORDERED_ITEM PRICE_ORDERED_ITEM_ID_IDX)
                            QUANTITY_ORDERED_ITEM_ID_IDX)
         */
    ORDERED_ITEM_ID
FROM ORDERED_ITEM
WHERE (PRICE = 40
    OR QUANTITY = 2);
```

<br/>

#### Execution Plan

```
Plan hash value: 650344972
 
---------------------------------------------------------------------------------------------------------------------
| Id  | Operation                            | Name                         | Rows  | Bytes | Cost (%CPU)| Time     |
---------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                     |                              |   208K|  7954K|  2213   (0)| 00:00:01 |
|   1 |  CONCATENATION                       |                              |       |       |            |          |
|   2 |   TABLE ACCESS BY INDEX ROWID BATCHED| ORDERED_ITEM                 |   188K|  7183K|   304   (0)| 00:00:01 |
|*  3 |    INDEX RANGE SCAN                  | QUANTITY_ORDERED_ITEM_ID_IDX |  2303 |       |    24   (0)| 00:00:01 |
|*  4 |   TABLE ACCESS BY INDEX ROWID BATCHED| ORDERED_ITEM                 | 20249 |   771K|  1909   (0)| 00:00:01 |
|*  5 |    INDEX RANGE SCAN                  | PRICE_ORDERED_ITEM_ID_IDX    |  2303 |       |    24   (0)| 00:00:01 |
---------------------------------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
"   3 - access(""QUANTITY""=2)"
"   4 - filter(LNNVL(""QUANTITY""=2))"
"   5 - access(""PRICE""=40)"
```

**??? 새로 만든 index를 탔음에도 불구하고 Table access가 여전히 발생하고 있다.**

Table access 때문에, 맨 위 Table full scan을 하는 경우보다 cost가 높게 나온다.

<br/>

### UNION ALL

그렇다면 .. 직접 `UNION ALL` 방식을 사용했을 때에는 실행 계획이 어떻게 나올까?

```sql
SELECT /*+ INDEX(ORDERED_ITEM PRICE_ORDERED_ITEM_ID_IDX) */
    ORDERED_ITEM_ID
FROM ORDERED_ITEM
WHERE PRICE = 40

UNION ALL

SELECT /*+ INDEX(ORDERED_ITEM QUANTITY_ORDERED_ITEM_ID_IDX) */
    ORDERED_ITEM_ID
FROM ORDERED_ITEM
WHERE QUANTITY = 2;
```

<br/>

#### Execution Plan

```
Plan hash value: 3539198596
 
--------------------------------------------------------------------------------------------------
| Id  | Operation         | Name                         | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |                              |   208K|  5303K|   620   (1)| 00:00:01 |
|   1 |  UNION-ALL        |                              |       |       |            |          |
|*  2 |   INDEX RANGE SCAN| PRICE_ORDERED_ITEM_ID_IDX    | 20249 |   514K|    62   (0)| 00:00:01 |
|*  3 |   INDEX RANGE SCAN| QUANTITY_ORDERED_ITEM_ID_IDX |   188K|  4789K|   558   (1)| 00:00:01 |
--------------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
"   2 - access(""PRICE""=40)"
"   3 - access(""QUANTITY""=2)"
```

이전 실행 계획과는 다르게, 나의 의도대로 Index 만을 타서 결과 집합을 가져오는 것을 볼 수 있다. 이에 cost도 가장 낮게 나온다.

| Operation         | Cost |
|-------------------|------|
| Table full scan   | 1920 |
| `USE_CONCAT` hint | 2213 |
| `UNION ALL`       | 620  |

<br/>

<h3 id="query_5_1_conclusion">Conclusion</h3>

`USE_CONCAT` hint는 query를 단순히 `UNION ALL` 방식으로 변환하는 것이 아니다.

조회하는 column이 Index에 포함되어 있는 경우에도 Table access가 발생하는데,  
내부적으로 이렇게 동작하도록 되어 있는 것으로 추측된다. (관련 문서는 찾지 못했음..)

<br/>

\+) `USE_CONCAT` hint 인자

- `USE_CONCAT(@"MAIN" 1)` : IN-List를 사용할 수 있는 경우, `UNION ALL`로 분리하지 않도록 강제한다.
- `USE_CONCAT(@"MAIN" 8)` : IN-List를 사용할 수 있는 경우, `UNION ALL`로 분리하도록 강제한다.

[참고] https://scidb.tistory.com/entry/USECONCAT-힌트-제대로-알기

<br/>
<br/>

## Query 5-2: 주문 상품 조회 - USE_CONCAT Hint vs. UNION ALL vs. INDEX_COMBINE Hint

모든 column을 조회하는 경우, 즉 Table access가 불가피한 경우에는 각 cost들이 어떨까?

```sql
SELECT *
FROM ORDERED_ITEM
WHERE (PRICE = 40
    OR QUANTITY = 2);
```

<br/>

### Execution Plan

```
Plan hash value: 1308414751
 
----------------------------------------------------------------------------------
| Id  | Operation         | Name         | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |              |   208K|    12M|  1920   (1)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| ORDERED_ITEM |   208K|    12M|  1920   (1)| 00:00:01 |
----------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
"   1 - filter(""PRICE""=40 OR ""QUANTITY""=2)"
```

Query 5-1의 default 실행 계획과 동일하다.

<br/>

### USE_CONCAT Hint

```sql
SELECT /*+ USE_CONCAT */
    *
FROM ORDERED_ITEM
WHERE (PRICE = 40
    OR QUANTITY = 2);
```

<br/>

#### Execution Plan

```
Plan hash value: 650344972
 
---------------------------------------------------------------------------------------------------------------------
| Id  | Operation                            | Name                         | Rows  | Bytes | Cost (%CPU)| Time     |
---------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                     |                              |   208K|    12M|  2213   (0)| 00:00:01 |
|   1 |  CONCATENATION                       |                              |       |       |            |          |
|   2 |   TABLE ACCESS BY INDEX ROWID BATCHED| ORDERED_ITEM                 |   188K|    11M|   304   (0)| 00:00:01 |
|*  3 |    INDEX RANGE SCAN                  | QUANTITY_ORDERED_ITEM_ID_IDX |  2303 |       |    24   (0)| 00:00:01 |
|*  4 |   TABLE ACCESS BY INDEX ROWID BATCHED| ORDERED_ITEM                 | 20249 |  1285K|  1909   (0)| 00:00:01 |
|*  5 |    INDEX RANGE SCAN                  | PRICE_ORDERED_ITEM_ID_IDX    |  2303 |       |    24   (0)| 00:00:01 |
---------------------------------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
"   3 - access(""QUANTITY""=2)"
"   4 - filter(LNNVL(""QUANTITY""=2))"
"   5 - access(""PRICE""=40)"
```

PRICE_ORDERED_ITEM_ID_IDX와 QUANTITY_ORDERED_ITEM_ID_IDX를 탔다.

PRICE_IDX와 QUANTITY_IDX를 타는 것보다 cost가 더 낮아서 선택된 것일텐데 왜일까?

<br/>

PRICE_IDX와 QUANTITY_IDX를 타는 것을 강제하는 경우, 실행 계획은 다음과 같다.

```sql
SELECT /*+
         USE_CONCAT
         INDEX(ORDERED_ITEM PRICE_IDX
                            QUANTITY_IDX)
        */
    *
FROM ORDERED_ITEM
WHERE (PRICE = 40
    OR QUANTITY = 2);
```

```
Plan hash value: 2684606241
 
-----------------------------------------------------------------------------------------------------
| Id  | Operation                            | Name         | Rows  | Bytes | Cost (%CPU)| Time     |
-----------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                     |              |   208K|    12M|  2581   (0)| 00:00:01 |
|   1 |  CONCATENATION                       |              |       |       |            |          |
|   2 |   TABLE ACCESS BY INDEX ROWID BATCHED| ORDERED_ITEM |   188K|    11M|   672   (0)| 00:00:01 |
|*  3 |    INDEX RANGE SCAN                  | QUANTITY_IDX |  2303 |       |   392   (0)| 00:00:01 |
|*  4 |   TABLE ACCESS BY INDEX ROWID BATCHED| ORDERED_ITEM | 20249 |  1285K|  1909   (0)| 00:00:01 |
|*  5 |    INDEX RANGE SCAN                  | PRICE_IDX    |  2303 |       |    45   (0)| 00:00:01 |
-----------------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
"   3 - access(""QUANTITY""=2)"
"   4 - filter(LNNVL(""QUANTITY""=2))"
"   5 - access(""PRICE""=40)"
```

PRICE_ORDERED_ITEM_ID_IDX와 QUANTITY_ORDERED_ITEM_ID_IDX에는 ORDERED_ITEM_ID column이 포함되어 있고,  
**이를 기준으로 정렬되어 저장되어 있기 때문에 (PRICE - ORDERED_ITEM_ID 또는 QUANTITY - ORDERED_ITEM_ID 순)**  
Index scan → Table access가 더 효율적으로 진행되는 것으로 예상한다.

<br/>

### UNION ALL

```sql
SELECT /*+ INDEX(ORDERED_ITEM PRICE_ORDERED_ITEM_ID_IDX) */
    *
FROM ORDERED_ITEM
WHERE PRICE = 40

UNION ALL

SELECT /*+ INDEX(ORDERED_ITEM QUANTITY_ORDERED_ITEM_ID_IDX) */
    *
FROM ORDERED_ITEM
WHERE QUANTITY = 2;
```

<br/>

#### Execution Plan

```
Plan hash value: 1474185937
 
---------------------------------------------------------------------------------------------------------------------
| Id  | Operation                            | Name                         | Rows  | Bytes | Cost (%CPU)| Time     |
---------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                     |                              |   208K|    12M| 14387   (1)| 00:00:01 |
|   1 |  UNION-ALL                           |                              |       |       |            |          |
|   2 |   TABLE ACCESS BY INDEX ROWID BATCHED| ORDERED_ITEM                 | 20249 |  1285K|  6827   (1)| 00:00:01 |
|*  3 |    INDEX RANGE SCAN                  | PRICE_ORDERED_ITEM_ID_IDX    | 20249 |       |    62   (0)| 00:00:01 |
|   4 |   TABLE ACCESS BY INDEX ROWID BATCHED| ORDERED_ITEM                 |   188K|    11M|  7560   (1)| 00:00:01 |
|*  5 |    INDEX RANGE SCAN                  | QUANTITY_ORDERED_ITEM_ID_IDX |   188K|       |   558   (1)| 00:00:01 |
---------------------------------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
"   3 - access(""PRICE""=40)"
"   5 - access(""QUANTITY""=2)"
```

Table access가 발생하니, 오히려 성능이 굉장히 좋지 않다.

<br/>

### INDEX_COMBINE Hint

문서 뒤적뒤적하다가 발견한 hint 이다.

> [[참고] INDEX_COMBINE ( [ @ queryblock ] tablespec [ indexspec [ indexspec ]...] )](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Comments.html#GUID-ED589B14-7A6B-4CEB-9F6F-410E9DFF6BA2)  
> The INDEX_COMBINE hint can use any type of index: bitmap, b-tree, or domain.   
> If you specify indexspec, then the optimizer uses all the hinted indexes that are legal and valid to use, regardless
> of cost.

<br/>

```sql
SELECT /*+
         INDEX_COMBINE(ORDERED_ITEM PRICE_ORDERED_ITEM_ID_IDX
                                    QUANTITY_ORDERED_ITEM_ID_IDX)
         */
    *
FROM ORDERED_ITEM
WHERE (PRICE = 40
    OR QUANTITY = 2);
```

<br/>

#### Execution Plan

```
Plan hash value: 1387759801
 
--------------------------------------------------------------------------------------------------------------------
| Id  | Operation                           | Name                         | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                    |                              |   208K|    12M|  1650   (1)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| ORDERED_ITEM                 |   208K|    12M|  1650   (1)| 00:00:01 |
|   2 |   BITMAP CONVERSION TO ROWIDS       |                              |       |       |            |          |
|   3 |    BITMAP OR                        |                              |       |       |            |          |
|   4 |     BITMAP CONVERSION FROM ROWIDS   |                              |       |       |            |          |
|   5 |      SORT ORDER BY                  |                              |       |       |            |          |
|*  6 |       INDEX RANGE SCAN              | PRICE_ORDERED_ITEM_ID_IDX    |       |       |    25   (0)| 00:00:01 |
|   7 |     BITMAP CONVERSION FROM ROWIDS   |                              |       |       |            |          |
|   8 |      SORT ORDER BY                  |                              |       |       |            |          |
|*  9 |       INDEX RANGE SCAN              | QUANTITY_ORDERED_ITEM_ID_IDX |       |       |    25   (0)| 00:00:01 |
--------------------------------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
"   6 - access(""PRICE""=40)"
"       filter(""PRICE""=40)"
"   9 - access(""QUANTITY""=2)"
"       filter(""QUANTITY""=2)"
```

1. `INDEX RANGE SCAN` : 두 Index에 대하여 range scan을 진행한다.
2. `SORT ORDER BY` → `BITMAP CONVERSION FROM ROWIDS` : 각 scan 결과 집합(ROWIDs)을 정렬하고, BITMAP으로 변환한다.
3. **`BITMAP OR` : 두 결과 집합을 OR 조건으로 합친다.**
4. `BITMAP CONVERSION TO ROWIDS` → `TABLE ACCESS BY INDEX ROWID BATCHED` : 최종 결과 집합을 ROWIDs로 변환하고, Table access를 진행한다.

<br/>

비트 연산을 사용하기 때문에 가장 빠르다. (cost가 가장 낮음)

| Operation            | Cost  |
|----------------------|-------|
| Table full access    | 1920  |
| `USE_CONCAT` hint    | 2213  |
| `UNION ALL`          | 14387 |
| `INDEX_COMBINE` hint | 1650  |

<br/>

### +) Query 5-1에서 INDEX_COMBINE hint를 사용한다면 ?

```sql
SELECT /*+
         INDEX_COMBINE(ORDERED_ITEM PRICE_ORDERED_ITEM_ID_IDX
                                    QUANTITY_ORDERED_ITEM_ID_IDX)
         */
    ORDERED_ITEM_ID
FROM ORDERED_ITEM
WHERE (PRICE = 40
    OR QUANTITY = 2);
```

```
Plan hash value: 1387759801
 
--------------------------------------------------------------------------------------------------------------------
| Id  | Operation                           | Name                         | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                    |                              |   208K|  7954K|  1650   (1)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| ORDERED_ITEM                 |   208K|  7954K|  1650   (1)| 00:00:01 |
|   2 |   BITMAP CONVERSION TO ROWIDS       |                              |       |       |            |          |
|   3 |    BITMAP OR                        |                              |       |       |            |          |
|   4 |     BITMAP CONVERSION FROM ROWIDS   |                              |       |       |            |          |
|   5 |      SORT ORDER BY                  |                              |       |       |            |          |
|*  6 |       INDEX RANGE SCAN              | PRICE_ORDERED_ITEM_ID_IDX    |       |       |    25   (0)| 00:00:01 |
|   7 |     BITMAP CONVERSION FROM ROWIDS   |                              |       |       |            |          |
|   8 |      SORT ORDER BY                  |                              |       |       |            |          |
|*  9 |       INDEX RANGE SCAN              | QUANTITY_ORDERED_ITEM_ID_IDX |       |       |    25   (0)| 00:00:01 |
--------------------------------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
"   6 - access(""PRICE""=40)"
"       filter(""PRICE""=40)"
"   9 - access(""QUANTITY""=2)"
"       filter(""QUANTITY""=2)"
```

연산량이 많아져, 오히려 성능이 떨어진다.

| Operation            | Cost |
|----------------------|------|
| Table full scan      | 1920 |
| `USE_CONCAT` hint    | 2213 |
| `UNION ALL`          | 620  |
| `INDEX_COMBINE` hint | 1650 |

<br/>

<h3 id="query_5_2_conclusion">Conclusion</h3>

1. Index에 조회하는 column이 전부 포함되어 있는 경우, Index만을 타도록 강제하자.    
   (`USE_CONCAT` hint 보다는 `UNION ALL`로 풀어쓰기 → 경우에 따라 다를 수도 있을 것 같아서.. 직접 실행 계획을 출력해보고 선택하자)

2. Table access가 불가피한 경우, `INDEX_COMBINE` hint를 고려해보자.

<br/>
<br/>

## Query 6: 주문 조회 - IN-List Iterator

```sql
SELECT *
FROM ORDERS
WHERE STATUS IN ('DELIVERED',
                 'SHIPPED');
```

<br/>

### Execution Plan

```
Plan hash value: 1275100350
 
----------------------------------------------------------------------------
| Id  | Operation         | Name   | Rows  | Bytes | Cost (%CPU)| Time     |
----------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |        |  1297K|    76M|  2469   (1)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| ORDERS |  1297K|    76M|  2469   (1)| 00:00:01 |
----------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
"   1 - filter(""STATUS""='DELIVERED' OR ""STATUS""='SHIPPED')"
```

Table full access가 발생하면서, **filter 조건**으로 `""STATUS""='DELIVERED' OR ""STATUS""='SHIPPED'`가 들어갔다.

<br/>

### IN-List Iterator

STATUS column을 선두 column으로 가지는 Index를 생성해보자.

```sql
CREATE INDEX STATUS_IDX
    ON ORDERS (STATUS);
```

<br/>

#### Execution Plan

```
Plan hash value: 2893997418
 
---------------------------------------------------------------------------------------------------
| Id  | Operation                            | Name       | Rows  | Bytes | Cost (%CPU)| Time     |
---------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                     |            |  1297K|    76M|   133   (0)| 00:00:01 |
|   1 |  INLIST ITERATOR                     |            |       |       |            |          |
|   2 |   TABLE ACCESS BY INDEX ROWID BATCHED| ORDERS     |  1297K|    76M|   133   (0)| 00:00:01 |
|*  3 |    INDEX RANGE SCAN                  | STATUS_IDX |  7781 |       |    26   (0)| 00:00:01 |
---------------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
"   3 - access(""STATUS""='DELIVERED' OR ""STATUS""='SHIPPED')"
```

<br/>

`INLIST ITERATOR` : 아래 과정을 **IN-List 개수만큼 반복한다.**

1. Index를 range scan 한다. `IN` 조건(e.g., ""STATUS""='DELIVERED')이 **Index access 조건**이 되어, Index scan 시작점을 찾을 수 있게 된다.
2. Table access를 진행한다.

<p align="center"><img width="700" alt="in_list_iterator" src="https://github.com/user-attachments/assets/d6521916-13d5-44a0-bc1c-26cf23e8b0f7">

> Index가 `IN` 조건에 있는 column을 선두 column으로 가질 때에만 IN-List Iterator 방식으로 처리할 수 있다.  
> 만약 아니라면, Index range scan을 진행할 수 없기 때문이다.

<br/>
<br/>

아래 query도 동일하게 IN-List Iterator 방식으로 처리된다.

```sql
SELECT *
FROM ORDERS
WHERE (STATUS = 'SHIPPED'
    OR STATUS = 'DELIVERED');
```

`IN` 조건은 `OR` 조건을 표현하는 다른 방식일 뿐이다.

<br/>

> [[참고] Inlist Iterator를 사용하지 말아야 할 때](https://scidb.tistory.com/entry/Inlist-Iterator%EB%A5%BC-%EC%82%AC%EC%9A%A9%ED%95%98%EC%A7%80-%EB%A7%90%EC%95%84%EC%95%BC-%ED%95%A0-%EB%95%8C)
>
> `IN` 조건 vs. Range 조건
> - **Access 하고자 하는 데이터가 연속적으로 존재할 때,** `IN` 조건 보다는 Range 조건(`BETWEEN` 또는 `LIKE` 조건)을 사용하는 것이 유리하다.  
    IN-List Iterator 방식은 Index range scan을 `IN` 조건마다 반복하기 때문에, 불필요한 Index inner block lookup이 반복되어 성능상 불리하기 때문이다.
>
>
> - Access 하고자 하는 데이터가 연속적이지 않을 때에는 `IN` 조건을 사용해야 한다.  
    그러나 `BETWEEN`을 사용하는 방법도 있는데, 연속적인 데이터들 중 단 몇 개만 빠지는 경우에만 사용해야 성능적으로 유리할 것으로 보인다.
> ```sql
> SELECT *
> FROM T
> WHERE ID BETWEEN 1 AND 10
>   AND ID != 5;
> ```
