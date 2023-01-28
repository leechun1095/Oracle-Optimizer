## Logical Optimizer(Transformer)란 무엇인가 ?
* 오라클 Logical Optimizer(Transformer)란 옵티마이져의 가장 중요한 구성요소이며 성능향상을 위해 SQL을 재작성(Query Transformation)하는 역할을 담당
* 옵티마이져가 성능향상을 목적으로 SQL 재작성 하는 것

## Optimizer Components
* Query Transformer - SQL변신
* Cost Estimator - 통계정보 등을 참조하여 가장 낮은 Cost(비용)을 갖는 SQL 찾기
* Plan Generator - SQL 실행 계획 생성

## Query Transformation 의 개념
* Query Transformation 은 Transformer 의 기능이며 '개발자가 작성한 SQL을 옵티마이져가 다른 모습으로 재작성하는 것'

## 쿼리블록은 무엇인가 ?
* 옵티마이져가 최적화를 수행하는 단위로 최적의 액세스 경로와 조인순서, 조인방식을 선택하는것
* 2개의 쿼리블록 = 서브쿼리와 메인쿼리

## DBMS_XPLAN.DISPLAY_CURSOR
: 실행계획을 볼 수 있는 Tool
1. 일반적으로 10046 Event Trace + Tkprof
2. DBMS_XPLAN.DISPLAY_CURSOR 혹은 DBMS_XPLAN.DISPLAY ( Query Transformation 장점( 10046 에 비해 )
* QUERY BLOCK NAME / QUERY ALIAS : 쿼리블럭정보
* Outline Data : 오라클 내부(Internal) Hint
* Predicate Information : Access 조건 및 조인 조건, Filter 조건
* Column Projection Information : Operation id 별로 Select 된 컬럼 정보
* Format : 자신에게 맞는 Format 설정이 자유로움

```sql
SELECT /*+ GATHER_PLAN_STATISTICS */
       *
  FROM (
        SELECT E.*
		  FROM EMPLOYEE E
		 WHERE E.DEPARTMENT_ID = 50
		 ORDER BY E.EMPLOYEE_ID
  )
 WHERE ROWNUM <= 100;

-- Format : advanced allstats last (가장 많은 정보를 보여주는 포맷)
SELECT *
  FROM TABLE (DBMS_XPLAN.DISPLAY_CURSOR (NULL, NULL, 'advanced allstats last'));
```
```sql
-- PLAN_TABLE_OUTPUT
-----------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                     | Name          | Starts | E-Rows |E-Bytes| Cost (%CPU)| E-Time   | A-Rows |   A-Time   | Buffers |
-----------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT              |               |      1 |        |       |     3 (100)|          |     45 |00:00:00.01 |       5 |
|*  1 |  COUNT STOPKEY                |               |      1 |        |       |            |          |     45 |00:00:00.01 |       5 |
|   2 |   VIEW                        |               |      1 |     45 |  5985 |     3   (0)| 00:00:01 |     45 |00:00:00.01 |       5 |
|*  3 |    TABLE ACCESS BY INDEX ROWID| EMPLOYEE      |      1 |     45 |  3105 |     3   (0)| 00:00:01 |     45 |00:00:00.01 |       5 |
|   4 |     INDEX FULL SCAN           | EMP_EMP_ID_PK |      1 |    107 |       |     1   (0)| 00:00:01 |    107 |00:00:00.01 |       2 |
-----------------------------------------------------------------------------------------------------------------------------------------
 
-- 쿼리블럭 정보
Query Block Name / Object Alias (identified by operation id):
-------------------------------------------------------------
 
   1 - SEL$1
   2 - SEL$2 / from$_subquery$_001@SEL$1
   3 - SEL$2 / E@SEL$2
   4 - SEL$2 / E@SEL$2
 
Outline Data -- 오라클이 내부적으로 사용한 힌트를 나타냄
-------------
 
  /*+
      BEGIN_OUTLINE_DATA
      IGNORE_OPTIM_EMBEDDED_HINTS
      OPTIMIZER_FEATURES_ENABLE('11.2.0.1')
      DB_VERSION('11.2.0.1')
      ALL_ROWS
      OUTLINE_LEAF(@"SEL$2")
      OUTLINE_LEAF(@"SEL$1")
      NO_ACCESS(@"SEL$1" "from$_subquery$_001"@"SEL$1")
      INDEX(@"SEL$2" "E"@"SEL$2" ("EMPLOYEE"."EMPLOYEE_ID"))
      END_OUTLINE_DATA
  */
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   1 - filter(ROWNUM<=100)
   3 - filter("E"."DEPARTMENT_ID"=50)
 
Column Projection Information (identified by operation id):  -- ID 별 SELECT 되는 컬럼의 정보를 나타냄
-----------------------------------------------------------
 
   1 - "from$_subquery$_001"."EMPLOYEE_ID"[NUMBER,22], "from$_subquery$_001"."FIRST_NAME"[VARCHAR2,20], 
       "from$_subquery$_001"."LAST_NAME"[VARCHAR2,25], "from$_subquery$_001"."EMAIL"[VARCHAR2,25], 
       "from$_subquery$_001"."PHONE_NUMBER"[VARCHAR2,20], "from$_subquery$_001"."HIRE_DATE"[DATE,7], 
       "from$_subquery$_001"."JOB_ID"[VARCHAR2,10], "from$_subquery$_001"."SALARY"[NUMBER,22], 
       "from$_subquery$_001"."COMMISSION_PCT"[NUMBER,22], "from$_subquery$_001"."MANAGER_ID"[NUMBER,22], 
       "from$_subquery$_001"."DEPARTMENT_ID"[NUMBER,22]
   2 - "from$_subquery$_001"."EMPLOYEE_ID"[NUMBER,22], "from$_subquery$_001"."FIRST_NAME"[VARCHAR2,20], 
       "from$_subquery$_001"."LAST_NAME"[VARCHAR2,25], "from$_subquery$_001"."EMAIL"[VARCHAR2,25], 
       "from$_subquery$_001"."PHONE_NUMBER"[VARCHAR2,20], "from$_subquery$_001"."HIRE_DATE"[DATE,7], 
       "from$_subquery$_001"."JOB_ID"[VARCHAR2,10], "from$_subquery$_001"."SALARY"[NUMBER,22], 
       "from$_subquery$_001"."COMMISSION_PCT"[NUMBER,22], "from$_subquery$_001"."MANAGER_ID"[NUMBER,22], 
       "from$_subquery$_001"."DEPARTMENT_ID"[NUMBER,22]
   3 - "E"."EMPLOYEE_ID"[NUMBER,22], "E"."FIRST_NAME"[VARCHAR2,20], "E"."LAST_NAME"[VARCHAR2,25], "E"."EMAIL"[VARCHAR2,25], 
       "E"."PHONE_NUMBER"[VARCHAR2,20], "E"."HIRE_DATE"[DATE,7], "E"."JOB_ID"[VARCHAR2,10], "E"."SALARY"[NUMBER,22], 
       "E"."COMMISSION_PCT"[NUMBER,22], "E"."MANAGER_ID"[NUMBER,22], "E"."DEPARTMENT_ID"[NUMBER,22]
   4 - "E".ROWID[ROWID,10], "E"."EMPLOYEE_ID"[NUMBER,22]
```

```sql
가. BASIC 항목
 - Id : 각 Operation 의 ID, *가 달려 있는 Predicated Information 에 access 및 Filter 정보
 - Operation :각각 실행되는 JOB
 - Name :Operation 이 액세스하는 테이블 및 인덱스

나. Query Optimizer Estimations 항목(옵티마이져 예상치)
 - E-Rows : 각 Operation 이 끝났을 때 return 되는 건수(예상치)
 - E-Bytes : 각 Operation 이 return 한 bytes 수 (예상치)
 - E-Temp : 각 Operation 이 Temporary Space 를 사용한 양 ( 예상치, 샘플엔 없음 )
 - Cost(%CPU) : 각 Operation 의 Cost (예상치)
 - E-Time : 수행시간(예상치)

다. Runtime Statistics 항목(실제 수행시간 및 실제 수행건수)
 - Starts : 각 Operation 을 반복 수행한 건수(NL 이라면 조인 시도 횟수)
 - A-Rows :각 Operation 이 Return 한 건수
 - A-Time : 실제 실행시간 0.01초 까지, Child Operation 의 A-Time 을 합친 누적

라. I/O Statistics (I/O관련 READ / WRITE 한 Block 수)
 - Buffers : 각 Operation 이 Memory 에서 읽은 Block 수
 - Reads : 각 Operation 이 Disk 에서 Read 한 Block 수. (샘플엔 없음)
 - Writes : 각 Operation 이 Disk 에 Write Block 수 ( 샘플엔 없음)

마. Memory Utilization Statistics ( Hash 작업이나 sort 작업 시 사용한 메모리 통계)
 - oMem : Optimal Execution 에 필요한 Memory ( 예상치 임)
 - 1Mem : One-pass Execution 에 필요한 Memory ( 예상치 임)
 - O/1/M : 각 Operation 이 실행한 Optimal/One-Pass/Multipass 횟수가 순서대로 표시됨
 - Used-Mem : 마직 실행 시 사용한 PGA Memory
 - Used-Tmp : 마지막 실행 시 메모리가 부족하여 Temporary space를 사용할 때 나타남,1024 곱해야 함.
 - Max-Tmp : 메모리가 부족하여 Temporary Space 를 사용할 때 최대 Temp 사용량임, 마지막이 아닌 최대값만 보임, 1024 곱해야 함
```

## Format : allstats last -rows +predicate 
쿼리 변형이 필요없는 경우 단순화하게 조회하는 포맷
```sql
SELECT *
  FROM TABLE (DBMS_XPLAN.DISPLAY_CURSOR (NULL, NULL, 'allstats last -rows +predicate'));
```
```sql
PLAN_TABLE_OUTPUT
------------------------------------------------------------------------------------------------
| Id  | Operation                     | Name          | Starts | A-Rows |   A-Time   | Buffers |
------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT              |               |      1 |     45 |00:00:00.01 |       5 |
|*  1 |  COUNT STOPKEY                |               |      1 |     45 |00:00:00.01 |       5 |
|   2 |   VIEW                        |               |      1 |     45 |00:00:00.01 |       5 |
|*  3 |    TABLE ACCESS BY INDEX ROWID| EMPLOYEE      |      1 |     45 |00:00:00.01 |       5 |
|   4 |     INDEX FULL SCAN           | EMP_EMP_ID_PK |      1 |    107 |00:00:00.01 |       2 |
------------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   1 - filter(ROWNUM<=100)
   3 - filter("E"."DEPARTMENT_ID"=50)
 
-- Format : allstats last -rows +alias +outline +predicate (쿼리변형이 발생하거나 복잡한 쿼리 튜닝시 쿼리블럭과 힌트정보를 추가로 출력하는 포맷)
SELECT *
  FROM TABLE (DBMS_XPLAN.DISPLAY_CURSOR (NULL, NULL, 'allstats last -rows +alias +outline +predicate')); 
```

## Hint가 무시되는 이유
```sql
CREATE INDEX loc_postal_idx ON location (postal_code);
CREATE INDEX dept_name_idx ON department (department_name);
CREATE INDEX coun_region_idx ON country (region_id);

CREATE OR REPLACE VIEW v_dept AS
SELECT d.department_id, d.department_name, d.manager_id, l.location_id,
       l.postal_code, l.city, c.country_id, c.country_name, c.region_id
  FROM department d, location l, country c
 WHERE d.location_id = l.location_id 
   AND l.country_id = c.country_id;

   
SELECT /*+ GATHER_PLAN_STATISTICS  */
       e.employee_id, e.first_name, e.last_name, e.job_id, v.department_name
  FROM employee e, v_dept v
 WHERE e.department_id = v.department_id
   AND v.department_name = 'Shipping'
   AND v.postal_code = '99236'
   AND v.region_id = 2
   AND e.job_id = 'ST_CLERK';

SELECT * FROM 
TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL, 'allstats last -rows +alias +outline +predicate'));
```
```sql
PLAN_TABLE_OUTPUT 
----------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                        | Name              | Starts | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
----------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                 |                   |      1 |     20 |00:00:00.01 |      13 |       |       |          |
|   1 |  NESTED LOOPS                    |                   |      1 |     20 |00:00:00.01 |      13 |       |       |          |
|   2 |   NESTED LOOPS                   |                   |      1 |     45 |00:00:00.01 |      10 |       |       |          |
|   3 |    NESTED LOOPS                  |                   |      1 |      1 |00:00:00.01 |       8 |       |       |          |
|   4 |     MERGE JOIN CARTESIAN         |                   |      1 |      5 |00:00:00.01 |       4 |       |       |          |
|*  5 |      INDEX RANGE SCAN            | COUN_REGION_IDX   |      1 |      5 |00:00:00.01 |       2 |       |       |          |
|   6 |      BUFFER SORT                 |                   |      5 |      5 |00:00:00.01 |       2 |  2048 |  2048 | 2048  (0)|
|   7 |       TABLE ACCESS BY INDEX ROWID| DEPARTMENT        |      1 |      1 |00:00:00.01 |       2 |       |       |          |
|*  8 |        INDEX RANGE SCAN          | DEPT_NAME_IDX     |      1 |      1 |00:00:00.01 |       1 |       |       |          |
|*  9 |     TABLE ACCESS BY INDEX ROWID  | LOCATION          |      5 |      1 |00:00:00.01 |       4 |       |       |          |
|* 10 |      INDEX RANGE SCAN            | LOC_POSTAL_IDX    |      5 |      5 |00:00:00.01 |       3 |       |       |          |
|* 11 |    INDEX RANGE SCAN              | EMP_DEPARTMENT_IX |      1 |     45 |00:00:00.01 |       2 |       |       |          |
|* 12 |   TABLE ACCESS BY INDEX ROWID    | EMPLOYEE          |     45 |     20 |00:00:00.01 |       3 |       |       |          |
----------------------------------------------------------------------------------------------------------------------------------
 
Query Block Name / Object Alias (identified by operation id):
-------------------------------------------------------------
 
   1 - SEL$F5BB74E1
   5 - SEL$F5BB74E1 / C@SEL$2
   7 - SEL$F5BB74E1 / D@SEL$2
   8 - SEL$F5BB74E1 / D@SEL$2
   9 - SEL$F5BB74E1 / L@SEL$2
  10 - SEL$F5BB74E1 / L@SEL$2
  11 - SEL$F5BB74E1 / E@SEL$1
  12 - SEL$F5BB74E1 / E@SEL$1
 
Outline Data
-------------
 
  /*+
      BEGIN_OUTLINE_DATA
      IGNORE_OPTIM_EMBEDDED_HINTS
      OPTIMIZER_FEATURES_ENABLE('11.2.0.1')
      DB_VERSION('11.2.0.1')
      ALL_ROWS
      OUTLINE_LEAF(@"SEL$F5BB74E1")
      MERGE(@"SEL$2")
      OUTLINE(@"SEL$1")
      OUTLINE(@"SEL$2")
      INDEX(@"SEL$F5BB74E1" "C"@"SEL$2" ("COUNTRY"."REGION_ID"))
      INDEX_RS_ASC(@"SEL$F5BB74E1" "D"@"SEL$2" ("DEPARTMENT"."DEPARTMENT_NAME"))
      INDEX_RS_ASC(@"SEL$F5BB74E1" "L"@"SEL$2" ("LOCATION"."POSTAL_CODE"))
      INDEX(@"SEL$F5BB74E1" "E"@"SEL$1" ("EMPLOYEE"."DEPARTMENT_ID"))
      LEADING(@"SEL$F5BB74E1" "C"@"SEL$2" "D"@"SEL$2" "L"@"SEL$2" "E"@"SEL$1")
      USE_MERGE_CARTESIAN(@"SEL$F5BB74E1" "D"@"SEL$2")
      USE_NL(@"SEL$F5BB74E1" "L"@"SEL$2")
      USE_NL(@"SEL$F5BB74E1" "E"@"SEL$1")
      NLJ_BATCHING(@"SEL$F5BB74E1" "E"@"SEL$1")
      END_OUTLINE_DATA
  */
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   5 - access("C"."REGION_ID"=2)
   8 - access("D"."DEPARTMENT_NAME"='Shipping')
   9 - filter(("D"."LOCATION_ID"="L"."LOCATION_ID" AND "L"."COUNTRY_ID"="C"."COUNTRY_ID"))
  10 - access("L"."POSTAL_CODE"='99236')
  11 - access("E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID")
  12 - filter("E"."JOB_ID"='ST_CLERK')
```

## 조인순서를 바꿀수 있겠는가 ? V_DEPT => EMPLOYEES 에서 EMPLOYEE => V_DEPT
* QT가 발생했기 때문에 아래와 같이 hint를 사용해도 변경되지 않는다.

```sql
SELECT /*+ GATHER_PLAN_STATISTICS LEADING(E V) */
       e.employee_id, e.first_name, e.last_name, e.job_id, v.department_name
  FROM employee e, v_dept v
 WHERE e.department_id = v.department_id
   AND v.department_name = 'Shipping'
   AND v.postal_code = '99236'
   AND v.region_id = 2
   AND e.job_id = 'ST_CLERK';

SELECT * FROM 
TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL, 'allstats last -rows +alias +outline +predicate'));
```

```sql
PLAN_TABLE_OUTPUT
-------------------------------------
----------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                        | Name              | Starts | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
----------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                 |                   |      1 |     20 |00:00:00.01 |      13 |       |       |          |
|   1 |  NESTED LOOPS                    |                   |      1 |     20 |00:00:00.01 |      13 |       |       |          |
|   2 |   NESTED LOOPS                   |                   |      1 |     45 |00:00:00.01 |      10 |       |       |          |
|   3 |    NESTED LOOPS                  |                   |      1 |      1 |00:00:00.01 |       8 |       |       |          |
|   4 |     MERGE JOIN CARTESIAN         |                   |      1 |      5 |00:00:00.01 |       4 |       |       |          |
|*  5 |      INDEX RANGE SCAN            | COUN_REGION_IDX   |      1 |      5 |00:00:00.01 |       2 |       |       |          |
|   6 |      BUFFER SORT                 |                   |      5 |      5 |00:00:00.01 |       2 |  2048 |  2048 | 2048  (0)|
|   7 |       TABLE ACCESS BY INDEX ROWID| DEPARTMENT        |      1 |      1 |00:00:00.01 |       2 |       |       |          |
|*  8 |        INDEX RANGE SCAN          | DEPT_NAME_IDX     |      1 |      1 |00:00:00.01 |       1 |       |       |          |
|*  9 |     TABLE ACCESS BY INDEX ROWID  | LOCATION          |      5 |      1 |00:00:00.01 |       4 |       |       |          |
|* 10 |      INDEX RANGE SCAN            | LOC_POSTAL_IDX    |      5 |      5 |00:00:00.01 |       3 |       |       |          |
|* 11 |    INDEX RANGE SCAN              | EMP_DEPARTMENT_IX |      1 |     45 |00:00:00.01 |       2 |       |       |          |
|* 12 |   TABLE ACCESS BY INDEX ROWID    | EMPLOYEE          |     45 |     20 |00:00:00.01 |       3 |       |       |          |
----------------------------------------------------------------------------------------------------------------------------------
 
Query Block Name / Object Alias (identified by operation id):
-------------------------------------------------------------
 
   1 - SEL$F5BB74E1
   5 - SEL$F5BB74E1 / C@SEL$2
   7 - SEL$F5BB74E1 / D@SEL$2
   8 - SEL$F5BB74E1 / D@SEL$2
   9 - SEL$F5BB74E1 / L@SEL$2
  10 - SEL$F5BB74E1 / L@SEL$2
  11 - SEL$F5BB74E1 / E@SEL$1
  12 - SEL$F5BB74E1 / E@SEL$1
 
Outline Data
-------------
 
  /*+
      BEGIN_OUTLINE_DATA
      IGNORE_OPTIM_EMBEDDED_HINTS
      OPTIMIZER_FEATURES_ENABLE('11.2.0.1')
      DB_VERSION('11.2.0.1')
      ALL_ROWS
      OUTLINE_LEAF(@"SEL$F5BB74E1")
      MERGE(@"SEL$2")
      OUTLINE(@"SEL$1")
      OUTLINE(@"SEL$2")
      INDEX(@"SEL$F5BB74E1" "C"@"SEL$2" ("COUNTRY"."REGION_ID"))
      INDEX_RS_ASC(@"SEL$F5BB74E1" "D"@"SEL$2" ("DEPARTMENT"."DEPARTMENT_NAME"))
      INDEX_RS_ASC(@"SEL$F5BB74E1" "L"@"SEL$2" ("LOCATION"."POSTAL_CODE"))
      INDEX(@"SEL$F5BB74E1" "E"@"SEL$1" ("EMPLOYEE"."DEPARTMENT_ID"))
      LEADING(@"SEL$F5BB74E1" "C"@"SEL$2" "D"@"SEL$2" "L"@"SEL$2" "E"@"SEL$1")
      USE_MERGE_CARTESIAN(@"SEL$F5BB74E1" "D"@"SEL$2")
      USE_NL(@"SEL$F5BB74E1" "L"@"SEL$2")
      USE_NL(@"SEL$F5BB74E1" "E"@"SEL$1")
      NLJ_BATCHING(@"SEL$F5BB74E1" "E"@"SEL$1")
      END_OUTLINE_DATA
  */
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   5 - access("C"."REGION_ID"=2)
   8 - access("D"."DEPARTMENT_NAME"='Shipping')
   9 - filter(("D"."LOCATION_ID"="L"."LOCATION_ID" AND "L"."COUNTRY_ID"="C"."COUNTRY_ID"))
  10 - access("L"."POSTAL_CODE"='99236')
  11 - access("E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID")
  12 - filter("E"."JOB_ID"='ST_CLERK')
```

#### 해결방법
* Query Transformation 에 의해 변경된 쿼리블럭명을 지정하여 힌트를 사용하거나
* Query Transformation 이 발생하지 않게 하면 힌트가 정상작동

```sql
-- NO_MERGE(v)
-- v_dept 뷰테이블의 MERGE가 안되게 하거나 쿼리블록의 Alias를 사용해야 조인순서가 변경된다
SELECT /*+ GATHER_PLAN_STATISTICS NO_MERGE(V) LEADING(E V) */
       e.employee_id, e.first_name, e.last_name, e.job_id, v.department_name
  FROM employee e, v_dept v
 WHERE e.department_id = v.department_id
   AND v.department_name = 'Shipping'
   AND v.postal_code = '99236'
   AND v.region_id = 2
   AND e.job_id = 'ST_CLERK';


SELECT * FROM 
TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL, 'allstats last -rows +alias +outline +predicate'));
```

```sql
PLAN_TABLE_OUTPUT
---------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                         | Name            | Starts | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
---------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                  |                 |      1 |     20 |00:00:00.01 |      10 |       |       |          |
|*  1 |  HASH JOIN                        |                 |      1 |     20 |00:00:00.01 |      10 |   915K|   915K|  409K (0)|
|   2 |   TABLE ACCESS BY INDEX ROWID     | EMPLOYEE        |      1 |     20 |00:00:00.01 |       2 |       |       |          |
|*  3 |    INDEX RANGE SCAN               | EMP_JOB_IX      |      1 |     20 |00:00:00.01 |       1 |       |       |          |
|   4 |   VIEW                            | V_DEPT          |      1 |      1 |00:00:00.01 |       8 |       |       |          |
|   5 |    NESTED LOOPS                   |                 |      1 |      1 |00:00:00.01 |       8 |       |       |          |
|   6 |     NESTED LOOPS                  |                 |      1 |      5 |00:00:00.01 |       7 |       |       |          |
|   7 |      MERGE JOIN CARTESIAN         |                 |      1 |      5 |00:00:00.01 |       4 |       |       |          |
|*  8 |       INDEX RANGE SCAN            | COUN_REGION_IDX |      1 |      5 |00:00:00.01 |       2 |       |       |          |
|   9 |       BUFFER SORT                 |                 |      5 |      5 |00:00:00.01 |       2 |  2048 |  2048 | 2048  (0)|
|  10 |        TABLE ACCESS BY INDEX ROWID| DEPARTMENT      |      1 |      1 |00:00:00.01 |       2 |       |       |          |
|* 11 |         INDEX RANGE SCAN          | DEPT_NAME_IDX   |      1 |      1 |00:00:00.01 |       1 |       |       |          |
|* 12 |      INDEX RANGE SCAN             | LOC_POSTAL_IDX  |      5 |      5 |00:00:00.01 |       3 |       |       |          |
|* 13 |     TABLE ACCESS BY INDEX ROWID   | LOCATION        |      5 |      1 |00:00:00.01 |       1 |       |       |          |
---------------------------------------------------------------------------------------------------------------------------------
 
-- SEL$1 : 쿼리블럭1 (employee)
-- SEL$2 : 쿼리블럭2 (뷰테이블)
Query Block Name / Object Alias (identified by operation id):
-------------------------------------------------------------
 
   1 - SEL$1
   2 - SEL$1 / E@SEL$1
   3 - SEL$1 / E@SEL$1
   4 - SEL$2 / V@SEL$1
   5 - SEL$2
   8 - SEL$2 / C@SEL$2
  10 - SEL$2 / D@SEL$2
  11 - SEL$2 / D@SEL$2
  12 - SEL$2 / L@SEL$2
  13 - SEL$2 / L@SEL$2
 
Outline Data
-------------
 
  /*+
      BEGIN_OUTLINE_DATA
      IGNORE_OPTIM_EMBEDDED_HINTS
      OPTIMIZER_FEATURES_ENABLE('11.2.0.1')
      DB_VERSION('11.2.0.1')
      ALL_ROWS
      OUTLINE_LEAF(@"SEL$2")
      OUTLINE_LEAF(@"SEL$1")
      INDEX_RS_ASC(@"SEL$1" "E"@"SEL$1" ("EMPLOYEE"."JOB_ID"))
      NO_ACCESS(@"SEL$1" "V"@"SEL$1")
      LEADING(@"SEL$1" "E"@"SEL$1" "V"@"SEL$1")
      USE_HASH(@"SEL$1" "V"@"SEL$1")
      INDEX(@"SEL$2" "C"@"SEL$2" ("COUNTRY"."REGION_ID"))
      INDEX_RS_ASC(@"SEL$2" "D"@"SEL$2" ("DEPARTMENT"."DEPARTMENT_NAME"))
      INDEX(@"SEL$2" "L"@"SEL$2" ("LOCATION"."POSTAL_CODE"))
      LEADING(@"SEL$2" "C"@"SEL$2" "D"@"SEL$2" "L"@"SEL$2")
      USE_MERGE_CARTESIAN(@"SEL$2" "D"@"SEL$2")
      USE_NL(@"SEL$2" "L"@"SEL$2")
      NLJ_BATCHING(@"SEL$2" "L"@"SEL$2")
      END_OUTLINE_DATA
  */
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   1 - access("E"."DEPARTMENT_ID"="V"."DEPARTMENT_ID")
   3 - access("E"."JOB_ID"='ST_CLERK')
   8 - access("C"."REGION_ID"=2)
  11 - access("D"."DEPARTMENT_NAME"='Shipping')
  12 - access("L"."POSTAL_CODE"='99236')
  13 - filter(("D"."LOCATION_ID"="L"."LOCATION_ID" AND "L"."COUNTRY_ID"="C"."COUNTRY_ID"))
 
 
-- 뷰테이블의 조인순서를 변경하기 (country -> location -> department)
-- 뷰테이블의 글로벌 hint를 사용한다.
-- 이 경우는 쿼리 블록이 2개이기 때문에 LEADING hint를 2번 사용할 수 있다.
/*
CREATE OR REPLACE VIEW v_dept AS
SELECT d.department_id, d.department_name, d.manager_id, l.location_id,
       l.postal_code, l.city, c.country_id, c.country_name, c.region_id
  FROM department d, location l, country c
 WHERE d.location_id = l.location_id 
   AND l.country_id = c.country_id;
*/
SELECT /*+ GATHER_PLAN_STATISTICS NO_MERGE(V) LEADING(E V) LEADING(V.C V.L V.D) */
       e.employee_id, e.first_name, e.last_name, e.job_id, v.department_name
  FROM employee e, v_dept v
 WHERE e.department_id = v.department_id
   AND v.department_name = 'Shipping'
   AND v.postal_code = '99236'
   AND v.region_id = 2
   AND e.job_id = 'ST_CLERK';

SELECT * FROM 
TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL, 'allstats last -rows +alias +outline +predicate'));

PLAN_TABLE_OUTPUT
---------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                        | Name             | Starts | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
---------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                 |                  |      1 |     20 |00:00:00.01 |      12 |       |       |          |
|*  1 |  HASH JOIN                       |                  |      1 |     20 |00:00:00.01 |      12 |   915K|   915K|  424K (0)|
|   2 |   TABLE ACCESS BY INDEX ROWID    | EMPLOYEE         |      1 |     20 |00:00:00.01 |       2 |       |       |          |
|*  3 |    INDEX RANGE SCAN              | EMP_JOB_IX       |      1 |     20 |00:00:00.01 |       1 |       |       |          |
|   4 |   VIEW                           | V_DEPT           |      1 |      1 |00:00:00.01 |      10 |       |       |          |
|   5 |    NESTED LOOPS                  |                  |      1 |      1 |00:00:00.01 |      10 |       |       |          |
|   6 |     NESTED LOOPS                 |                  |      1 |      1 |00:00:00.01 |       9 |       |       |          |
|   7 |      NESTED LOOPS                |                  |      1 |      1 |00:00:00.01 |       7 |       |       |          |
|*  8 |       INDEX RANGE SCAN           | COUN_REGION_IDX  |      1 |      5 |00:00:00.01 |       2 |       |       |          |
|*  9 |       TABLE ACCESS BY INDEX ROWID| LOCATION         |      5 |      1 |00:00:00.01 |       5 |       |       |          |
|* 10 |        INDEX RANGE SCAN          | LOC_COUNTRY_IX   |      5 |      8 |00:00:00.01 |       3 |       |       |          |
|* 11 |      INDEX RANGE SCAN            | DEPT_LOCATION_IX |      1 |      1 |00:00:00.01 |       2 |       |       |          |
|* 12 |     TABLE ACCESS BY INDEX ROWID  | DEPARTMENT       |      1 |      1 |00:00:00.01 |       1 |       |       |          |
---------------------------------------------------------------------------------------------------------------------------------
 
Query Block Name / Object Alias (identified by operation id):
-------------------------------------------------------------
 
   1 - SEL$1
   2 - SEL$1 / E@SEL$1
   3 - SEL$1 / E@SEL$1
   4 - SEL$2 / V@SEL$1
   5 - SEL$2
   8 - SEL$2 / C@SEL$2
   9 - SEL$2 / L@SEL$2
  10 - SEL$2 / L@SEL$2
  11 - SEL$2 / D@SEL$2
  12 - SEL$2 / D@SEL$2
 
Outline Data
-------------
 
  /*+
      BEGIN_OUTLINE_DATA
      IGNORE_OPTIM_EMBEDDED_HINTS
      OPTIMIZER_FEATURES_ENABLE('11.2.0.1')
      DB_VERSION('11.2.0.1')
      ALL_ROWS
      OUTLINE_LEAF(@"SEL$2")
      OUTLINE_LEAF(@"SEL$1")
      INDEX_RS_ASC(@"SEL$1" "E"@"SEL$1" ("EMPLOYEE"."JOB_ID"))
      NO_ACCESS(@"SEL$1" "V"@"SEL$1")
      LEADING(@"SEL$1" "E"@"SEL$1" "V"@"SEL$1")
      USE_HASH(@"SEL$1" "V"@"SEL$1")
      INDEX(@"SEL$2" "C"@"SEL$2" ("COUNTRY"."REGION_ID"))
      INDEX_RS_ASC(@"SEL$2" "L"@"SEL$2" ("LOCATION"."COUNTRY_ID"))
      INDEX(@"SEL$2" "D"@"SEL$2" ("DEPARTMENT"."LOCATION_ID"))
      LEADING(@"SEL$2" "C"@"SEL$2" "L"@"SEL$2" "D"@"SEL$2")
      USE_NL(@"SEL$2" "L"@"SEL$2")
      USE_NL(@"SEL$2" "D"@"SEL$2")
      NLJ_BATCHING(@"SEL$2" "D"@"SEL$2")
      END_OUTLINE_DATA
  */
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   1 - access("E"."DEPARTMENT_ID"="V"."DEPARTMENT_ID")
   3 - access("E"."JOB_ID"='ST_CLERK')
   8 - access("C"."REGION_ID"=2)
   9 - filter("L"."POSTAL_CODE"='99236')
  10 - access("L"."COUNTRY_ID"="C"."COUNTRY_ID")
  11 - access("D"."LOCATION_ID"="L"."LOCATION_ID")
  12 - filter("D"."DEPARTMENT_NAME"='Shipping')
 
 
-- Query Transformation 이 발생한 경우 힌트를 어떻게 적용할 것인가? (= View Merging이 발생하여 QT가 됐을 때)

-- 변경된 쿼리블럭
   1 - SEL$F5BB74E1
   5 - SEL$F5BB74E1 / C@SEL$2
   7 - SEL$F5BB74E1 / D@SEL$2
   8 - SEL$F5BB74E1 / D@SEL$2
   9 - SEL$F5BB74E1 / L@SEL$2
  10 - SEL$F5BB74E1 / L@SEL$2
  11 - SEL$F5BB74E1 / E@SEL$1
  12 - SEL$F5BB74E1 / E@SEL$1
  
-- 변경 안된 쿼리 블록
   1 - SEL$1
   2 - SEL$1 / E@SEL$1
   3 - SEL$1 / E@SEL$1
   4 - SEL$2 / V@SEL$1
   5 - SEL$2
   8 - SEL$2 / C@SEL$2
  10 - SEL$2 / D@SEL$2
  11 - SEL$2 / D@SEL$2
  12 - SEL$2 / L@SEL$2
  13 - SEL$2 / L@SEL$2
```

## 한 단계 업그레이드 해서, 뷰 내부의 테이블에 대한 조인순서의 변경은 가능한가?
* 즉 조인순서를 Employee -> 뷰(Country -> Location -> Department ) 로 변경하고 싶다.
* Global Hit 를 사용하라.

#### 순서 변경안된 SQL
```sql
SELECT /*+ GATHER_PLAN_STATISTICS LEADING(E V.C V.L V.D) */
       e.employee_id, e.first_name, e.last_name, e.job_id, v.department_name
  FROM employee e, v_dept v
 WHERE e.department_id = v.department_id
   AND v.department_name = 'Shipping'
   AND v.postal_code = '99236'
   AND v.region_id = 2
   AND e.job_id = 'ST_CLERK';

SELECT * FROM 
TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL, 'allstats last -rows +alias +outline +predicate'));
```

```sql
PLAN_TABLE_OUTPUT
----------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                        | Name              | Starts | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
----------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                 |                   |      1 |     20 |00:00:00.01 |      13 |       |       |          |
|   1 |  NESTED LOOPS                    |                   |      1 |     20 |00:00:00.01 |      13 |       |       |          |
|   2 |   NESTED LOOPS                   |                   |      1 |     45 |00:00:00.01 |      10 |       |       |          |
|   3 |    NESTED LOOPS                  |                   |      1 |      1 |00:00:00.01 |       8 |       |       |          |
|   4 |     MERGE JOIN CARTESIAN         |                   |      1 |      5 |00:00:00.01 |       4 |       |       |          |
|*  5 |      INDEX RANGE SCAN            | COUN_REGION_IDX   |      1 |      5 |00:00:00.01 |       2 |       |       |          |
|   6 |      BUFFER SORT                 |                   |      5 |      5 |00:00:00.01 |       2 |  2048 |  2048 | 2048  (0)|
|   7 |       TABLE ACCESS BY INDEX ROWID| DEPARTMENT        |      1 |      1 |00:00:00.01 |       2 |       |       |          |
|*  8 |        INDEX RANGE SCAN          | DEPT_NAME_IDX     |      1 |      1 |00:00:00.01 |       1 |       |       |          |
|*  9 |     TABLE ACCESS BY INDEX ROWID  | LOCATION          |      5 |      1 |00:00:00.01 |       4 |       |       |          |
|* 10 |      INDEX RANGE SCAN            | LOC_POSTAL_IDX    |      5 |      5 |00:00:00.01 |       3 |       |       |          |
|* 11 |    INDEX RANGE SCAN              | EMP_DEPARTMENT_IX |      1 |     45 |00:00:00.01 |       2 |       |       |          |
|* 12 |   TABLE ACCESS BY INDEX ROWID    | EMPLOYEE          |     45 |     20 |00:00:00.01 |       3 |       |       |          |
----------------------------------------------------------------------------------------------------------------------------------
 
-- SEL$F5BB74E1 : 쿼리블럭명
-- 오브젝트 Alias
--	E@SEL$1
--	C@SEL$2
--	D@SEL$2
--	L@SEL$2

Query Block Name / Object Alias (identified by operation id):
-------------------------------------------------------------
 
   1 - SEL$F5BB74E1
   5 - SEL$F5BB74E1 / C@SEL$2
   7 - SEL$F5BB74E1 / D@SEL$2
   8 - SEL$F5BB74E1 / D@SEL$2
   9 - SEL$F5BB74E1 / L@SEL$2
  10 - SEL$F5BB74E1 / L@SEL$2
  11 - SEL$F5BB74E1 / E@SEL$1
  12 - SEL$F5BB74E1 / E@SEL$1
 
Outline Data
-------------
 
  /*+
      BEGIN_OUTLINE_DATA
      IGNORE_OPTIM_EMBEDDED_HINTS
      OPTIMIZER_FEATURES_ENABLE('11.2.0.1')
      DB_VERSION('11.2.0.1')
      ALL_ROWS
      OUTLINE_LEAF(@"SEL$F5BB74E1")
      MERGE(@"SEL$2")
      OUTLINE(@"SEL$1")
      OUTLINE(@"SEL$2")
      INDEX(@"SEL$F5BB74E1" "C"@"SEL$2" ("COUNTRY"."REGION_ID"))
      INDEX_RS_ASC(@"SEL$F5BB74E1" "D"@"SEL$2" ("DEPARTMENT"."DEPARTMENT_NAME"))
      INDEX_RS_ASC(@"SEL$F5BB74E1" "L"@"SEL$2" ("LOCATION"."POSTAL_CODE"))
      INDEX(@"SEL$F5BB74E1" "E"@"SEL$1" ("EMPLOYEE"."DEPARTMENT_ID"))
      LEADING(@"SEL$F5BB74E1" "C"@"SEL$2" "D"@"SEL$2" "L"@"SEL$2" "E"@"SEL$1")
      USE_MERGE_CARTESIAN(@"SEL$F5BB74E1" "D"@"SEL$2")
      USE_NL(@"SEL$F5BB74E1" "L"@"SEL$2")
      USE_NL(@"SEL$F5BB74E1" "E"@"SEL$1")
      NLJ_BATCHING(@"SEL$F5BB74E1" "E"@"SEL$1")
      END_OUTLINE_DATA
  */
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   5 - access("C"."REGION_ID"=2)
   8 - access("D"."DEPARTMENT_NAME"='Shipping')
   9 - filter(("D"."LOCATION_ID"="L"."LOCATION_ID" AND "L"."COUNTRY_ID"="C"."COUNTRY_ID"))
  10 - access("L"."POSTAL_CODE"='99236')
  11 - access("E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID")
  12 - filter("E"."JOB_ID"='ST_CLERK')
```

#### 순서 변경된 SQL
* QT가 발생한 경우 hint에 쿼리블럭명과 Object Alias를 사용해야 순서를 조정할 수 있다.
* 쿼리블럭명 : @SEL$F5BB74E1
* Object Alias : E@SEL$1 C@SEL$2 L@SEL$2 D@SEL$2
```sql
SELECT /*+ GATHER_PLAN_STATISTICS LEADING(@SEL$F5BB74E1 E@SEL$1 C@SEL$2 L@SEL$2 D@SEL$2) */
       e.employee_id, e.first_name, e.last_name, e.job_id, v.department_name
  FROM employee e, v_dept v
 WHERE e.department_id = v.department_id
   AND v.department_name = 'Shipping'
   AND v.postal_code = '99236'
   AND v.region_id = 2
   AND e.job_id = 'ST_CLERK';

SELECT * FROM 
TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL, 'allstats last -rows +alias +outline +predicate'));
```

```sql
PLAN_TABLE_OUTPUT
-------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                       | Name            | Starts | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
-------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                |                 |      1 |     20 |00:00:00.01 |      12 |       |       |          |
|   1 |  NESTED LOOPS                   |                 |      1 |     20 |00:00:00.01 |      12 |       |       |          |
|   2 |   NESTED LOOPS                  |                 |      1 |     20 |00:00:00.01 |      10 |       |       |          |
|*  3 |    HASH JOIN                    |                 |      1 |     20 |00:00:00.01 |       6 |   894K|   894K|  752K (0)|
|   4 |     MERGE JOIN CARTESIAN        |                 |      1 |    100 |00:00:00.01 |       3 |       |       |          |
|   5 |      TABLE ACCESS BY INDEX ROWID| EMPLOYEE        |      1 |     20 |00:00:00.01 |       2 |       |       |          |
|*  6 |       INDEX RANGE SCAN          | EMP_JOB_IX      |      1 |     20 |00:00:00.01 |       1 |       |       |          |
|   7 |      BUFFER SORT                |                 |     20 |    100 |00:00:00.01 |       1 |  2048 |  2048 | 2048  (0)|
|*  8 |       INDEX RANGE SCAN          | COUN_REGION_IDX |      1 |      5 |00:00:00.01 |       1 |       |       |          |
|   9 |     TABLE ACCESS BY INDEX ROWID | LOCATION        |      1 |      1 |00:00:00.01 |       3 |       |       |          |
|* 10 |      INDEX RANGE SCAN           | LOC_POSTAL_IDX  |      1 |      1 |00:00:00.01 |       2 |       |       |          |
|* 11 |    INDEX RANGE SCAN             | DEPT_NAME_IDX   |     20 |     20 |00:00:00.01 |       4 |       |       |          |
|* 12 |   TABLE ACCESS BY INDEX ROWID   | DEPARTMENT      |     20 |     20 |00:00:00.01 |       2 |       |       |          |
-------------------------------------------------------------------------------------------------------------------------------
 
Query Block Name / Object Alias (identified by operation id):
-------------------------------------------------------------
 
   1 - SEL$F5BB74E1
   5 - SEL$F5BB74E1 / E@SEL$1
   6 - SEL$F5BB74E1 / E@SEL$1
   8 - SEL$F5BB74E1 / C@SEL$2
   9 - SEL$F5BB74E1 / L@SEL$2
  10 - SEL$F5BB74E1 / L@SEL$2
  11 - SEL$F5BB74E1 / D@SEL$2
  12 - SEL$F5BB74E1 / D@SEL$2
 
Outline Data
-------------
 
  /*+
      BEGIN_OUTLINE_DATA
      IGNORE_OPTIM_EMBEDDED_HINTS
      OPTIMIZER_FEATURES_ENABLE('11.2.0.1')
      DB_VERSION('11.2.0.1')
      ALL_ROWS
      OUTLINE_LEAF(@"SEL$F5BB74E1")
      MERGE(@"SEL$2")
      OUTLINE(@"SEL$1")
      OUTLINE(@"SEL$2")
      INDEX_RS_ASC(@"SEL$F5BB74E1" "E"@"SEL$1" ("EMPLOYEE"."JOB_ID"))
      INDEX(@"SEL$F5BB74E1" "C"@"SEL$2" ("COUNTRY"."REGION_ID"))
      INDEX_RS_ASC(@"SEL$F5BB74E1" "L"@"SEL$2" ("LOCATION"."POSTAL_CODE"))
      INDEX(@"SEL$F5BB74E1" "D"@"SEL$2" ("DEPARTMENT"."DEPARTMENT_NAME"))
      LEADING(@"SEL$F5BB74E1" "E"@"SEL$1" "C"@"SEL$2" "L"@"SEL$2" "D"@"SEL$2")
      USE_MERGE_CARTESIAN(@"SEL$F5BB74E1" "C"@"SEL$2")
      USE_HASH(@"SEL$F5BB74E1" "L"@"SEL$2")
      USE_NL(@"SEL$F5BB74E1" "D"@"SEL$2")
      NLJ_BATCHING(@"SEL$F5BB74E1" "D"@"SEL$2")
      END_OUTLINE_DATA
  */
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   3 - access("L"."COUNTRY_ID"="C"."COUNTRY_ID")
   6 - access("E"."JOB_ID"='ST_CLERK')
   8 - access("C"."REGION_ID"=2)
  10 - access("L"."POSTAL_CODE"='99236')
  11 - access("D"."DEPARTMENT_NAME"='Shipping')
  12 - filter(("E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID" AND "D"."LOCATION_ID"="L"."LOCATION_ID"))
```

## Global Hint 는 위험한가 ?
* SQL을 다시 실행할 경우 쿼리블럭이 바뀌지 않을까하는 걱정 ? ( Global Hint,쿼리블록 표기법)
* 쿼리블럭이 변경 되는 경우는 1) SQL 이 변경되거나, 2)통계정보 등이 변경되어 Plan 이 변경되는 경우, 이 경우는 일반적인 힌트도 적용 되지 않는다. 
* 고로 Global Hint 쿼리블록 표기법이 특별히 더 위험하지 않다.

## 10053 이벤트 내용 순서
```sql
1) DBMS 정보, OS 정보, 하드웨어 정보 그리고 SQL 을 실행한 Client 정보
2) 쿼리블럭 정보와 수행된 SQL을 Parse 로 부터 받아서 출력한다.
3) 10053 Event 에서 사용될 용어 출력
4) Optimizer 에 관련된 성능 파라미터를 출력, Bug Fix Control 정보 출력
   Default 값을 기본적으로 출력하되, 수정된 파라미터가 있을 경우 따로 출력
   SQL 에서 opt_param 힌트를 사용 시, 따로 출력
5) Heuristic Query Transformation(이하 HQT)과정 출력
6) Bind Peeking 이 수행
7) Cost Based Query Transformation(이하 CBQT)과정 출력
   연이어 HQT 혹은 CBQT 로 인해 신규로 생성된 쿼리블럭 정보 출력
8) 시스템 통계정보와 테이블과 인덱스의 통계정보 출력
9) 테이블 단위로 최적의 Access Path 와 Cost 출력
10) 최적의 조인방법과 조인순서 정함
11) SQL Dump 와 Explain Plan Dump 를 수행 하여 SQL과 실행계획을 출력
12) 마지막으로 Optimizer state dump 와 Hint dump 를 수행하여 SQL 이 수행되었던
    당시의 옵티마이져 관련 파라미터 정보, Bug Fix Control, 최종 적용된 힌트 정보 출력

-- Query Transformation : 5번과 7번
-- Physical Transformation : 9번과 10번
```

## 10053 Event Trace
```sql
ALTER SESSION SET optimizer_mode = first_rows_1;
ALTER SESSION SET EVENTS '10053 trace name context forever, level 1'; -- Trace 생성시작

SELECT /*+ QB_NAME(main) */
       department_id, department_name, manager_id, location_id
  FROM department d
 WHERE EXISTS (SELECT /*+ QB_NAME(sub) */
                      NULL
                 FROM employee e
                WHERE e.department_id = d.department_id 
                  AND e.email = :v_email1);
       
ALTER SESSION SET EVENTS '10053 trace name context off';  -- Trace 생성 종료
```

#### trace 내용 중 중요한 부분
```sql
-- 5.Heuristic Query Transformation 과정이 출력된다.
Considering Query Transformations on query block MAIN (#0)
**************************
Query transformations (QT)
**************************
JF: Checking validity of join factorization for query block SUB (#0)
JF: Bypassed: not a UNION or UNION-ALL query block.
ST: not valid since star transformation parameter is FALSE
TE: Checking validity of table expansion for query block SUB (#0)
TE: Bypassed: No partitioned table in query block.
CBQT: Validity checks passed for cr4dmgd411hum.
CSE: Considering common sub-expression elimination in query block MAIN (#0)
*************************
Common Subexpression elimination (CSE)
*************************
CSE: Considering common sub-expression elimination in query block SUB (#0)
*************************
Common Subexpression elimination (CSE)
*************************
CSE:     CSE not performed on query block SUB (#0).
CSE:     CSE not performed on query block MAIN (#0).
OBYE:   Considering Order-by Elimination from view MAIN (#0)
***************************
Order-by elimination (OBYE)
***************************
OBYE:     OBYE bypassed: no order by to eliminate.
query block MAIN (#0) unchanged
Considering Query Transformations on query block MAIN (#0)
**************************
Query transformations (QT)
**************************
CSE: Considering common sub-expression elimination in query block MAIN (#0)
*************************
Common Subexpression elimination (CSE)
*************************
CSE: Considering common sub-expression elimination in query block SUB (#0)
*************************
Common Subexpression elimination (CSE)
*************************
CSE:     CSE not performed on query block SUB (#0).
CSE:     CSE not performed on query block MAIN (#0).
query block MAIN (#0) unchanged
apadrv-start sqlid=14668702803393823571
  :
    call(in-use=1704, alloc=16344), compile(in-use=69896, alloc=70888), execution(in-use=3832, alloc=4032)
```

```sql
-- 7. Cost Based Query Transformation 과정이 출력된다.
CBQT: Considering cost-based transformation on query block MAIN (#0)
********************************
COST-BASED QUERY TRANSFORMATIONS
********************************
FPD: Considering simple filter push (pre rewrite) in query block SUB (#0)
FPD:  Current where clause predicates "E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID" AND "E"."EMAIL"=:B1

FPD: Considering simple filter push (pre rewrite) in query block MAIN (#0)
FPD:  Current where clause predicates  EXISTS (SELECT /*+ QB_NAME ("SUB") */ NULL FROM "EMPLOYEE" "E")

OBYE:   Considering Order-by Elimination from view MAIN (#0)
***************************
Order-by elimination (OBYE)
***************************
OBYE:     OBYE bypassed: no order by to eliminate.
Considering Query Transformations on query block MAIN (#0)
**************************
Query transformations (QT)
**************************
CSE: Considering common sub-expression elimination in query block MAIN (#0)
*************************
Common Subexpression elimination (CSE)
*************************
CSE: Considering common sub-expression elimination in query block SUB (#0)
*************************
Common Subexpression elimination (CSE)
*************************
CSE:     CSE not performed on query block SUB (#0).
CSE:     CSE not performed on query block MAIN (#0).
kkqctdrvTD-start on query block MAIN (#0)
kkqctdrvTD-start: :
    call(in-use=1720, alloc=16344), compile(in-use=112344, alloc=114608), execution(in-use=3952, alloc=4032)

Registered qb: MAIN 0x1e5b8f68 (COPY MAIN)
---------------------
QUERY BLOCK SIGNATURE
---------------------
  signature(): NULL
Registered qb: SUB 0x1e5b8608 (COPY SUB)
---------------------
QUERY BLOCK SIGNATURE
---------------------
  signature(): NULL
*****************************	-- 여기서 서브쿼리가 조인으로 변경되는 작업이 수행된다.
Cost-Based Subquery Unnesting	-- 이것을 Subquery Unnesting 이라고 부른다.
*****************************
SU: Unnesting query blocks in query block MAIN (#1) that are valid to unnest.
Subquery Unnesting on query block MAIN (#1)SU: Performing unnesting that does not require costing.
SU: Considering subquery unnest on query block MAIN (#1).
SU:   Checking validity of unnesting subquery SUB (#2)
SU:   Passed validity checks.
SU:   Transforming EXISTS subquery to a join.		-- 서브쿼리가 조인으로 바뀐 것을 알 수 있다.
Registered qb: SEL$526A7031 0x1e5b8f68 (SUBQUERY UNNEST MAIN; SUB)
---------------------
QUERY BLOCK SIGNATURE				-- 새로 생성된 쿼리블럭의 구조를 보여준다.
---------------------
  signature (): qb_name=SEL$526A7031 nbfros=2 flg=0
    fro(0): flg=0 objn=95044 hint_alias="D"@"MAIN"
    fro(1): flg=0 objn=95046 hint_alias="E"@"SUB" 	-- 쿼리블럭 SUB가 Unnesting 되어 쿼리블럭 SEL$526A7031에 통합되었다.  

*******************************
Cost-Based Complex View Merging
*******************************
CVM: Finding query blocks in query block SEL$526A7031 (#1) that are valid to merge.
kkqctdrvTD-cleanup: transform(in-use=8888, alloc=12568) :
    call(in-use=22400, alloc=32712), compile(in-use=133528, alloc=147864), execution(in-use=3992, alloc=8088)

kkqctdrvTD-end:
    call(in-use=22400, alloc=32712), compile(in-use=121136, alloc=147864), execution(in-use=3992, alloc=8088)

-- (중간 생략)

JPPD:  Considering Cost-based predicate pushdown from query block SEL$526A7031 (#1)
************************************
Cost-based predicate pushdown (JPPD)
************************************
kkqctdrvTD-start on query block SEL$526A7031 (#1)
kkqctdrvTD-start: :
    call(in-use=22856, alloc=49080), compile(in-use=125600, alloc=147864), execution(in-use=4032, alloc=8088)

kkqctdrvTD-cleanup: transform(in-use=0, alloc=0) :
    call(in-use=22856, alloc=49080), compile(in-use=126248, alloc=147864), execution(in-use=4032, alloc=8088)

kkqctdrvTD-end:
    call(in-use=22856, alloc=49080), compile(in-use=126608, alloc=147864), execution(in-use=4032, alloc=8088)

JPPD: Applying transformation directives
query block MAIN transformed to SEL$526A7031 (#1)
FPD: Considering simple filter push in query block SEL$526A7031 (#1)
"E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID" AND SYS_OP_C2C("E"."EMAIL")=:B1
try to generate transitive predicate from check constraints for query block SEL$526A7031 (#1)
finally: "E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID" AND SYS_OP_C2C("E"."EMAIL")=:B1

Final query after transformations:
*************************
First K Rows: Setup begin
kkoqbc: optimizing query block SEL$526A7031 (#1)
        
        :
    call(in-use=23064, alloc=49080), compile(in-use=133872, alloc=147864), execution(in-use=4368, alloc=8088)

kkoqbc-subheap (create addr=0x000000001E5BD670)
****************
QUERY BLOCK TEXT
****************
SELECT /*+ QB_NAME(main) */
       department_id, department_name, manager_id, location_id
  FROM department d
 WHERE EXISTS (SELECT /*+ QB_NAME(sub) */
                      NULL
                 FROM employee e
                WHERE e.department_id = d.d
-- 최종적으로 완성된 쿼리블럭 정보가 출력된다.
---------------------
QUERY BLOCK SIGNATURE -- 쿼리블럭 SEL$526A7031이 생성되었고 쿼리블럭 MAIN과 SUB가 통합되었다.
---------------------
signature (optimizer): qb_name=SEL$526A7031 nbfros=2 flg=0
  fro(0): flg=0 objn=95044 hint_alias="D"@"MAIN"
  fro(1): flg=0 objn=95046 hint_alias="E"@"SUB"
```
