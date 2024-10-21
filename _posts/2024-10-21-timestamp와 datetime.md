# Oracle DB의 timestamp 자료형에 대하여 알아보자 

```sql
-- DB time zone 을 UTC+9 로 설정
ALTER DATABASE SET TIME_ZONE = 'Asia/Seoul';
```

```sql
-- 테이블 생성
create table tb_timestamp
(
    id  number primary key,
    ts  timestamp,
    ts3 timestamp(3),
    ts6 timestamp(6),
    ts9 timestamp(9)
);
comment on column tb_timestamp.id is 'id';
comment on column tb_timestamp.ts is '기본 timestamp';
comment on column tb_timestamp.ts3 is '밀리초 timestamp';
comment on column tb_timestamp.ts6 is '마이크로초 timestamp';
comment on column tb_timestamp.ts9 is '나노초 timestamp';
```

```sql
-- 데이터 생성
insert into tb_timestamp (id, ts, ts3, ts6, ts9)
values 
( 
  1,
  timestamp '2024-10-22 00:00:00.123456789',
  timestamp '2024-10-22 00:00:00.123456789',
  timestamp '2024-10-22 00:00:00.123456789',
  timestamp '2024-10-22 00:00:00.123456789'
);
```
저장된 데이터를 조회해보면 다음과 같다.

| ID   | TS                         | TS3                     | TS6                        | TS9                           |
| ---- | -------------------------- | ----------------------- | -------------------------- | ----------------------------- |
| 1    | 2024-10-22 00:00:00.123457 | 2024-10-22 00:00:00.123 | 2024-10-22 00:00:00.123457 | 2024-10-22 00:00:00.123456789 |

timestamp 컬럼의 크기를 확인해보자.

vsize() 는 Oracle에서 특정 데이터가 메모리에서 차지하는 실제 바이트 크기를 확인할 수 있는 함수이다.
```sql
-- 저장된 데이터 조회
select vsize(id), vsize(ts), vsize(ts3), vsize(ts6), vsize(ts9) from tb_timestamp;
```
데이터를 조회해보면, 아래와 같이 조회된다.
Oracle 에서 timestamp 필드는 11바이트이다.

|VSIZE(ID)|VSIZE(TS)|VSIZE(TS3)|VSIZE(TS6)|VSIZE(TS9)|
|---|---|---|---|---|
|2|11|11|11|11|

표현할 수 있는 범위는 아래와 같다.
```sql
SELECT TIMESTAMP '-4712-12-31 23:59:59.999999999' AS min_timestamp FROM dual; -- 기원전(B.C.)
SELECT TIMESTAMP '9999-12-31 23:59:59.999999999' AS max_timestamp FROM dual;  -- 서기(A.D.)
```



