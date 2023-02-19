---
layout: post
title: Index
parent: DB
---
<details open markdown="block">
  <summary>
    목차
  </summary>
    {: .no_toc .text-delta }

1. TOC
{:toc}
</details>  

# Index 란
흔히 index는 목차/색인으로 많이 비유한다.   
인덱스는 목차/색인처럼 특정 구간(data) 등을 찾기 위하여 사용된다.

DB에서 인덱스는 table의 data를 검색할 때, 전체 rows를 검색하는 것이 아닌 일부 rows 검색하기 위한 방법이다.

즉, Disk IO를 줄이는것이 목표로 작용

> 전체 rows(Record)를 검색하는 것을 DBMS에 따라 아래와 같이 표현한다.   
> MySQL: Full Scan   
> MSSQL: Table Scan(≒Index Scan)   
> Oracle: Table Scan   

예를 들어 10억건의 데이터가 존재하는 테이블에서 Full Scan, Table Scan이 일어날 경우 총 10억개의 rows를 검색하는 효율이 나오게 된다.   

인덱스는 B-Tree를 차용하여 데이터를 검색한다.
> 검색의 경우 $O_{(logN)}$의 시간 복잡도를 가진다.   

단, 삽입 / 삭제는 
>$O_{(N)}$의 시간 복잡도를 가진다.

인덱스의 설정은 데이터를 빠르게 검색이 가능하나, 삽입 삭제가 많은 도메인의 경우 좋지 않은 효율을 가져올 수 있으므로 신중하게 선택할 필요가 있다.

OLTP 성격을 가진 DB의 경우 검색 쿼리 비중이 더 높으므로, 빠른 latency가 필요한 경우 Index 설정을 고려 해야 한다.

## 정렬된 인덱스(Clustered Index)
물리적으로 정렬된 인덱스를 Clustered Index로 칭하며 B+ Tree 형태로 저장되어 있다.   

물리적으로 정렬되어 저장되어 있기 때문에 범위 검색에서의 이점을 가져간다.   

> MySQL, MSSQL(SQL Server)에서 PK가 Clustered Index로 생성된다.     
   
물리적으로 정렬되어 저장되어 있어 삽입/삭제에서는 비용이 증가하게 된다.   

MSSQL(SQL Server) 에서 Clustrered index의 leaf page 는 data page를 의미하며, leaf page는 정렬되어 저장됩니다.

## 정렬되지 않은 인덱스(Non-Clustered Index)
물리적으로 정렬되지 않은 인덱스를 흔히 Non-Clustered Index로 말한다.   
물리적으로 정렬되어 저장되어 있지 않지만 hash 형태로
index 영역이 따로 저장되어 있다.   
> Oracle, PostgreSQL는 PK가 non-Clustered Index로 생성된다.  
> MySQL, MSSQL의 경우 Unique를 사용하거나 index 를 지정하여 생성할 경우 non-Clustered Index로 생성된다.   

Index page는 데이터가 쌓인 순서대로 가지게 됩니다.

non-Clustered Index의 경우 범위 검색이 아닌, 포인트(Equals, in 등) 검색 시 높은 효율을 가지게 된다.

데이터의 중복이 많은 경우(성별과 같은 column)는 Index page를 찾고 data page를 찾는 것은 비용이 더 증가하게 된다.
> data page를 바로 찾는것이 오히려 더 효율적인 상황

MSSQL(SQL Server) 에서
Non-Clustered Index의 leaf page는 data가 포함되어 있지 않으며, RID(Row Identifier) 형태로 data를 검색하게 됩니다.(Heap 구조)   
> Index Seek -> RID Lookup : non-clustered index로 데이터 검색 이후 RID로 데이터 page 조회   

예제)
```sql
CREATE TABLE IF NOT EXISTS test_01 (
    BIGINT id PRIMARY KEY NOT NULL
    , BIGINT seconary_id 
    , varchar(50) name
)

CREATE NONCLUSTERED INDEX IX_tbltest_sec ON test_01(secondary_id ASC);

INSERT INTO test_01 VALUES
(1,1,'Kim')
, (2,2,'Yoo')
, (3,3,'Park');

SELECT name 
FROM test_01
WHERE secondary_id = 2
```

> 단! index scan -> RID Lookup 를 진행할 경우 전체 data page를 조회 하는 경우 index leaf page를 전체를 읽으면서 RID를 통하여 data를 읽는 경우 이다.

예제)
```sql
CREATE TABLE IF NOT EXISTS test_01 (
    BIGINT id PRIMARY KEY NOT NULL
    , BIGINT secondary_id 
    , varchar(50) name
)

CREATE NONCLUSTERED INDEX IX_tbltest_sec ON test_01(secondary_id ASC);

INSERT INTO test_01 VALUES
(1,1,'Kim')
, (2,2,'Yoo')
, (3,3,'Park');

SELECT name
FROM test_01
WHERE 1 = secondary_id;
-- secondary_id가 우항에 있어 index가 있지만 전체를 읽게 됨.
```


# ReF.
- [SQL Server Index architecture](https://learn.microsoft.com/ko-kr/sql/relational-databases/sql-server-index-design-guide?view=sql-server-ver16)
- [SQL Server Non-Clustered index](https://learn.microsoft.com/ko-kr/sql/relational-databases/indexes/create-nonclustered-indexes?view=sql-server-ver16)   
- [SQL Server Heap architecture](https://learn.microsoft.com/ko-kr/sql/relational-databases/indexes/heaps-tables-without-clustered-indexes?view=sql-server-ver16)
- [MySQL Clustered index](https://dev.mysql.com/doc/refman/5.7/en/innodb-index-types.html)
- [MySQL Index](https://dev.mysql.com/doc/refman/5.7/en/mysql-indexes.html)
- [tistory Blog](https://swingswing.tistory.com/3)