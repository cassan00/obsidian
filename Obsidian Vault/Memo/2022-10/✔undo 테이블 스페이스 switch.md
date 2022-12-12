

# 개념 
## undo & redo

> REDO : 오래된 데이터를 최신 데이터로 만들기 위해 존재 (읽기 일관성을 위해 언두사용)
> UNDO : 최신데이터를 오래된 데이터로 만들기 위해 존재 
>  - ORA-1555 발생시 `undo_retention` 이나 언두 TBS 크기의 튜닝을 검토
>  - 장비에 장애가 발생하거나 인스턴스가 비정상 종료했을떄는, 리두와 언두를 사용해 데이터를 복구하고 커밋하지 않은 데이터의 롤백을 수행

## undo Table 관리
* `Undo Data가 늘어나도 늘어난 데이터파일의 크기를 줄일 수는 없다`. Commit을 수행해도 Undo Segment안에 Undo Data는 지워지지 않고 남아있기 때문.
* Commit 수행 시 다른 서버 프로세스가 덮어 쓸 수 있게 해주는 것일 뿐 Undo Segment 안의 자료를 지우는것은 아니다.
* _Undo Tablespace 크기가 비정상적으로 클 경우에는 관리자가 다른 작은 Undo Tablespace를 신규로 만들고, Undo Tablespace를 신규 Undo Tablespace로 변경시킨 후 기존 Undo Tablespace를 삭제해 주어야한다._

이하 작업내용 기술 



----------------------
# 작업내용
## 1. TBS정보(이름, 데이터파일의 경로) 확인
 - tbAdmin을 통해서는 확인할 수 없어 콘솔접근 필요

```shell
[... _tibero ~]$ tbsql

SQL> conn SYS/tibero 

Connected to Tibero.

# 1) 테이블 스페이스 이름 확인 
SQL > show parameter undo;

# NAME 							TYPE			VALUE
# ------------------------------------------------------ 
# UNDO_RETENTION			INT32 		900
# UNDO_TABLESPACE 			STRING		UNDO 
    
# → 현재 테이블 스페이스의 이름은 `UNDO`  
# → 현재 UNDO 데이타 보유기간은 900초   



# 2) 테이블 스페이스 상태 확인 
SQL > SELECT 
				TS_ID, DATAFILE_COUNT, TABLESPACE_NAME, STATUS, 
				CONTENTS --, EXTENT_MANAGEMENT, SEGMENT_SPACE_MANAGEMENT 
			FROM 
			   DBA_TABLESPACES
			WHERE 1=1
			AND TABLESPACE_NAME = 'UNDO';
			
#	TS_ID		DATAFILE_COUNT	TABLESPACE_NAME	STATUS	CONTENTS	 
#	1		1	UNDO	ONLINE		UNDO	 

# → 현재 테이블 스페이스는 `ONLINE` 상태 (데이터 테이블스페이스라면 핫백업 필요)



# 3) 테이블 스페이스 용량 확인 
SQL > SELECT DISTINCT 
				STATUS, SUM(BYTES)/(1024*1024) MB, COUNT(*) 
			 FROM DBA_UNDO_EXTENTS 
			 GROUP BY STATUS;

#	STATUS		   MB				COUNT(*)
#	Expired		  11.984375			    3
#	Unexpired	 991.8515625	       248



# 4) 테이블 스페이스 데이터파일의 경로 확인 
SQL > SELECT FILE_NAME,	FILE_ID,	TABLESPACE_NAME,	BYTES
			FROM DBA_DATA_FILES 
			WHERE 1=1
			AND TABLESPACE_NAME = 'UNDO';
			
# FILE_NAME		FILE_ID		TABLESPACE_NAME	BYTES	 			
# ------------------------------------------------------ 
#	/home/jeus/Tibero/tibero5/database/tibero/undo_001.dbf	1	UNDO	1048576000	

```
 - 테이블스페이스 이름/용량/상태 : UNDO / 1G / ONLINE 
 - 데이터파일경로 : /home/jeus/Tibero/tibero5/database/tibero/undo_001.dbf
 
 

## 2. 신규 TBS 생성

```shell
# 1) 신규 TBS 생성 
SQL> create undo tablespace UNDO_02
	  datafile '/home/jeus/Tibero/tibero5/database/tibero/undo_002.dbf' size 100M
	  autoextend on;
			
Tablespace 'UNDO_02' created.


# 2) 생성된 TBS 의 이름 및 데이터파일 경로 확인 
SQL > SELECT * FROM DBA_TABLESPACES 
			WHERE 1=1
			AND TABLESPACE_NAME = 'UNDO_02';
			
SQL > SELECT FILE_NAME,	FILE_ID,	TABLESPACE_NAME,	BYTES
			FROM DBA_DATA_FILES 
			WHERE 1=1
			AND TABLESPACE_NAME = 'UNDO_02';
			
# FILE_NAME		FILE_ID		TABLESPACE_NAME	BYTES	 			
# ------------------------------------------------------ 
#	/home/jeus/Tibero/tibero5/database/tibero/undo_002.dbf	1	UNDO_02	   -

			
```

## 3. 교체, 변경내역 확인 

```shell
SQL > alter system set undo_tablespace=UNDO_02;

System altered.

SQL > show parameter undo;

# NAME 										TYPE			VALUE
# ------------------------------------------------------ 
# UNDO_RETENTION				INT32 		900
# UNDO_TABLESPACE 			STRING		UNDO 

SQL > quit();

```

## 4. DBMS 재기동
```shell
[... _tibero ~]$ ps -ef|grep tb  # 프로세스 확인
[... _tibero ~]$ tbdown 
  Select action : 1, 2 (Wait, Immediately)
  
  Tibero instance terminated.
  
[... _tibero ~]$ ps -ef|grep tb  # 프로세스 종료 확인  
[... _tibero ~]$ tbboot

  Tibero instance started up.
    
[... _tibero ~]$ tbsql

SQL> conn SYS/tibero 

Connected to Tibero.

SQL > show parameter undo;  

# NAME 										TYPE			VALUE
# ------------------------------------------------------ 
# UNDO_RETENTION				INT32 		900
# UNDO_TABLESPACE 			STRING		UNDO_02 		<<< 변경확인 완료

```

## 5. 이전 UNDO TBS 삭제 

```shell

# 전체 테이블 스페이스 확인 
SQL > SELECT 
				TS_ID, DATAFILE_COUNT, TABLESPACE_NAME, STATUS, 
				CONTENTS --, EXTENT_MANAGEMENT, SEGMENT_SPACE_MANAGEMENT 
			FROM 
			   DBA_TABLESPACES
			WHERE 1=1
			
# 삭제
SQL > drop tablespace UNDO;

	Tablespace 'UNDO' dropped.
	
```


※ 단, DBMS 논리 단위에서 `UNDO` TBS를 삭제한 것이므로 경로상 남아있는 데이터파일은 직접삭제가 필요함
 - 경로 : /home/jeus/Tibero/tibero5/database/tibero/undo_001.dbf	



## 참고자료
- Undo tablespace full로 인한 서비스 실패를 막기 위한 시나리오  - https://in0de.tistory.com/75
-  undo_retention 이슈 및 작업 정리 https://sksggg123.github.io/db/undo-retention/
* 언두와 리두 : http://dbcafe.co.kr/wiki/index.php/UNDO_REDO
*  UNDO tablespace 사용량 급증 현상 해결 : https://jckim-dev.tistory.com/m/entry/oracle-%EC%84%A4%EC%A0%95-UNDO-tablespace-%EC%82%AC%EC%9A%A9%EB%9F%89-%EA%B8%89%EC%A6%9D-%ED%98%84%EC%83%81-%ED%95%B4%EA%B2%B0 
* undoTable 관리하기(오라클) : https://hohsworks.tistory.com/m/entry/7Tablespace%EC%99%80-Datafile-%EA%B4%80%EB%A6%AC%ED%95%98%EA%B8%B0-Undo-Tablespace 
