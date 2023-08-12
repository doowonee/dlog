# Clickhouse

[System Requirement](https://clickhouse.tech/docs/en/operations/requirements/)를 보면 코어 클럭수 보다 코어 갯수가 많은것을 더 권고 하고 있고 최소 4GB 이상의 메모리를 추천하고 있다. 평균 데이터 압축률은 6~10배이다. Distributed query가 많으면 네트워크 성능도 중요하다. <https://clickhouse.tech/docs/en/operations/tips/>를 보면 메모리는 많을 수록 대용량 쿼리 성능이 대폭 향상 되는것으로 보이고 파일 시스템은 `ext4`를 권고함.

Clickhouse 설정 파일은 `macros.xml`, `config.xml`, `users.xml` 총 3가지가 존재 한다. 그중 `macros.xml`은 replica 구성시 사용하며 자세한건 [링크](https://clickhouse.tech/docs/en/engines/table-engines/mergetree-family/replication/#creating-replicated-tables) 참조 하면 된다. 결론적으로 built-in substitutions이 있기 때문에 건드릴 일은 없다고 보는게 나을것 같다.

## 쿼리 노트

```sql
-- CH disk 정보
SELECT
    name,
    path,
    formatReadableSize(free_space) AS free,
    formatReadableSize(total_space) AS total,
    formatReadableSize(keep_free_space) AS reserved
FROM system.disks

-- 테이블 별로 스토리지 사용률 내림차순 GB 단위로 출력 row 갯수 1백만 단위로 출력
SELECT name, storage_policy, intDivOrZero(total_rows, 1000 * 1000) as row_count_1m, intDivOrZero(total_bytes, 1024 * 1024 * 1024) as usuage_gb
FROM system.tables
ORDER BY usuage_gb DESC

-- 100분 이내로 실행 했던 DROP이 포함된 쿼리 최신순으로 20개만 출력
SELECT * FROM system.query_log WHERE position(query, 'DROP') > 0 AND position(query, 'SELECT') == 0 AND  event_time >= subtractMinutes(now(), 100) ORDER BY event_time DESC LIMIT 20

-- 특정 데이터베이스에 존재하는 테이블 목록 조회 tabix 안쓸때 유용함 테이블 뭐뭐있는지 모르다 보니..
SELECT name FROM system.tables WHERE database = 'db_name'

-- 100분이내로 특정 테이블 질의 쿼리 실행 로그 보기
SELECT * FROM system.query_log
WHERE position(query, 'SELECT') > 0
AND position(query, 'TABIX') == 0 
AND position(query, 'table_name') > 0
AND event_time >= subtractMinutes(now(), 100)
ORDER BY event_time DESC
LIMIT 20

-- lifetime이 0인 영구 dictionary들 invalidate 하는 명령어
SYSTEM RELOAD DICTIONARY
-- replacing merge tree merge 하기
OPTIMIZE TABLE db.table FINAL DEDUPLICATE;

-- delete update 등 mutation 상태 조회
SELECT * FROM system.mutations WHERE table = 'some_log'

-- 컬럼 추가
ALTER TABLE db.w_logs ADD COLUMN content_group_id String DEFAULT '' AFTER content_type;

-- s3에 있는 csv.gz 바로 table에 insert 하기
-- Select 후 테이블의 기본 컬럼 순서와 맞추고
-- from s3는 CSV의 컬럼 순서와 맞춰야 한다 타입은 전부 String으로 하고 쿼리로 예외상황 처리하는게 낫다
-- parseDateTimeBestEffort 통해 2022-01-29 11:24:49 이아닌 2022-01-29T11:24:49Z 형태도 datetime으로 변환 되어 들어 가게 된다 도저히 파싱안되면 epoch 0 되는거임
-- w_logs/*/*/*/*/*.csv.gz 를 path로 하면 w_logs 산하 모든 file을 import함
INSERT INTO db.w_logs
SELECT parseDateTimeBestEffortOrZero(event_at, 'UTC'), lang, toUInt32OrZero(current_position), log_id
FROM s3('https://s3.ap-northeast-2.amazonaws.com/repo-name/w_logs/2021/08/26/12/23_05_1.csv.gz', 'aws_cred','aws_cred_secret', 'CSVWithNames',
'event_at String, lang String, current_position String, log_id String', 'gzip');

-- CH 데이터를 읽어서 gzip 압축 후 CSV 파일 업로드
INSERT INTO FUNCTION s3('https://s3.ap-northeast-2.amazonaws.com/repo-name/w_logs/2021/08/26/12/23_05_1.csv.gz', 'aws_cred','aws_cred_secret', 'CSVWithNames', 'event_at String, lang String, current_position String, log_id String', 'gzip')
SELECT event_at, lang, current_position, log_id FROM db.w_logs;

-- INSERT statment 에 파티션이 100 개이상 다르면 에러남 이때는 INSERT 할때 SETTINGS 값 변경 하면 처리 가능함
INSERT INTO db.w_logs
SELECT parseDateTimeBestEffortOrZero(event_at, 'UTC'), lang, toUInt32OrZero(current_position), log_id
FROM s3('https://s3.ap-northeast-2.amazonaws.com/repo-name/w_logs/2021/08/26/12/23_05_1.csv.gz', 'aws_cred','aws_cred_secret', 'CSVWithNames',
'event_at String, lang String, current_position String, log_id String')
SETTINGS max_partitions_per_insert_block=0;

-- 시간이 epoch mill일때 datetime으로 변환
SELECT fromUnixTimestamp64Milli(epoch_mill) AS dt, msg FROM db.logs WHERE epoch_mill > toUnixTimestamp64Milli(toDateTime64('2022-01-01 21:23:00', 3))
```

```bash
# 배포 스크립트 
docker run -d --name clickhouse \
    --network=host \
    --log-driver json-file --log-opt max-size=3g \
    -v /hot-data:/var/lib/clickhouse \
    -v /cold-data:/cold-data \
    -v /home/ubuntu/my-config.xml:/etc/clickhouse-server/config.xml \
    -v /home/ubuntu/my-users.xml:/etc/clickhouse-server/users.xml \
    yandex/clickhouse-server:21.3
```

외부 볼륨을 docker 사용시 별도의 디렉토리에 마운트 해서 사용할경우 **There is no RW access to disk** 에러가 발생해서 cold-storage는 사용이 불가능 할것이다. 공식 clickhouse 컨테이너가 사용하는 유저 정보는 `uid=101(clickhouse) gid=101(clickhouse) groups=101(clickhouse)` 같으므로 그룹권한은 유지 하지만 소유자 권한을 명령어 `sudo chown -R 101:ubuntu /hot-data && sudo chown -R 101:ubuntu /cold-data` 사용하여 변경 해준다. host OS 유저인 ubuntu는 쓰기 권한이 필요 없고 읽기는 가능해야 하므로 그룹 소유자는 ubuntu로 지정한다. 호스트 OS인 ubuntu 에서는 이 `101` 숫자가 `systemd-resolve`란 이름이라서 컨테이너 기동하면 소유자가 변경 되는거였다.

`docker exec -it clickhouse bash`로 컨테이너 쉘 접속하고 `clickhouse-client --user=user_id --password=user_pwd`로 서버에서 로컬로 붙는 CH 클라이언트 사용 가능하다.

자체적으로 HTTP health check를 지원함 `8123` 포트로 `/ping`, `/replicas_status`

최신 데이터만 조회 하고 싶을 때 `WHERE kst_datetime < now() order by kst_datetime DESC` 이렇게 하면 엄청 느리다. 나는 쿼리를 이렇게 하면 clcikhouse가 데이터 스캔을 뒤에서 부터 하는줄 알았는데 그게아니라 처음부터 스캔하는거였음 즉 최초 데이터 부터 시간이 지금 이하의 데이터를 스캔하고 그 결과를 정렬 해서 보여주는것이었다.  그래서 스캔 범위 자체를 줄여야 하기 때문에 `WHERE kst_datetime > subtractMinutes(now(), 10) order by kst_datetime DESC` 이렇게 하면 빠르게 스캔하고 그 결과를 정렬해서 보여준다.

`Distributed` table의 data source 테이블 `_local`이 바뀌면 Distributed도 바뀌어야 한다 특히 스키마 변경시 필수로 바꾸어 주어야함

Clickhouse는 DML로 row 값 수정과 DDL로 컬럼을 변결하는건 모두 `Alter`로 하는데 `sort key`에 해당하는 컬럼은 값 수정도 타입 변경도 할수 없다.

csv로 export 하는건 `clickhouse-client --query "SELECT * from table" --format FormatName > result.txt` 입력 한뒤 docker cp 명령어로 csv 파일을 꺼내오고 scp로 파일 옮기면 된다.

Distributed 처럼 Clickhouse 에서 `Replica`는 zookeeper를 사용하여 그냥 테이블을 DROP 하면 곧 바로 테이블 생성이 안된다. 뒤체 `SYNC` 명령을 붙이면 바로 테이블 생성 가능하다.

## Superset과 PostgreSQL Engine

[문서](https://clickhouse.com/docs/en/engines/table-engines/integrations/postgresql/)를 보다시피 PostgreSQL에 있는 데이터를 Clickohouse에 삽입 할필요 없이 바로 조회 할수 있다. `superset`에서 조인은 같은 종류의 데이터베이스에서만 가능하므로 원래라면 ReplicatedMergeTree 사용해서 별도의 동기화 프로세스를 만들어야 하지만 PostgreSQL Engine 을 사용해서 손 쉽게 CH를 통해 PG에 Select 해서 CH에 데이터 그대로 가저올수 있다. `Primary Key`는 선언 하지 않아도 된다. 근데 주의 할점은 PostgreSQL과 Clickhouser간 `datetime` 데이터 구조가 달라서 Clickhouse에서 지원하지 않는 날짜 범위나 단위일경우 값이 이상하게 들어감

```sql
-- PostgreSQL Nullable 한 컬럼 주의 해야함
-- PosergreSQL TIMESTAMP -> Datetime 자동으로 변환됌 초 이하 단위는 버려짐
CREATE TABLE db.s_log
(
    `id` String,
    `user_id` String,
    `start_at` Datetime,
    `cancel_at` Nullable(Datetime),
)
ENGINE = PostgreSQL('172.30.xxx.xxx:x432', 'db', 'table', 'pg_id', 'pg_pwd');
```

근데 수퍼셋에서 dataset 추가 하면 에러남 근데 SQL Lab에서 조인해서 사용은 가능한거 확인함 `clickhouse-sqlalchemy` 0.1.10 버전 사용시 추가되드라 0.1.6 이 superset 공식 문서에 적힌 버전인데 이걸로 하면 안됌

DDL 로 컬럼 변경은 불가능함 `Received from localhost:9000. DB::Exception: Alter of type 'ADD_COLUMN' is not supported by storage PostgreSQL. (NOT_IMPLEMENTED)` 에러남 그래서 테이블 드랍하고 하거나 애초에 추가를 PostgreSQL 테이터베이스로 하면 `detach`, `attach`를 통해 갱신된 스키마 스펙 그대로 테이블 질의가 가능하다.

```sql
SELECT * FROM system.replicas;\G
-- replica 관련 설정 주키퍼에 있는 replica 용 데이터랑 ch에 있는 replica 테이블 DDL 이랑 뭔가 별개 인거 같음
-- 20분 이상 지나면 뭔가 반영되서 테이블 생성 가능함 아래 명령어 들로 뭔사 다른 처리를 해야 하는거 같음
SELECT * FROM system.replicas;\G
SYSTEM SYNC REPLICA db.w_logs;
SYSTEM RESTART REPLICA db.w_logs;
SYSTEM DROP REPLICA 'replica_name' FROM TABLE database.table;
-- replica 된 테이블은 뒤에 SYNC 붙이면 바로 테이블 생성 가능 해진다
DROP TABLE db.w_logs ON CLUSTER 'replicated' SYNC;
```

## tabix GUI

[tabix](https://tabix.io/doc/Install/)는 GUI 웹 콘솔인데 [repo](https://github.com/tabixio/tabix)를 보다시피 3년전이 마지막 활동이다. 그리고 공식 docker image는 없지만 [docker hub](https://hub.docker.com/r/spoonest/clickhouse-tabix-web-client)에 누군가 이미지를 올려 놨다.

```bash
docker run -d --name tabix \
    -p 8011:80 \
    --log-driver json-file --log-opt max-size=1g \
    spoonest/clickhouse-tabix-web-client:latest
```

로그인 화면에서 `users.xml`로 지정한 유저 정보를 통해 로그인 하면된다. 참고로 DB서버 경로는 HTTP 경로로 8123포트를 적으면 된다. 별도의 컨테이너 없이 아래 처럼 하면 tabix spa에 필요한 스크립트와 css를 로드해서 clickhouse내에서 기본 http 포트 8123으로 tabix를 서빙 할수있다. 이러면 웹 브라우져서 html 파싱해서 spa가 로드 되는 방식인데 23년 6월 즈음 `http://ui.tabix.io/`서버에 문제가 있어서 동작 안하는 문제 발생했었음.

```xml
<dictionaries_lazy_load>true</dictionaries_lazy_load>

<http_server_default_response><![CDATA[<html ng-app="SMI2"><head><base href="http://ui.tabix.io/"></head><body><div ui-view="" class="content-ui"></div><script src="http://loader.tabix.io/master.js"></script></body></html>]]></http_server_default_response>

<http_handlers>
    <defaults/>
</http_handlers>
```

## startup log

컨테이너 재기동 했는데 생각보다 기동 완료 까지 시간이 좀 걸리네 내부적으로 뭔가 많이 처리 하고 최종 서빙 준비 완료 됌

```log
-4fb0-887e-cc0118519973/15170.bin. 5461 rows, 283.25 KiB. State has 153478 unique rows.
2023.03.02 08:38:06.830337 [ 77 ] {} <Information> StorageSetOrJoinBase: Loaded from backup file store/e34/e34c217c-56be-4fb0-887e-cc0118519973/15171.bin. 5461 rows, 283.25 KiB. State has 153478 unique rows.
2023.03.02 08:38:06.831733 [ 77 ] {} <Information> StorageSetOrJoinBase: Loaded from backup file store/e34/e34c217c-56be-4fb0-887e-cc0118519973/15172.bin. 5461 rows, 283.25 KiB. State has 153478 unique rows.
2023.03.02 08:38:06.833200 [ 77 ] {} <Information> StorageSetOrJoinBase: Loaded from backup file store/e34/e34c217c-56be-4fb0-887e-cc0118519973/15173.bin. 5461 rows, 283.25 KiB. State has 153478 unique rows.
2023.03.02 08:38:06.834524 [ 77 ] {} <Information> StorageSetOrJoinBase: Loaded from backup file store/e34/e34c217c-56be-4fb0-887e-cc0118519973/15174.bin. 5461 rows, 283.25 KiB. State has 153478 unique rows.
2023.03.02 08:38:06.835798 [ 77 ] {} <Information> StorageSetOrJoinBase: Loaded from backup file store/e34/e34c217c-56be-4fb0-887e-cc0118519973/15175.bin. 5461 rows, 283.25 KiB. State has 153478 unique rows.
2023.03.02 08:38:06.835855 [ 77 ] {} <Information> DatabaseAtomic (csService): 91.66666666666667%
2023.03.02 08:39:34.113059 [ 76 ] {} <Information> DatabaseAtomic (csService): 100.0%
2023.03.02 08:39:34.113126 [ 45 ] {} <Information> DatabaseAtomic (csService): Starting up tables.
2023.03.02 08:39:34.120931 [ 45 ] {} <Information> DatabaseAtomic (test): Total 3 tables and 0 dictionaries.
2023.03.02 08:39:34.149372 [ 45 ] {} <Information> DatabaseAtomic (test): Starting up tables.
2023.03.02 08:39:34.150843 [ 45 ] {} <Information> DatabaseAtomic (test2): Total 2 tables and 0 dictionaries.
2023.03.02 08:39:34.160134 [ 45 ] {} <Information> DatabaseAtomic (test2): Starting up tables.
2023.03.02 08:39:34.162274 [ 45 ] {} <Information> DatabaseAtomic (viewlog): Total 2 tables and 0 dictionaries.
2023.03.02 08:40:06.173051 [ 80 ] {} <Information> DatabaseAtomic (viewlog): 100.0%
2023.03.02 08:40:06.173112 [ 45 ] {} <Information> DatabaseAtomic (viewlog): Starting up tables.
2023.03.02 08:40:06.176348 [ 45 ] {} <Information> DatabaseAtomic (viewlog_dev): Total 2 tables and 0 dictionaries.
2023.03.02 08:40:06.179595 [ 45 ] {} <Information> DatabaseAtomic (viewlog_dev): Starting up tables.
2023.03.02 08:40:06.182733 [ 45 ] {} <Information> DatabaseAtomic (moup): Total 9 tables and 0 dictionaries.
2023.03.02 08:40:56.515593 [ 79 ] {} <Information> DatabaseAtomic (moup): 66.66666666666667%
2023.03.02 08:41:43.911788 [ 82 ] {} <Information> DatabaseAtomic (moup): 100.0%
2023.03.02 08:41:43.911845 [ 45 ] {} <Information> DatabaseAtomic (moup): Starting up tables.
2023.03.02 08:41:43.916554 [ 45 ] {} <Information> DatabaseAtomic (ytb): Total 2 tables and 0 dictionaries.
2023.03.02 08:41:43.917561 [ 45 ] {} <Information> DatabaseAtomic (ytb): Starting up tables.
2023.03.02 08:41:43.917837 [ 45 ] {} <Information> DatabaseCatalog: Found 0 partially dropped tables. Will load them and retry removal.
2023.03.02 08:41:43.917922 [ 45 ] {} <Information> Application: It looks like the process has no CAP_SYS_NICE capability, the setting 'os_thread_priority' will have no effect. It could happen due to incorrect ClickHouse package installation. You could resolve the problem manually with 'sudo setcap cap_sys_nice=+ep /usr/bin/clickhouse'. Note that it will not work on 'nosuid' mounted filesystems.
2023.03.02 08:41:43.918294 [ 45 ] {} <Information> Application: Listening for http://[::]:8123
2023.03.02 08:41:43.918408 [ 45 ] {} <Information> Application: Listening for connections with native protocol (tcp): [::]:9000
2023.03.02 08:41:43.918478 [ 45 ] {} <Information> Application: Listening for replica communication (interserver): http://[::]:9009
2023.03.02 08:41:43.983775 [ 45 ] {} <Information> Application: Listening for MySQL compatibility protocol: [::]:9004
2023.03.02 08:41:43.984113 [ 45 ] {} <Information> Application: Listening for http://0.0.0.0:8123
2023.03.02 08:41:43.984179 [ 45 ] {} <Information> Application: Listening for connections with native protocol (tcp): 0.0.0.0:9000
2023.03.02 08:41:43.984238 [ 45 ] {} <Information> Application: Listening for replica communication (interserver): http://0.0.0.0:9009
2023.03.02 08:41:44.057117 [ 45 ] {} <Information> Application: Listening for MySQL compatibility protocol: 0.0.0.0:9004
2023.03.02 08:41:44.078539 [ 45 ] {} <Information> DNSCacheUpdater: Update period 15 seconds
2023.03.02 08:41:44.078613 [ 45 ] {} <Information> Application: Available RAM: 30.96 GiB; physical cores: 2; logical cores: 4.
2023.03.02 08:41:44.079579 [ 45 ] {} <Information> Application: Ready for connections.
```
