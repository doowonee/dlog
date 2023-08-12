# PostgreSQL

[공식 도커 이미지](https://hub.docker.com/_/postgres)를 보면 자세하게 admin으로서 알아야 할 사항들이 적혀 있다. 기본 유저이름은 `postgres` 인것 같고 자동으로 [initdb](https://www.postgresql.org/docs/9.5/app-initdb.html) 과정이 진행 되는것으로 보인다. datbase 파일 경로는 기본이 `/var/lib/postgresql/data` 이다. 공식 이미지 크기는 315MB이다.

```bash
docker run -d --name postgresql \
    --network=host \
    --log-driver json-file --log-opt max-size=4g \
    -e "POSTGRES_PASSWORD=password_is_sceret" \
    -v /pg-data:/var/lib/postgresql/data \
    postgres:13.3
```

실행 하면 아래처럼 나온다.

```text
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.utf8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

initdb: error: directory "/var/lib/postgresql/data" exists but is not empty
It contains a lost+found directory, perhaps due to it being a mount point.
Using a mount point directly as the data directory is not recommended.
Create a subdirectory under the mount point.
```

언어야 영어로 되고 상관없을것 같고 인코딩은 utf이니 그대로 일단 사용하려 한다. 그러나 외부 볼륨 바인딩한 `/pg-data` 다른 디렉토리가 있어서 initdb 작업이 에러가 난것을 볼수 있다. 전부 제거하고 다시 시도해 준다. 기본 타임존은 UTC로 적용 된다.

설정 종류는 PDF 내용 그대로 웹 페이지 <https://www.postgresql.org/docs/13/runtime-config.html>에도 있는것으로 보인다.

`docker exec -it postgresql bash` 실행해서 컨테이너 내부 쉘을 열고 `psql -U postgres`로 SQL 터미널을 실행 할수 있다.

## Tablespace

[공식 문서](https://www.postgresql.org/docs/current/manage-ag-tablespaces.html)를 보면 되는데 removal 한 스토리지에 table space를 생성하지 말라고 하네 기본 `pg_default`, `pg_global` 존재 한다. 이게 `pg-data` 디렉토리 기본 PostgreSQL Data Directory가 아닌 다른 디렉토리로 지정해야 한다는거다. 오라클이랑 약간 다른 개념인거 같음 쓸일 없을듯? 스토리지 서버가 따로있거나 attach 하는 디스크가 엄청 많을때 뭐 이럴떄 쓰는거 같음

## Schema vs Database

[Schema](https://www.postgresql.org/docs/current/ddl-schemas.html)는 테이블 생성시 자동으로 `public`으로 테이블이 생성되고 [Database](https://www.postgresql.org/docs/current/sql-createdatabase.html)는 아예 접근제어부터 분리되어서 다른 서버를 사용듯한 느낌을 주는것으로 MySQL database에서 좀더 강화형인거 같다. DDL로 테이블만들면 자동으로 `public` schema에 테이블이 생성된다. 일단 database 따로 만들어서 하자. 뭐가 정답인지는 아직은 잘 모르겠다. scema는 사용하지 말자 일단ㅇㅇ

```sql
-- DB 목록 조회
SELECT * FROM pg_database;
-- 스키마 목록 조회
SELECT * FROM information_schema.schemata;
-- DB 생성
CREATE DATABASE my_db

-- 인덱스 목록 조회
SELECT
    tablename,
    indexname,
    indexdef
FROM
    pg_indexes
WHERE
    schemaname = 'public'
ORDER BY
    tablename,
    indexname;

-- 테이블 통계 정보로  n_live_tup 이 예상 row 갯수임 tuple 이 곧 row
SELECT *  FROM pg_stat_user_tables
-- 테이블 디스크 공간 사용률 보는 쿼리 public은 스키마 이름으로 db 이름인 db_name 보다 하위 개념임
SELECT pg_size_pretty(pg_total_relation_size('"public"."users"'));

-- row 갯수 많은것 부터 정렬
SELECT
    relname,
    n_live_tup
FROM pg_stat_user_tables
WHERE schemaname = 'public'
ORDER BY 2 DESC
LIMIT 20;

-- 테이블 디스크 사용 순서 별로 정렬
SELECT
  table_name,
  pg_size_pretty(pg_relation_size(quote_ident(table_name)))
FROM information_schema.tables
WHERE table_schema = 'public'
ORDER BY 2 DESC
LIMIT 20;

-- 임시로 데이터 랜덤 문자열과 순서대로 데이터 삽입 하기
do $$
begin
for r in 1..500000 loop
INSERT INTO t_t (c1, c2, c3, c4, c5)
VALUES('test_' || r, md5(random()::text), now()::timestamp, NULL, NULL);
end loop;
end;
$$;

-- 이게 압도적으로 빠름 위는 6초 밑에는 1초 이 방식이 훨씬 빠르다
INSERT INTO t_t (c1, c2, c3, c4, c5)
SELECT 'test_' || gs, md5(random()::text), now()::timestamp, NULL, NULL
FROM generate_series(0,100000) gs

-- 일자 변환 하는 함수로 기존 without timezone 컬럼의 값을 UTC 적용한 시간 기준 컬럼 변환
-- 그 변환 한걸 다시 한국 시간으로 변환
EXTRACT(DAY FROM (dt AT TIME ZONE 'UTC' AT TIME ZONE 'Asia/Seoul'))::INTEGER % 5 != 0
```

## DDL

Primary key나 unique constraint를 사용할때만 index를 자동생성한다. Forigen constraint 생성시 자동으로 index를 생성 하지 않는다.

```sql
-- 테이블 컬럼 정보 조회
SELECT *
FROM information_schema.columns
WHERE table_schema = 'public'
AND table_name   = 'table_name';

-- 컬럼 제거
ALTER TABLE t_t DROP COLUMN c_c;

-- 컬럼 추가
ALTER TABLE t_t ADD COLUMN c_c SMALLINT;
ALTER TABLE t_t ADD COLUMN c_c VARCHAR NOT NULL DEFAULT '[0]'
```

## DML

insert 문 실행시 기본 `INSERT oid count`를 반환 한다. 즉 아래 쿼리 실행시 하단 줄을 반환 한다.

```sql
-- 기본 insert
INSERT INTO links (url, name)
VALUES('https://www.postgresqltutorial.com','PostgreSQL Tutorial');

-- 응답 값 INSERT oid count insert의 oid는 0
-- 여기서 oid는 pg 내부에서 쓰는 internally as a primary key for its system tables
INSERT 0 1

-- RETURNING clause 사용
INSERT INTO table_name(column1, column2, …)
VALUES (value1, value2, …)
RETURNING id;
```

delete 문도 실행시 기본적으로 삭제한 row 갯수를 반환 하는데 `RETURNING` clause도 지원한다.

## Full text search

핵심은 기본으로 제공되는 [pg_trgm](https://www.postgresql.org/docs/13/pgtrgm.html) extension 인데 gitlab의 스키마를 보면 `CREATE EXTENSION IF NOT EXISTS pg_trgm;`를 통해 extension을 활성화 하고 제목이나 내용등의 컬럼에 `CREATE INDEX index_issues_on_title_trigram ON issues USING gin (title gin_trgm_ops);` 이런식으로 Btree 인덱스가 아닌 다른 인덱스를 사용한다. extesion 설치 후 `SELECT * FROM pg_extension;` 로 확인할수 있다.

[GIN VS GIST 공식문서](https://www.postgresql.org/docs/9.1/textsearch-indexes.html)에 따르면 `GIN`이 `GIST`보다 인덱싱에 시간은 더 걸리지만 탐색 속도는 더 빠른것으로 보인다.

## pgAdminv4

GUI 툴에서도 import export를 할수있는데 기본 CSV로 할수있음 근데 이건 서로 다른 DB간에 사용하기에는 부적절 하다. 왜냐면 컬럼 순서가 일치해야함. 대신 테이블을 누르고 tool 메뉼에 backup을 보면 `pg_dumpp` 툴을 GUI로 사용할수 있다. data-only 옵션에 column-data insert 구문 옵션을 체크 하면 컬럼도 선언되어있는 insert 쿼리로 백업 데이터를 얻을수 있음. `pg_dumpp` 기본 데이터는 insert 쿼리 형식이 아니라 raw copy 형식임

기본적으로 백업본을 부으려면 format은 custom으로 해야 하며 부으려는 데이터베이스에 해당 데이터베이스를 drop 해야한다 superset 설치 이후 db 컨테이너만 기동 후 databse drop 하고 부어야 하며 백업 할때 role, tablespace, privilege는 제외하고 스키마랑 데이터만 백업 하게 해야한다.

export 시 format은 custom으로 해야 하고 pgadmin4 dialog 옵션에서 role, tablespace, privilege 제외하고 database 생성 관련 `--create --clean`옵션도 없도록 선택해서 `/Applications/pgAdmin 4.app/Contents/SharedSupport/pg_dump --file "/local_location/44" --host "172.16.1.1" --port "5432" --username "postgres" --no-password --verbose --format=c --blobs --no-owner --no-privileges --no-tablespaces --no-unlogged-table-data --encoding "UTF8" "superset"` 처럼 하고 import 대상 db에서 database를 생성하고 그 커넥션으로 restore를 해야 한다 또한 반드시 role 관련 부분을 처리 되지 않게 제외 해야 한다. pgadmin에서 `/Applications/pgAdmin 4.app/Contents/SharedSupport/pg_restore --host "172.16.1.1" --port "5432" --username "superset" --no-password --dbname "superset" --no-owner --no-privileges --no-tablespaces --verbose "/local_location/44"` 이렇게 되어야함

## 로깅 파라미터 표

| 이름  | 설명  |
|---------|---------|
| log_temp_files  |   지정된 값 이상의 임시 파일을 사용하는 쿼리 기록 대부분 full table scan   |
| Log_min_messages  |   로그 레벨 필터링   |
| log_lock_waits  |   트랜잭션으로 교착상태 지정 시간 이상 교착 상태시 기록    |
| log_statement  |   로그 남길 쿼리 기준 적용 DDL, DML 등 all 사용은 비추    |
| log_min_duration_statement  |   지정 시간 이상 실행 시간 쿼리 기록    |
| log_autovacuum_min_duration  |   지정 시간 이상 autovacuum 실행 기록    |
| rds.force_autovacuum_logging_level  |   autovacuum 로깅 로그레벨    |
| auto_explain.log_min_duration  |   지정 실행시간이 긴 쿼리의 실행 계획 기록   |
| shared_preload_libraries  |   auto_explain, log_min_duration 관련     |
| rds.force_admin_logging_level       |   마스터 계정의 활동 로그 레벨    |
