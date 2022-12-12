
# 개요
개발 DB용량 초과 확인 - 08.30  
→ 우선 Tibero Log 파일 삭제조치 


#### 2022.09.27 작업이력 
현재 개발 DB 는 VM에서 구동중인데, 지난 7월 하드디스크 관련 경고 메세지 발생 후 다시 켜지지 않는 사태가 발생했었다. 당시 부팅 메세지를 확인해봤을때, 가용 디스크 용량이 99% 까지 사용되며 `부팅파일을 실행시키지 못해 LOCK 상태에 빠지는 것으로 판단`했다. 
문제는 CLI모드로도 접속이 불가해서.. 웹서버의 경우 비정상 용량초과 발생시 보통 불필요한 로그파일을 삭제하는 것으로 처리했었는데 CLI 모드 접속이 안되니.. 대용량 LOG파일의 추적/삭제가 불가했다.

다행히 개발Server이미지를 반기별로 백업하고 있어 백업 이미지로 복구를 진행했다. 1) VM 이미지를 가장 최근 백업본인 6월로 되돌린 후 2) 형상관리되던 DML들을 모아 3) 전체 테스트를 진행. 다행히 누락되는 부분이 없어, 반나절 안에 개발운영이 가능했고 전체 테스트-안정화 기간은 3일정도 소요했다. 
실은 복원 관련해서 가장 이상적인 방법은 실DB의 TableSpace를 가져오는 것인데, 1) 실DB의 민감정보는 모두 암호화툴로 디코딩되어있고 2) 타사에서 관리하는 TableSpace 는 Select 권한만 주어진 상태이니 카피할 수 없기에 테이블 변경사항 발생시 개발DB도 함께 반영하는 방식으로 운영하고 있다. 3) 무엇보다 실DB 카피본을 만드는데 시간이 너무 많이 소요된다. 또 사용자 트래픽이 집중되는 업무시간에 그런 민감 작업을 수행할 수는 없는 노릇이기도 했다.

복구 후 후속조치는 다음과 같이 진행했다. 
1) 매 주 DB용량 체크 
2) 정확한 원인 파악까지 매월 Backup진행 

#### 2022.10.14 작업이력 
10월 현재 동일증상이 발생해 조치 중. 이번에는 미연에 발견했으니 대용량 파일이 발생한 위치를 직접 따볼예정. 작업 전 스냅샷을 떠 직전 월 VM이미지의 용량과 비교를 진행예정. 아래는 조치 내역 History


#### 2022.10.19 작업이력 
점검결과 TBS용량증가로 판단되어 후속조치 방법 모색. (`3. 점검결과`, `4. 테이블스페이스 조사` 항목)


#### 2022.10.24 작업이력 
undo TBS switch 완료. (`4-1. undo TBS에 관한 조치` 항목)

#### 2022.10.27 작업이력 
apm TBS 재생성 완료. (`### 4-2. APM TBS 에 관한` 항목) 
11월 한달간 테스트 및 재발방지대책 수립 예정 


-----------------------------------
## 1. 에러발생 환경확인
- VM DISK (CentOS), 동적확장저장소 (가상:50GB/실제:22.78GB)  
- 해당 오류사항으로 시스템 구동이 불가하여 빽업본으로 복구한 이력있음
- 금일 작업 사항 : 대용량 log 파일 경로 확인 및 제거 (예정) 
- 이슈사항 : System log 인지 DB EVENT log 인지 확인 필요 


## 2. 작업절차

### 2.1. 전체 DISK 용량확인

du -ah -d2 명령어를 통해 통량이 증가한 path를 추적

```Shell
 [jeus@xx_tibero ~] $ df -h        # Disk Size Check
# Filesystem     Size    Used     Avail   Use$   mounted on
# /dev/mapper/cl-root   47G    26G   22G   54%   /     # 07월
# /dev/mapper/cl-root   47G    30G   18G   64%   /     # 10월 
  ...
 [root@xx_tibero ~] $ cd ..        # Top Path 
 [root@xx_tibero ~] $ pwd
 /

 [root@xx_tibero ~] $ du -sh ./* | sort -hr| head -10 # 현 경로의 모든 파일/디렉토리를 용량순으로 합산해 정렬 (최대 10개) 
#	23G		./home           <<<<< 06월 : 19G (약 4G증가)
#	6.1G	./usr 
#	614M	./root
#	200M	./var
#	150M	./boot
#	30M		./etc
#	6.6M	./run
#	8.0K	./tmp
#	8.0K	./dev   
#	0		./sys
    ...	
 [root@xx_tibero ~] $ du -ah -d2 ./* | sort -hr | head -100 # 현 경로의 모든 파일/디렉토리를 용량순으로 하위디렉터리 2뎁스까지 정렬 (최대 100개)
#	23G		./home/                 <<<<< 06월 : 19G (약 4G증가)
#	22G		./home/jeus/Tibero      <<<<< 06월 12G 
#	22G		./home/jeus/Tibero
#	5.0G	./usr/
#	4.3G	./usr/local
#	2.4G	./usr/local/src
#	2.8G	./home/util
#	2.6G	./usr/local/mysql
#	656M	./usr/share
#	614M	./root
  ...
```

가장 큰 용량을 차지하는 `./home/`  을 확인한다. 이전 OS와 비교했을때 Tibero 설치경로의 용량이 대폭 증가한 것을 확인 

```Shell   
 # 사용메소드 정리 
 ## Directory size summary
 [jeus@xx_tibero ~] $ du -sh ./*/
 ## Size order by Top 1000 (Direcroty) 
 [jeus@xx_tibero ~] $ du -ah -d2 ./*/ | sort -hr | head-1000 # Size order by Top 1000 (Direcroty) 
 ## Size order by Top 1000 (Direc+File)
 [jeus@xx_tibero ~] $ du -ah -d2 ./* | sort-hr | head-1000
 ## Size order by Top 1000 (Full)
 [jeus@xx_tibero ~] $ du -ah -d2 | sort-hr | head-1000
  ...

 # du OPTION 
 #	-a : all
 #	-s : sumarize 
 #	-h : human
 #	-d : (max)-depth=2  (print the total for a dirctory

```


### 2.2. Tibero 로그 용량확인

- Tibero EVENT 가이드 참고
 > TB_EVENT는 $TB_HOME/instnace/log/event 내에 Session ID별로 binary trace log가 남는다. Event마다 고유한 번호를 부여하여 Event 단위로 켜고 끌 수 있다. (파일 형식: [SID]-[TID].btr ) 또한 TBEV(Tibero Event Trace Viewer)를 이용하여 binary 파일을 텍스트로 변환하여 로그를 확인 할 수 있다. 
 
```SQL
 /* Event 로그 생성 경로 확인 */
 select * from vt_parameter 
 where 1=1
 and name like '%EVENT%'
-- and name in ( 'EVENT_TRACE_DEST', 'EVENT_RAECE_TOTAL_SIZE_LIMIT' ) 
 -- 로그파일이 저장되는 경로와 전체 용량을 확인 
 -- PATH : /home/jeus/Tibero/tibero5/instance/log/event 
 -- DLFT_VALUE : 734003200 
```

해당 경로로 접근해 디스크 용량 확인 
```Shell
# /home/jeus/Tibero/tibero5/instance/log/event 
 [jeus@xx_tibero ~] $ pwd 
#  /home/jeus/Tibero/tibero5/instance/log/event
 [jeus@xx_tibero ~] $ du -ah | sort -hr | haed-1000 | 
    ...  
#  412M  .
	...
```

→ 로그파일 자체는 대용량을 차지하지 않음을 확인 (약 500M ~ 1G)


### 2.3. Tibero TBS 용량확인
분석테이블이 생성되었거나, 특정 TBS에 대용량 data가 입력되었을 수도 있는 경우를 점검
→ `./apm_ts.dtf` ,  `./undo 002.dbf` 용량증가 확인. 사용 TS 확인 

```SQL
-- 테이블스페이스 확인
SELECT 
	fl.tablespace_name, df.name, fs.phyrds, fs.phywrts, 
	round((PHYRDS / (SELECT sum(phyrds) FROM v$filestat)) *100, 1) "P_READ(%)", 
	round((PHYWRTS / DECODE((SELECT sum(phywrts) FROM v$filestat), 0, 1, 
							(SELECT sum(phywrts) FROM v$filestat)))*100, 1) "P_WRITE(%)", round((phyrds + phywrts) / (SELECT sum(phyrds) + sum(phywrts) FROM v$filestat) * 100, 1) "TOTAL IO (%)" , 
	round(fs.AVGIOTIM/1000, 3) "AVG_TIME(msec)" 
FROM V$DATAFILE df, V$FILESTAT fs, dba_data_files fl 
WHERE df.file# = fs.file# 
AND df.file# = fl.file_id 
ORDER BY phyrds+phywrts DESC;

-- tablespace_name, path (name으로 표기) 확인
```

```Shell
# TBS datafile 경로에서 크기 확인 
#  /home/jeus/Tibero/tibero5/database/tibero
 [jeus@xx_tibero ~] $ du -ah | sort -hr | haed-1000 |
# 1G    /undo_001.dbf
# 999G  /_APM_TS.dbf
  ...

```

가장 큰 용량을 차지하는 `_APM_TS`  , `UNDO` 을 확인. 
이전 OS와 비교했을때 Tibero 설치경로의 용량이 증가한 것을 확인 


### 3. 점검결과

아래 3가지 항목에 대한 점검을 진행했고 이 중 `DBMS TBS 용량증가` 건으로 파악된다.

 - 전체 DISK 용량
 - DBMS 로그 용량
 - DBMS TBS 용량 <<<

갑작스런 TBS 용량의 증가는 보통 `대용량 데이터 입력`, `분석테이블 생성`, `인스턴스강제종료로 인한 빽업 데이터 생성` 으로 발생하는데, 해당 기간 내에 세 가지 경우가 모두 발생했었다.



## 4. 조치사항 - 테이블 스페이스 용량 조사

> ※ 테이블스페이스 관련 조치전 점검사항 
 >1. AUM(Autometic Undo Managemenet) - APM 
 >     - Oracle 9i undo 관리기능. Tibero의 경우 `APM`으로 대체 ⇒ 사용
 >2. UNDO_EXTENTS 상태확인 
 >     - Expored 수치를 통해 트랜잭션이 할당가능한 공간의 여유가 남아있는지를 확인  ⇒ 부족
 >3. RETENTION 상태확인 
 >     - Tibero의 경우  `show parameter undo` 로 확인  ⇒ 900 초
 >4. UNDO TBS auto-extensible 사용여부 확인 
 >     - TBS공간 부족시 자동확장 여부 확인   ⇒ 사용 

```SQL
-- # 1. AUM 사용여부 확인 
-- Tibero는 오라클 의 방법으로 확인할 수 없음. 단 Tibero 5 부터는 기본으로 사용하도록 되어있으며, 테이블스페이스 조회시 _APM_TS 가 존재하면 사용,


-- # 2. UNDO_EXENTS 상태확인 
SELECT DISTINCT STATUS, SUM(BYTES)/(1024*1024) MB, COUNT(*) FROM DBA_UNDO_EXTENTS GROUP BY STATUS;

-- # STATUS       MB      COUNT(*)  ----------------------
-- 1 Expored      12.98       3   -- undo_retention 시간초과 Extent (트랜젝션 할당가능)
-- 2 Unexpired   992.85     248   -- undo_retention 시간초과x Extent (트랜젝션 할당되지않고 보존된)


-- # 3. RETENTION 상태확인
-- Tibero는 오라클 의 방법으로 확인할 수 없음 (SELECT RETENTION FROM DBA_TABLESPACES;) show parameter undo; 의 UNDO_RETENTION 값 확인 (기본 900초)


-- # 4. TBS auto-extensible 사용여부 확인
-- TBS 물리적위치 및 자동확장 기능 사용여부 확인 
SELECT * FROM DBA_DATA_FILES  -- AUTOEXTENSIBLE : YES 
WHERE 1=1
AND TABLESPACE_NAME = '(TBS명칭)'
```

> ※ 항목별 검토사항 
 >1. AUM(Autometic Undo Managemenet)-APM  : 기본값 사용이므로 사용여부는 변경치 아니할 예정
 >2. UNDO_EXTENTS 상태확인 : 여유공간이 부족한 편이므로 UNDO TBS의 교정이 필요
 >3. RETENTION 상태확인 : 
 >   현재 UNDO 데이타 보유기간은 900초로인데 이는 트랜잭션에 영향을 미치므로 기본값 유지 
 >4. UNDO TBS auto-extensible 사용여부 확인 
 >  : 자동 확장이 불가하도록 설정할 경우.. 처음부터 대용량 공간을 잡아야하므로 UNDO TBS 를 정리하는 의미가 없음. 사용으로 재생성 
 

### 4-1. undo TBS에 관한 조치 
 - `undo` 테이블스페이스는 크기가 늘어나기만 하고 줄어들지 않는 속성을 가지고 있음. 때문에 HDD 공간자원이 적은상태에서는 주기적 관리가 필요함
 - undo TBS 교체, 데이터파일 삭제, 물리파일 삭제 
 - [undo 테이블 스페이스 교체] : https://cassandra.tistory.com/35?category=1011330 
 
### 4-2. APM TBS 에 관한 조치 
 - `_APM_TS` 테이블스페이스의 증가는 `7일이 경과된 스냅샷이 삭제되지 않아 발생` 하는 것으로 예상 (개발서버의 OS Time 은 정시로 고정되어 있는것이 아닌 과거 시간대와 혼용해 사용중임)
 - `_APM_TS` RESIZE 또는 RECREATE 로 처리 예정 
 - https://cassandra.tistory.com/36?category=1011330
 
 