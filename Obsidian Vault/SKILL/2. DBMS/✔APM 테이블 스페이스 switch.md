#DBMS #Tibero 

# 개념

## Automatic Performance Monitering 

>(매뉴얼) Tibero DBMS는 DBA가 성능 문제를 진단하는데 도움을 주기 위해 다양한 종류의 통계를 제공하고 있다. Automatic Performance Monitoring(이하 APM)은 이러한 통계 정보를 주기적으로 자동 수집하여 DBA가 이를 위한 작 업을 따로 할 필요가 없어졌고, 수집한 통계 자료에 대한 자체적인 분석 리포트 출력 기능을 제공하여 시스템 부하 분 석에 도움을 줄 수 있는 기능이다. Oracle의 `AWR(Automatic Workload Repository)에 해당`하는 기능이다.
> 
> ※ 주요기능 
>  - 스냅샷 (`v$sysstat, v$system_event, v$sqlstats, v$sgastat` 등 Tibero의 각종 성능 통계 정보를 주기적(보통 1 시간)으로 테이블에 저장 
>  - 세션상태 저장 : 1초에 한번씩 현재 `RUNNING 상태인 세션들의 ID와 대기중인 이벤트 정보(WAIT_EVENT 등)를 메모리(ash queue)에 저장`. v$active_session_history 뷰로 조회 
  

저장된 스냅샷과 세션 상태는 테이블과 뷰를 통해 직접 이용할 수 있지만, 보통은 성능 분석 리포트를 만드는데 이용한다. 


## 성능분석 리포트 

#### 1) 스냅샷 생성여부 확인 
```SQL
select * from _apm_snapshot;
```
→ 현재날짜는 2022-10 경이나, 생성된지 7일 이상인 스냅샷 32596 row 발견

##### ※ 한번의 스냅샷으로 쌓이는 데이터 양
 - 한번의 스냅샷으로 쌓이는 데이터 총량 (최대치) : 약 150KB 
 - 최대 차지하는 공갂 : 150KB * 7일 * 24시간 = 25MB 
 - `_APM_TS Tablespace`에 공간 차지 <<<< 삭제가 안되고 있었다. 
 

#### 2) 성능리포트 생성 확인 
```Shell
$TB_HOME/instance/$TB_SID/apm_report.{mthr_pid}.{current_time}
```
 → 현재 생성된 성능 리포트는 없음 


## 기타 특이사항
(매뉴얼) APM관련 테이블을 재생성하고 싶을경우 : $TB_HOME/scripts 내에 `apm.sql`, `apm_drop.sql` 을 이용하시기 바랍니다. 참고로 기존의 APM 데이터는 모두 제거 됩니다

 
`_APM_TS` 테이블스페이스의 증가는 `7일이 경과된 스냅샷이 삭제되지 않아 발생` (개발서버의 OS Time 은 정시로 고정되어 있는것이 아닌 과거 시간대와 혼용해 사용중임)

`_APM_TS` 테이블스페이스의 RESIZE 또는 DROP/CREATE 으로 처리 예정 

------
# 작업내역

## 1. 방법1> TBS RESIZE (실패)
https://oracle.tistory.com/292  포스팅 (DATAFILE SIZE를 줄이는법 - 방법1)을 참고해 `Tibero` 에 맞도록 변경 작업

※ 이 방법은 TBS의 논리공간을 축소하는 것이기 때문에 물리공간(하드디스크를 차지하는을 의미)에 위치한 DATAFILE의 용량이 줄어들진 않음. 때문에 실패라 표기 

```SQL

-- 1. 관련 파라미터 확인 (FILD_ID, BLOCK_ID, BLOCKS)
 select * from dba_data_files
 where 1=1
 and tablespace_name in ('_APM_TS','UNDO_02')
 order by tablespace_name

--#	FILE_NAME	FILE_ID	TABLESPACE_NAME	BYTES	BLOCKS	STATUS	RELATIVE_FNO	AUTOEXTENSIBLE	MAXBYTES	MAXBLOCKS	INCREMENT_BY
--1	/home/jeus/Tibero/tibero5/database/tibero/undo_002.dbf	19	UNDO_02	104857600	12800	AVAILABLE	19	YES	34359738368	4194304	32
--2	/home/jeus/Tibero/tibero5/database/tibero/apm_ts.dtf	3	_APM_TS	11417288704	1393712	AVAILABLE	3	YES	34359738368	4194304	1280

/* 요약 
 - tablespace_name : _APM_TS
 - FILE_ID : 3 
 - BLOCK_ID : X
 - BLOCKS : 1393920 (*) 
 - BYTES : 11418992640
*/



-- 2. EXTENTS 검색
select * from dba_extents
where 1=1
and file_id = '3' 
and segment_type = 'TABLE' 
order by block_id*1 desc 
 -- 마지막 BLOCK_ID , BLOCK 로 차지하는 용량 계산. 각항목에 대한 설명은 TIBERO 매뉴얼 참고 

--#	OWNER	SEGMENT_NAME	PARTITION_NAME	SEGMENT_TYPE	TABLESPACE_NAME	EXTENT_ID	FILE_ID	BLOCK_ID	BYTES	BLOCKS	RELATIVE_FNO
--1	SYS	_APM_SNAPSHOT	<NULL>	TABLE	_APM_TS	0	3	1393655	131072	16	3  <<< 
--2	SYS	_APM_SQLSTATS	<NULL>	TABLE	_APM_TS	49	3	1159799	1048576	128	3
--3	SYS	_APM_PGASTAT	<NULL>	TABLE	_APM_TS	16	3	1159671	1048576	128	3
--4	SYS	_APM_LATCH	<NULL>	TABLE	_APM_TS	48	3	805255	1048576	128	3
--5	SYS	_APM_UNDOSTAT	<NULL>	TABLE	_APM_TS	5	3	805111	131072	16	3


-- 3. 계산식
select (block_id * 2048) + (blocks * 2048) as BYTE,
	   ((block_id * 2048) + (blocks * 2048)) / 1024 / 1024 / 1024 AS GB 
from dba_extents
where 1=1
and file_id = '3' 
and segment_type = 'TABLE' 
order by block_id*1 desc 

--#	  BYTE	          GB
-- 1 2854238208    2.6582164761.. 

-- → 약 2.6 GB 차지


```

```Shell
# 4. TBS 용량 감축  
[... _tibero ~]$ tbsql

SQL> conn SYS/tibero 

Connected to Tibero.

SQL > alter database datafile '/home/jeus/Tibero/tibero5/database/tibero/apm_ts.dtf' resize 3000M 

# dataFile 경로 : /home/jeus/Tibero/tibero5/database/tibero/apm_ts.dtf 

[... _tibero ~]$ df -h 
# 용량 변화 없음! 

```



## 2. 방법2> TBS DROP/CREATE

```shell
[... _tibero ~]$ tbsql

SQL> conn SYS/tibero 

Connected to Tibero.


# 1. APM 관련 파라미터 확인  
SQL > show parameter APM;

# NAME 							TYPE		VALUE
# ------------------------------------------------------ 
# APM_METRIC			          INT32 	NO 
# APM_SEGMENT_STATISTICS 		  INT32		NO 
# APM_SNAPSHOT_RETENTION 		  INT32		7 
# APM_SNAPSHOT_SAMPLING_INTERVAL  INT32		60 
# APM_SNAPSHOT_TOP_SEGMENT_CNT 	  INT32		5 
# APM_SNAPSHOT_TOP_SQL_CNT 		  INT32		5 
----------------------------------------

# - APM_SNAPSHOT_SAMPLING_INTERVAL : 스냅샷 저장 주기 (60분)
# - APM_SNAPSHOT_RETENTION : 스냅샷 보관 주기 (7일)
# - APM_SNAPSHOT_TOP_SQL_CNT : 리포트에 출력할 상위 SQL갯수
# - APM_SEGMENT_STATISTICS : 'Y'로 설정하면 APM에서 Segment별 Stat 수집 기능을 활성화 (기본값:N)

→ 한 시간에 1번씩 스냅샷을 보관하고 7일 동안 보관하는 것을 알 수 있다. 


# 2. _apm_snapshot 테이블 확인
SQL > SELECT * FROM _APM_SNAPSHOT; -- 5176
# -- 모든 스냅샷은 7일 주기로 지워져야하지만, 2022-07-11 생성된 스냅샷이 그대로 있는 것을 확인
# 해당 테이블을 TRUNCATE 했으나 이미 사용된 테이블스페이스 공간은 반납할 수 없음 



# 3. 테이블 스페이스 삭제
## 삭제 전 /home/jeus/Tibero/tibero5/database/tibero/script 경로의 `apm_ts.dtf` 파일을 카피해 백업

SQL > DROP TABLESPACE _APM_TS INCLUDING CONTENTS AND DATAFILES;

# 4. 테이블 스페이스 생성
SQL > CREATE TABLESPACE _APM_TS DATAFILE '/home/jeus/Tibero/tibero5/database/tibero/apm_ts.dtf'; 

# 5. `apm.sql`, `apm_drop.sql`  실행 
# cd 명령어로 /home/jeus/Tibero/tibero5/database/tibero/script 경로로 이동 

SQL > @apm_drop.sql
SQL > @apm.sql

```

  
# 참고자료

![[Pasted image 20221024172043.png]]
![[Pasted image 20221024172052.png]]
- 제14장 Automatic Performance Monitoring - https://technet.tmaxsoft.com/upload/download/online/tibero/pver-20140808-000002/tibero_admin/apm.html
- DELETE, DROP, TRUNCATE의 비교 - http://www.gurubee.net/article/1455 
- TRUNCATE 를 하면 테이블스페이스가 줄어들까? (아니요) - https://okky.kr/articles/477897
- [Oracle] sqlplus에서 sql스크립트 실행하기 - https://m.blog.naver.com/PostView.naver?isHttpsRedirect=true&blogId=sophie_yeom&logNo=220811401399
