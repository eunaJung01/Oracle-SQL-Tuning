# Index Basic

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

즉, 물리적으로 disk에 저장된 순서대로 index leaf block들을 읽어들이는 것이다.

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

## Query 3: 전체 회원 조회 - Index Column + ORDERY BY DESC

```sql
SELECT MEMBER_ID
FROM MEMBER
ORDER BY MEMBER_ID DESC;
```

`DESC` : 내림차순 정렬

<br/>

### Execution Plain

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

그 후에 실행 계획을 출력해보면 ..

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
