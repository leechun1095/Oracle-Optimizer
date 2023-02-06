# Heuristic Query Transformation 이란?
* Heuristic : 문제 해결에 있어 노력을 최소화 하기 위해 사용되는 고찰이나 과정
* 자원/시간 낭비하는 Coasting 과정을 생략하여 특정 Rule을 적용하여 최적화를 진행하겠다는 의미  

> 1. "특정 작업의 부하를 줄일 수 있는 가장 좋은 방법은 그 작업을 수행하지 않는 것이다."  
  > SQL 자체 성능 최적화 + Hard parsing 시 Coasting 과정에서 부하 제거 하여 parsing 의 성능을 빠르게함
```sql
-- count( * ) 를 하는데 Order by 는 필요없음 ! 불필요한 부하 제거를 위해 ORDER BY 를 제거하는것.
-- > Heuristic Rule
SELECT COUNT(*)
  FROM tab1
 ORDER BY col1;
```
> 2. "꼭 필요한 일만 해라"  
  > 고객번호, 고객명을 select 하는데 공통코드 테이블과 조인은 필요없다. ( A.고객구분코드 가 NULL 인 데이터가 없을 경우 )  
  > 공통코드 테이블에 Unique 인덱스만 있으면 위 SQL에서 공통코드쪽을 삭제한다.
```sql
SELECT a.고객번호, a.고객명
  FROM 고객 a, 공통코드 b 
 WHERE a.고객구분코드=b.고객구분코드(+);
```

</br>

## 2.1. CSE (Common Subexpression Elimination)
#### Where절에서 or 사용시 중첩된 조건절은 제거하라
* CSE : 중첩되는 Where조건을 찾아서 제거하는 것

```sql
SELECT /*+ GATHER_PLAN_STATISTICS */
       DEPARTMENT_ID, SALARY
  FROM EMPLOYEE
 WHERE (DEPARTMENT_ID = 60 AND SALARY = 4200) 
    OR (DEPARTMENT_ID = 60);
```
* 같은 조건이 존재. where절에 department_id = 60 만 있으면 된다.  
* Logical Optimizer는 이러한 경우에 (department_id = 60 AND salary = 4200) 조건을 삭제해 버린다. ( 넓은 범위 조건을 남긴다. )  
* predicate information 을 보면 DEPARTMENT_ID=60 만이 남아있고 나머지 조건은 삭제됨.

```sql
PLAN_TABLE_OUTPUT 
-----------------------------------------------------------------------------------------------------------
| Id  | Operation                   | Name              | Starts | E-Rows | A-Rows |   A-Time   | Buffers |
-----------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |                   |      1 |        |      5 |00:00:00.01 |       4 |
|   1 |  TABLE ACCESS BY INDEX ROWID| EMPLOYEE          |      1 |      5 |      5 |00:00:00.01 |       4 |
|*  2 |   INDEX RANGE SCAN          | EMP_DEPARTMENT_IX |      1 |      5 |      5 |00:00:00.01 |       2 |
-----------------------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   2 - access("DEPARTMENT_ID"=60)
```

```sql
-- TRACE 10053 로그 확인하기
ALTER SESSION SET OPTIMIZER_MODE = FIRST_ROWS_1;
ALTER SESSION SET EVENTS '10053 TRACE NAME CONTEXT FOREVER, LEVEL 1'; -- TRACE 생성시작

SELECT /*+ GATHER_PLAN_STATISTICS */
       DEPARTMENT_ID, SALARY
  FROM EMPLOYEE
 WHERE (DEPARTMENT_ID = 60 AND SALARY = 4200) 
    OR (DEPARTMENT_ID = 60);
       
ALTER SESSION SET EVENTS '10053 TRACE NAME CONTEXT OFF';  -- TRACE 생성 종료
```
* 10053 Trace : Or 부분이 (salary = 4200 OR 0=0) 로 재작성 되었다. 0=0 조건은 항상 True이므로 괄호 안에 모든 조건은 삭제되어도 무방하다.

</br>

#### CSE(Common Subexpression Elimination) 기능을 없애고 실행한 결과
* Full table Scan 발생.
* predicate information 을 보면 where 절이 그대로 수행됨.

```sql
ALTER SESSION SET "_eliminate_common_subexpr" = FALSE;

SELECT /*+ GATHER_PLAN_STATISTICS */
       DEPARTMENT_ID, SALARY
  FROM EMPLOYEE
 WHERE (DEPARTMENT_ID = 60 AND SALARY = 4200) 
    OR (DEPARTMENT_ID = 60);

SELECT *
  FROM TABLE (DBMS_XPlan.display_cursor (NULL, NULL, 'allstats last'));
```
```sql
PLAN_TABLE_OUTPUT
----------------------------------------------------------------------------------------
| Id  | Operation         | Name     | Starts | E-Rows | A-Rows |   A-Time   | Buffers |
----------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |          |      1 |        |      5 |00:00:00.01 |       8 |
|*  1 |  TABLE ACCESS FULL| EMPLOYEE |      1 |      5 |      5 |00:00:00.01 |       8 |
----------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   1 - filter(("DEPARTMENT_ID"=60 OR ("SALARY"=4200 AND "DEPARTMENT_ID"=60)))
```

## 2.2 JE (Join Elimination)
#### 부모 테이블 & 자식 테이블 조인 시 SELECT 절에 자식 테이블 컬럼만 사용할 때 PK가 생성되어 있다면 부모 테이블을 접근하지 않는다. 왜냐하면 FK가 존재하는 자식 테이블에 존재하는 값이 100% 부모 테이블에도 존재한다는 것을 보장하기 때문에 부모 테이블을 액세스 할 필요가 없어지기 때문이다.
* FK를 사용하면 느려진다 ? - JE기능이 나오기 전까지는 그러했다.
* 이제는 상황에 따라 FK로 인해 성능이 향상되는 경우가 많이 있다.
* SELECT 절에 자식 쪽 컬럼만 나열하는 경우, 부모와 자식간에 FK를 만들어 두면 부모테이블을 액세스 하지 않는다.
  * FK가 자식 테이블에 존재하는 값이 100% 부모테이블에도 존재한다는 것을 보장함.
  * 부모쪽 테이블 조인을 제거하여 I/O와 조인 부하를 감소시킬 수 있다.
* JE기능 : FROM절의 부모테이블 제거, WHERE절 부모 자식간 조인절 제거 ( 10gR2 )

#### JE 가 발생하지 않음
```sql
SELECT /*+ GATHER_PLAN_STATISTICS LEADING(E) USE_NL(D) */
       E.EMPLOYEE_ID, E.FIRST_NAME, E.LAST_NAME, E.EMAIL, E.SALARY
  FROM EMPLOYEE E, DEPARTMENT D
 WHERE E.DEPARTMENT_ID = D.DEPARTMENT_ID 
   AND E.JOB_ID = 'SH_CLERK';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ALLSTATS LAST'));

PLAN_TABLE_OUTPUT
-----------------------------------------------------------------------------------------------------
| Id  | Operation                    | Name       | Starts | E-Rows | A-Rows |   A-Time   | Buffers |
-----------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |            |      1 |        |     20 |00:00:00.01 |       9 |
|   1 |  NESTED LOOPS                |            |      1 |      2 |     20 |00:00:00.01 |       9 |
|   2 |   TABLE ACCESS BY INDEX ROWID| EMPLOYEE   |      1 |      3 |     20 |00:00:00.01 |       5 |
|*  3 |    INDEX RANGE SCAN          | EMP_JOB_IX |      1 |     20 |     20 |00:00:00.01 |       2 |
|*  4 |   INDEX UNIQUE SCAN          | DEPT_ID_PK |     20 |      1 |     20 |00:00:00.01 |       4 |
-----------------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   3 - access("E"."JOB_ID"='SH_CLERK')
   4 - access("E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID")
```

#### JE 발생
```sql
-- FK 생성
--  > 부모 테이블 : DEPARTMENT
--  > 자식 테이블 : EMPLOYEE
ALTER TABLE EMPLOYEE ADD CONSTRAINT EMP_DEPT_FK
FOREIGN KEY (DEPARTMENT_ID) REFERENCES DEPARTMENT (DEPARTMENT_ID);

SELECT /*+ GATHER_PLAN_STATISTICS LEADING(E) USE_NL(D) */
       E.EMPLOYEE_ID, E.FIRST_NAME, E.LAST_NAME, E.EMAIL, E.SALARY
  FROM EMPLOYEE E, DEPARTMENT D
 WHERE E.DEPARTMENT_ID = D.DEPARTMENT_ID 
   AND E.JOB_ID = 'SH_CLERK';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ADVANCED ALLSTATS LAST'));


PLAN_TABLE_OUTPUT
------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                   | Name       | Starts | E-Rows |E-Bytes| Cost (%CPU)| E-Time   | A-Rows |   A-Time   | Buffers |
------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |            |      1 |        |       |     2 (100)|          |     20 |00:00:00.01 |       5 |
|*  1 |  TABLE ACCESS BY INDEX ROWID| EMPLOYEE   |      1 |      1 |    52 |     2   (0)| 00:00:01 |     20 |00:00:00.01 |       5 |
|*  2 |   INDEX RANGE SCAN          | EMP_JOB_IX |      1 |     20 |       |     1   (0)| 00:00:01 |     20 |00:00:00.01 |       2 |
------------------------------------------------------------------------------------------------------------------------------------
 
Query Block Name / Object Alias (identified by operation id):
-------------------------------------------------------------
 
   1 - SEL$F7859CDE / E@SEL$1
   2 - SEL$F7859CDE / E@SEL$1
 
Outline Data
-------------
 
  /*+
      BEGIN_OUTLINE_DATA
      IGNORE_OPTIM_EMBEDDED_HINTS
      OPTIMIZER_FEATURES_ENABLE('11.2.0.1')
      DB_VERSION('11.2.0.1')
      OPT_PARAM('_eliminate_common_subexpr' 'false')
      FIRST_ROWS(1)
      OUTLINE_LEAF(@"SEL$F7859CDE")
      ELIMINATE_JOIN(@"SEL$1" "D"@"SEL$1")    -- 오라클이 JE 기능을 사용하기 위해 ELIMINATE JOIN 힌트를 사용함.
      OUTLINE(@"SEL$1")
      INDEX_RS_ASC(@"SEL$F7859CDE" "E"@"SEL$1" ("EMPLOYEE"."JOB_ID"))
      END_OUTLINE_DATA
  */
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   1 - filter("E"."DEPARTMENT_ID" IS NOT NULL)
   2 - access("E"."JOB_ID"='SH_CLERK')
 
Column Projection Information (identified by operation id):
-----------------------------------------------------------
 
   1 - "E"."EMPLOYEE_ID"[NUMBER,22], "E"."FIRST_NAME"[VARCHAR2,20], "E"."LAST_NAME"[VARCHAR2,25], "E"."EMAIL"[VARCHAR2,25], 
       "E"."SALARY"[NUMBER,22]
   2 - "E".ROWID[ROWID,10]
```

#### FK가 존재하더라도 NO_ELIMINATION_JOIN 힌트를 사용하거나 _optimizer_join_elimination_enabled 파라미터를 False로 적용하는 경우 JE 기능을 사용할 수 없다.
> ALTER SESSION SET "_optimizer_join_elimination_enabled" = FALSE;

```sql
SELECT /*+ GATHER_PLAN_STATISTICS NO_ELIMINATE_JOIN(D) */
       E.EMPLOYEE_ID, E.FIRST_NAME, E.LAST_NAME, E.EMAIL, E.SALARY
  FROM EMPLOYEE E, DEPARTMENT D
 WHERE E.DEPARTMENT_ID = D.DEPARTMENT_ID 
   AND E.JOB_ID = 'SH_CLERK';
   
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL,'ADVANCED ALLSTATS LAST'));



PLAN_TABLE_OUTPUT
-------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                    | Name       | Starts | E-Rows |E-Bytes| Cost (%CPU)| E-Time   | A-Rows |   A-Time   | Buffers |
-------------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |            |      1 |        |       |     2 (100)|          |     20 |00:00:00.01 |       9 |
|   1 |  NESTED LOOPS                |            |      1 |      2 |   112 |     2   (0)| 00:00:01 |     20 |00:00:00.01 |       9 |
|   2 |   TABLE ACCESS BY INDEX ROWID| EMPLOYEE   |      1 |     19 |   988 |     2   (0)| 00:00:01 |     20 |00:00:00.01 |       5 |
|*  3 |    INDEX RANGE SCAN          | EMP_JOB_IX |      1 |     20 |       |     1   (0)| 00:00:01 |     20 |00:00:00.01 |       2 |
|*  4 |   INDEX UNIQUE SCAN          | DEPT_ID_PK |     20 |      1 |     4 |     0   (0)|          |     20 |00:00:00.01 |       4 |
-------------------------------------------------------------------------------------------------------------------------------------
 
Query Block Name / Object Alias (identified by operation id):
-------------------------------------------------------------
 
   1 - SEL$1
   2 - SEL$1 / E@SEL$1
   3 - SEL$1 / E@SEL$1
   4 - SEL$1 / D@SEL$1
 
Outline Data
-------------
 
  /*+
      BEGIN_OUTLINE_DATA
      IGNORE_OPTIM_EMBEDDED_HINTS
      OPTIMIZER_FEATURES_ENABLE('11.2.0.1')
      DB_VERSION('11.2.0.1')
      OPT_PARAM('_eliminate_common_subexpr' 'false')
      FIRST_ROWS(1)
      OUTLINE_LEAF(@"SEL$1")
      INDEX_RS_ASC(@"SEL$1" "E"@"SEL$1" ("EMPLOYEE"."JOB_ID"))
      INDEX(@"SEL$1" "D"@"SEL$1" ("DEPARTMENT"."DEPARTMENT_ID"))
      LEADING(@"SEL$1" "E"@"SEL$1" "D"@"SEL$1")
      USE_NL(@"SEL$1" "D"@"SEL$1")
      END_OUTLINE_DATA
  */
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   3 - access("E"."JOB_ID"='SH_CLERK')
   4 - access("E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID")
 
Column Projection Information (identified by operation id):
-----------------------------------------------------------
 
   1 - "E"."EMPLOYEE_ID"[NUMBER,22], "E"."FIRST_NAME"[VARCHAR2,20], "E"."LAST_NAME"[VARCHAR2,25], "E"."EMAIL"[VARCHAR2,25], 
       "E"."SALARY"[NUMBER,22]
   2 - "E"."EMPLOYEE_ID"[NUMBER,22], "E"."FIRST_NAME"[VARCHAR2,20], "E"."LAST_NAME"[VARCHAR2,25], "E"."EMAIL"[VARCHAR2,25], 
       "E"."SALARY"[NUMBER,22], "E"."DEPARTMENT_ID"[NUMBER,22]
   3 - "E".ROWID[ROWID,10]
```

#### JE기능 제약사항
* join 되는 FK 가 Multi Column일 경우 기능이 수행되지 않는다. ( sequence 를 이용하여 인공key를 pk로 사용해야 한다.)
* Equal(=) 조인 이외에 Range join (Between , > , < , Like) 등을 조인절에 사용하면 기능이 수행되지 않는다.
* Department 테이블의 PK를 제외한 컬럼을 select 절에 사용하면 JE 기능을 사용할 수 없다.
* Oracle 10g 에서는 JE가 발생할 수 있는 상황에 ANSI SQL (ANSI Style join)을 사용하면 안된다.
* ANSI SQL에서 JE 기능은 11g 부터 사용 가능하다.

</br>

## 2.3. OE (Outer Join Table Elimination)
#### 불필요한 Outer쪽 테이블은 삭제하라
* 11g 에서 OE 기능이 추가됨.
* OE : OUTER조인 시 불필요한 테이블 제거하는 기능
* JE와 달리 PK/FK가 필요 없다

PK/FK를 제거, Unique 인덱스만 있으면 DEPARTMENT쪽을 액세스 하지 않음.
```sql
ALTER TABLE EMPLOYEE DROP CONSTRAINT EMP_DEPT_FK;
ALTER TABLE DEPARTMENT DROP PRIMARY KEY CASCADE KEEP INDEX;  

SELECT /*+ GATHER_PLAN_STATISTICS */ E.EMPLOYEE_ID, E.EMAIL, E.DEPARTMENT_ID
  FROM EMPLOYEE E, DEPARTMENT D
 WHERE E.DEPARTMENT_ID = D.DEPARTMENT_ID(+);


PLAN_TABLE_OUTPUT
----------------------------------------------------------------------------------------
| Id  | Operation         | Name     | Starts | E-Rows | A-Rows |   A-Time   | Buffers |
----------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |          |      1 |        |    101 |00:00:00.01 |       5 |
|   1 |  TABLE ACCESS FULL| EMPLOYEE |      1 |      1 |    101 |00:00:00.01 |       5 |
----------------------------------------------------------------------------------------
```

* OE기능을 Control 하는 파라미터는 _optimizer_join_elimination_enabled 파라미터로 JE와 같다.
* Outline Data - ELIMINATE_JOIN 힌트를 사용 ( OE기능은 JE기능을 확장한 것 )
* Outer Join Table Elimination기능의 제약사항은 JE 제약사항과 일치한다.

#### SELECT 절에 INNER 쪽 컬럼만 사용하는 경우
* Where절에 Outer 테이블 쪽의 조건이 추가되어도 결과는 동일하다.
* d.manager_id( + ) = 10 조건이 결과에 영향이 없으나, Outer Join Table Elimination이 수행되지 않음
```sql
EXPLAIN PLAN FOR
SELECT E.EMPLOYEE_ID, E.EMAIL, E.DEPARTMENT_ID
  FROM EMPLOYEE E, DEPARTMENT D
 WHERE E.DEPARTMENT_ID = D.DEPARTMENT_ID (+)
   AND D.MANAGER_ID(+) = 10;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

PLAN_TABLE_OUTPUT
-------------------------------------------------------------------------------------------
| Id  | Operation                    | Name       | Rows  | Bytes | Cost (%CPU)| Time     |
-------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |            |     1 |    22 |     3   (0)| 00:00:01 |
|   1 |  NESTED LOOPS OUTER          |            |     1 |    22 |     3   (0)| 00:00:01 |
|   2 |   TABLE ACCESS FULL          | EMPLOYEE   |     1 |    15 |     2   (0)| 00:00:01 |
|*  3 |   TABLE ACCESS BY INDEX ROWID| DEPARTMENT |     1 |     7 |     1   (0)| 00:00:01 |
|*  4 |    INDEX UNIQUE SCAN         | DEPT_ID_PK |     1 |       |     0   (0)| 00:00:01 |
-------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   3 - filter("D"."MANAGER_ID"(+)=10)
   4 - access("E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID"(+))
```

</br>

## 2.4 OJE (Outer-Join Elimination)
#### 의미 없는 Outer 조인을 Inner 조인으로 바꾸어라
* Oracle은 Outer 조인 조건절 분석하여, 가능한 경우 Outer 조인을 Inner 조인으로 변경한다.
* OE, OJE 차이점
  * OE : SQL 의 From 절에서 Outer쪽의 테이블을 삭제하는 기능
  * OJE : Outer join 을 Inner Join 으로 바꾸는 기능 ( oracle v7 )

```sql
EXPLAIN PLAN FOR
SELECT E.EMPLOYEE_ID, E.HIRE_DATE, E.SALARY, D.DEPARTMENT_NAME
  FROM EMPLOYEE E, DEPARTMENT D
 WHERE E.DEPARTMENT_ID = D.DEPARTMENT_ID(+) 
   AND D.LOCATION_ID = 1700;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);


PLAN_TABLE_OUTPUT
-----------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                    | Name             | E-Rows |E-Bytes| Cost (%CPU)| E-Time   |  OMem |  1Mem | Used-Mem |
-----------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |                  |        |       |     6 (100)|          |       |       |          |
|*  1 |  HASH JOIN                   |                  |      1 |    48 |     6  (17)| 00:00:01 |  1011K|  1011K|  850K (0)|
|   2 |   TABLE ACCESS FULL          | EMPLOYEE         |    107 |  3103 |     3   (0)| 00:00:01 |       |       |          |
|   3 |   TABLE ACCESS BY INDEX ROWID| DEPARTMENT       |     14 |   266 |     2   (0)| 00:00:01 |       |       |          |
|*  4 |    INDEX RANGE SCAN          | DEPT_LOCATION_IX |     21 |       |     1   (0)| 00:00:01 |       |       |          |
-----------------------------------------------------------------------------------------------------------------------------
 
Query Block Name / Object Alias (identified by operation id):
-------------------------------------------------------------
 
   1 - SEL$6E71C6F6
   2 - SEL$6E71C6F6 / E@SEL$1
   3 - SEL$6E71C6F6 / D@SEL$1
   4 - SEL$6E71C6F6 / D@SEL$1
 
Outline Data
-------------
 
  /*+
      BEGIN_OUTLINE_DATA
      IGNORE_OPTIM_EMBEDDED_HINTS
      OPTIMIZER_FEATURES_ENABLE('11.2.0.1')
      DB_VERSION('11.2.0.1')
      OPT_PARAM('_eliminate_common_subexpr' 'false')
      FIRST_ROWS(1)
      OUTLINE_LEAF(@"SEL$6E71C6F6")
      OUTER_JOIN_TO_INNER(@"SEL$1")                     -- ** 오라클이 outer 조인을 제거함.
      OUTLINE(@"SEL$1")
      FULL(@"SEL$6E71C6F6" "E"@"SEL$1")
      INDEX_RS_ASC(@"SEL$6E71C6F6" "D"@"SEL$1" ("DEPARTMENT"."LOCATION_ID"))
      LEADING(@"SEL$6E71C6F6" "E"@"SEL$1" "D"@"SEL$1")
      USE_HASH(@"SEL$6E71C6F6" "D"@"SEL$1")
      END_OUTLINE_DATA
  */
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   1 - access("E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID")
   4 - access("D"."LOCATION_ID"=1700)
 
Column Projection Information (identified by operation id):
-----------------------------------------------------------
 
   1 - (#keys=1) "E"."EMPLOYEE_ID"[NUMBER,22], "E"."HIRE_DATE"[DATE,7], "E"."SALARY"[NUMBER,22], 
       "D"."DEPARTMENT_NAME"[VARCHAR2,30]
   2 - "E"."EMPLOYEE_ID"[NUMBER,22], "E"."HIRE_DATE"[DATE,7], "E"."SALARY"[NUMBER,22], "E"."DEPARTMENT_ID"[NUMBER,22]
   3 - "D"."DEPARTMENT_ID"[NUMBER,22], "D"."DEPARTMENT_NAME"[VARCHAR2,30]
   4 - "D".ROWID[ROWID,10]
 
Note
-----
   - Warning: basic plan statistics not available. These are only collected when:
       * hint 'gather_plan_statistics' is used for the statement or
       * parameter 'statistics_level' is set to 'ALL', at session or system level
```

#### 조건절 수정
* Outer 조인을 제대로 사용하기 위해 where 조건을 변경해야함.
* LOCATION_ID 에 ( + ) 추가하니 HASH JOIN OUTER 가 나왔다.
* IS NULL 조건을 사용할 경우 OJE 가 작동하지 않으므로 주의해야 한다.
```sql
EXPLAIN PLAN FOR
SELECT /*+ ORDERED USE_HASH(D) */ E.EMPLOYEE_ID, E.HIRE_DATE, E.SALARY, D.DEPARTMENT_NAME
  FROM EMPLOYEE E, DEPARTMENT D
 WHERE E.DEPARTMENT_ID = D.DEPARTMENT_ID(+) 
   AND D.LOCATION_ID(+) = 1700;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

PLAN_TABLE_OUTPUT
Plan hash value: 1528295863
 
-------------------------------------------------------------------------------------------------
| Id  | Operation                    | Name             | Rows  | Bytes | Cost (%CPU)| Time     |
-------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |                  |     1 |    48 |     6  (17)| 00:00:01 |
|*  1 |  HASH JOIN OUTER             |                  |     1 |    48 |     6  (17)| 00:00:01 |
|   2 |   TABLE ACCESS FULL          | EMPLOYEE         |   107 |  3103 |     3   (0)| 00:00:01 |
|   3 |   TABLE ACCESS BY INDEX ROWID| DEPARTMENT       |    21 |   399 |     2   (0)| 00:00:01 |
|*  4 |    INDEX RANGE SCAN          | DEPT_LOCATION_IX |    21 |       |     1   (0)| 00:00:01 |
-------------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   1 - access("E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID"(+))
   4 - access("D"."LOCATION_ID"(+)=1700);
```

> D.DEPARTMENT_ID IS NULL 조건은 Outer 조인 수행 후 판단 가능하므로 OJE 사용 안되는 것은 당연한 것.
> 10053 Trace 로 IS NULL 조건으로 인해 OJE 사용 되지 않음을 확인할 수 있다.

```sql
-- ANSI 
SELECT /*+ ORDERED USE_HASH(D) */ E.EMPLOYEE_ID, E.HIRE_DATE, E.SALARY, D.DEPARTMENT_NAME
  FROM EMPLOYEE E
  LEFT JOIN DEPARTMENT D
    ON E.DEPARTMENT_ID = D.DEPARTMENT_ID 
 WHERE D.LOCATION_ID = 1700;

PLAN_TABLE_OUTPUT
-------------------------------------------------------------------------------------------------
| Id  | Operation                    | Name             | Rows  | Bytes | Cost (%CPU)| Time     |
-------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |                  |     1 |    48 |     6  (17)| 00:00:01 |
|*  1 |  HASH JOIN                   |                  |     1 |    48 |     6  (17)| 00:00:01 |
|   2 |   TABLE ACCESS FULL          | EMPLOYEE         |   107 |  3103 |     3   (0)| 00:00:01 |
|   3 |   TABLE ACCESS BY INDEX ROWID| DEPARTMENT       |    14 |   266 |     2   (0)| 00:00:01 |
|*  4 |    INDEX RANGE SCAN          | DEPT_LOCATION_IX |    21 |       |     1   (0)| 00:00:01 |
-------------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   1 - access("E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID")
   4 - access("D"."LOCATION_ID"=1700)
   
   

SELECT /*+ ORDERED USE_HASH(D) */ E.EMPLOYEE_ID, E.HIRE_DATE, E.SALARY, D.DEPARTMENT_NAME, D.LOCATION_ID
  FROM EMPLOYEE E
  LEFT JOIN DEPARTMENT D
    ON E.DEPARTMENT_ID = D.DEPARTMENT_ID 
   AND D.LOCATION_ID = 1700;

PLAN_TABLE_OUTPUT
-------------------------------------------------------------------------------------------------
| Id  | Operation                    | Name             | Rows  | Bytes | Cost (%CPU)| Time     |
-------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |                  |     1 |    48 |     6  (17)| 00:00:01 |
|*  1 |  HASH JOIN OUTER             |                  |     1 |    48 |     6  (17)| 00:00:01 |
|   2 |   TABLE ACCESS FULL          | EMPLOYEE         |   107 |  3103 |     3   (0)| 00:00:01 |
|   3 |   TABLE ACCESS BY INDEX ROWID| DEPARTMENT       |    21 |   399 |     2   (0)| 00:00:01 |
|*  4 |    INDEX RANGE SCAN          | DEPT_LOCATION_IX |    21 |       |     1   (0)| 00:00:01 |
-------------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   1 - access("E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID"(+))
   4 - access("D"."LOCATION_ID"(+)=1700)
```

#### Outer 쪽 조건에 IS NULL이 붙는다면, Anti Join 을 활용하는 것이 성능상 유리하다.
* Anti Join - 2.23 OJTAJ (Outer Join To Anti Join)

```sql
SELECT /*+ GATHER_PLAN_STATISTICS */ E.EMPLOYEE_ID, E.HIRE_DATE, E.SALARY, D.DEPARTMENT_NAME
  FROM EMPLOYEE E, DEPARTMENT D
 WHERE E.DEPARTMENT_ID = D.DEPARTMENT_ID(+) 
   AND D.DEPARTMENT_ID IS NULL;
   
SELECT T1.SQL_ID ,T1.CHILD_NUMBER ,T1.SQL_TEXT
  FROM V$SQL T1
 WHERE T1.SQL_TEXT LIKE '%GATHER_PLAN_STATISTICS%'
 ORDER BY T1.LAST_ACTIVE_TIME DESC;
 
SELECT *
  FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('4rb3k9467qk91',0,'ALLSTATS LAST'));
       
PLAN_TABLE_OUTPUT
----------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                | Name             | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
----------------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT         |                  |      1 |        |      1 |00:00:00.01 |      15 |       |       |          |
|*  1 |  FILTER                  |                  |      1 |        |      1 |00:00:00.01 |      15 |       |       |          |
|*  2 |   HASH JOIN OUTER        |                  |      1 |      1 |    107 |00:00:00.01 |      15 |  1011K|  1011K|  887K (0)|
|   3 |    TABLE ACCESS FULL     | EMPLOYEE         |      1 |    107 |    107 |00:00:00.01 |       7 |       |       |          |
|   4 |    VIEW                  | index$_join$_002 |      1 |     27 |     27 |00:00:00.01 |       8 |       |       |          |
|*  5 |     HASH JOIN            |                  |      1 |        |     27 |00:00:00.01 |       8 |  1096K|  1096K| 1539K (0)|
|   6 |      INDEX FAST FULL SCAN| DEPT_ID_PK       |      1 |     27 |     27 |00:00:00.01 |       4 |       |       |          |
|   7 |      INDEX FAST FULL SCAN| DEPT_NAME_IDX    |      1 |     27 |     27 |00:00:00.01 |       4 |       |       |          |
----------------------------------------------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   1 - filter("D"."DEPARTMENT_ID" IS NULL)
   2 - access("E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID")
   5 - access(ROWID=ROWID)   
```

</br>

## 2.5 OBYE (Order By Elimination)
#### 불필요한 Order By를 삭제하라
* Order by 가 필요없는 경우에 OBYE 가 발생된다.
* JOB_ID = 'ST_CLERK' 조건을 만족하는 데이터를 DEPARTMENT_ID 로 Group By 하여 count 하는 쿼리
```sql
ALTER SESSION SET optimizer_mode = first_rows_1;
ALTER SESSION SET EVENTS '10053 trace name context forever, level 1';

SELECT /*+ GATHER_PLAN_STATISTICS */ E.DEPARTMENT_ID, COUNT (*) CNT
  FROM EMPLOYEE E,
       (SELECT D.DEPARTMENT_ID, D.DEPARTMENT_NAME
          FROM DEPARTMENT D
         ORDER BY D.DEPARTMENT_ID) D
 WHERE E.DEPARTMENT_ID = D.DEPARTMENT_ID 
   AND E.JOB_ID = 'ST_CLERK'
 GROUP BY E.DEPARTMENT_ID;


ALTER SESSION SET EVENTS '10053 trace name context off';

SELECT * FROM 
TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL, 'allstats last -rows +alias +outline +predicate'));
```
* 실행계획으로 보면 Sort Order By 가 존재하지 않음
* 오라클이 Order by 절을 삭제하기 위한 힌트를 사용하였다 > ELIMINATE_OBY(@"SEL$2")
```sql
PLAN_TABLE_OUTPUT
----------------------------------------------------------------------------------------------------
| Id  | Operation                     | Name              | Starts | A-Rows |   A-Time   | Buffers |
----------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT              |                   |      1 |      1 |00:00:00.01 |      13 |
|   1 |  SORT GROUP BY NOSORT         |                   |      1 |      1 |00:00:00.01 |      13 |
|   2 |   NESTED LOOPS                |                   |      1 |     20 |00:00:00.01 |      13 |
|   3 |    NESTED LOOPS               |                   |      1 |    106 |00:00:00.01 |       4 |
|   4 |     INDEX FULL SCAN           | DEPT_ID_PK        |      1 |     27 |00:00:00.01 |       1 |
|*  5 |     INDEX RANGE SCAN          | EMP_DEPARTMENT_IX |     27 |    106 |00:00:00.01 |       3 |
|*  6 |    TABLE ACCESS BY INDEX ROWID| EMPLOYEE          |    106 |     20 |00:00:00.01 |       9 |
----------------------------------------------------------------------------------------------------
 
Query Block Name / Object Alias (identified by operation id):
-------------------------------------------------------------
 
   1 - SEL$51F12574
   4 - SEL$51F12574 / D@SEL$2
   5 - SEL$51F12574 / E@SEL$1
   6 - SEL$51F12574 / E@SEL$1
 
Outline Data
-------------
 
  /*+
      BEGIN_OUTLINE_DATA
      IGNORE_OPTIM_EMBEDDED_HINTS
      OPTIMIZER_FEATURES_ENABLE('11.2.0.1')
      DB_VERSION('11.2.0.1')
      OPT_PARAM('_eliminate_common_subexpr' 'false')
      FIRST_ROWS(1)
      OUTLINE_LEAF(@"SEL$51F12574")
      MERGE(@"SEL$73523A42")
      OUTLINE(@"SEL$1")
      OUTLINE(@"SEL$73523A42")
      ELIMINATE_OBY(@"SEL$2")
      OUTLINE(@"SEL$2")
      INDEX(@"SEL$51F12574" "D"@"SEL$2" ("DEPARTMENT"."DEPARTMENT_ID"))
      INDEX(@"SEL$51F12574" "E"@"SEL$1" ("EMPLOYEE"."DEPARTMENT_ID"))
      LEADING(@"SEL$51F12574" "D"@"SEL$2" "E"@"SEL$1")
      USE_NL(@"SEL$51F12574" "E"@"SEL$1")
      NLJ_BATCHING(@"SEL$51F12574" "E"@"SEL$1")
      END_OUTLINE_DATA
  */
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   5 - access("E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID")
   6 - filter("E"."JOB_ID"='ST_CLERK')
 
Note
-----
   - cardinality feedback used for this statement
```

#### trace 10053
* order by removed 로 OBYE 가 수행됨을 확인
* OBYE 가 수행된 결과로 새로운 쿼리블럭인 SEL$73523A42 가 생성되었다.
* 새로운 쿼리블럭 SEL$73523A42 의 From 절에 DEPARTMENT "D" 가 있음을 확인.
* OBYE 파라미터 _optimizer_order_by_eliment_enabled 이며 default = true
* ELIMINATE_OBY/NO_ELIMINATE_OBY 힌트를 사용하여 해당 기능을 Control 할 수 있다.
```sql
OBYE:   Considering Order-by Elimination from view SEL$1 (#0)
***************************
Order-by elimination (OBYE)
***************************
OBYE:   Considering Order-by Elimination from view SEL$2 (#0)
***************************
Order-by elimination (OBYE)
***************************
OBYE: Removing order by from query block SEL$2 (#0) (order not used)
Registered qb: SEL$73523A42 0x1de21080 (ORDER BY REMOVED FROM QUERY BLOCK SEL$2; SEL$2)
---------------------
QUERY BLOCK SIGNATURE
---------------------
  signature (): qb_name=SEL$73523A42 nbfros=1 flg=0
    fro(0): flg=0 objn=95044 hint_alias="D"@"SEL$2"

OBYE:     OBYE performed.
```

</br>

## 2.6. DE (Distinct Eliminate)
#### 불필요한 Distinct 를 제거하라\
* DE는 select 절에 Unique 한 컬럼이 포함된 경우, 불필요한 Distinct 를 제거하는 기능.
* 이로 인해 Sort , 중복제거 부하가 많은 작업을 수행하지 않을 수 있다. (이 기능은 11g 에서 추가되었다)
```sql
SELECT /*+ GATHER_PLAN_STATISTICS USE_NL(l) */  distinct d.DEPARTMENT_ID, l.location_id
  FROM department d, LOCATION l
 WHERE d.location_id = l.location_id ;
 
SELECT * FROM 
TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL, 'allstats last -rows +alias +outline +predicate')); 
```
* 실행계획에서 Distinct Operation 인 Sort Unique , Hash Unique 가 없음.
* Transformer가 위 SQL을 분석, Distinct 가 없어도 Unique 함을 확인함.
* d.department_id , l.location_id 는 두 테이블의 PK. PK 컬럼이 SELECT 절에 포함되면 Unique 하므로 Distinct operation 이 삭제된다.
```sql
PLAN_TABLE_OUTPUT
----------------------------------------------------------------------------------
| Id  | Operation          | Name       | Starts | A-Rows |   A-Time   | Buffers |
----------------------------------------------------------------------------------
|   0 | SELECT STATEMENT   |            |      1 |     27 |00:00:00.01 |      12 |
|   1 |  NESTED LOOPS      |            |      1 |     27 |00:00:00.01 |      12 |
|   2 |   TABLE ACCESS FULL| DEPARTMENT |      1 |     27 |00:00:00.01 |       8 |
|*  3 |   INDEX UNIQUE SCAN| LOC_ID_PK  |     27 |     27 |00:00:00.01 |       4 |
----------------------------------------------------------------------------------
 
Query Block Name / Object Alias (identified by operation id):
-------------------------------------------------------------
 
   1 - SEL$1
   2 - SEL$1 / D@SEL$1
   3 - SEL$1 / L@SEL$1
 
Outline Data
-------------
 
  /*+
      BEGIN_OUTLINE_DATA
      IGNORE_OPTIM_EMBEDDED_HINTS
      OPTIMIZER_FEATURES_ENABLE('11.2.0.1')
      DB_VERSION('11.2.0.1')
      OPT_PARAM('_eliminate_common_subexpr' 'false')
      FIRST_ROWS(1)
      OUTLINE_LEAF(@"SEL$1")
      FULL(@"SEL$1" "D"@"SEL$1")
      INDEX(@"SEL$1" "L"@"SEL$1" ("LOCATION"."LOCATION_ID"))
      LEADING(@"SEL$1" "D"@"SEL$1" "L"@"SEL$1")
      USE_NL(@"SEL$1" "L"@"SEL$1")
      END_OUTLINE_DATA
  */
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   3 - access("D"."LOCATION_ID"="L"."LOCATION_ID")
```

#### 10053 trace 
* SEL$1 에서 Distinct 가 삭제되었음.
* DE 가 OBYE 자리에서 수행됨을 확인할 수 있다. - 실행순서 상 OBYE 먼저 발생, 이후에 DE 수행됨.
* (DE가 OBYE에 포함되는 개념은 아니다)
```sql
OBYE:   Considering Order-by Elimination from view SEL$1 (#0)
***************************
Order-by elimination (OBYE)
***************************
OBYE:     OBYE bypassed: no order by to eliminate.
Eliminated SELECT DISTINCT from query block SEL$1 (#0)
JE:   Considering Join Elimination on query block SEL$1 (#0)    **  Distinct 가 삭제되었음
```
#### Unique 하다고 항상 DE 가 발생하지는 않는다! (제약사항)
* 컬럼 하나만 변경 ( l.location_id ) 되었는데 Hash Unique 가 수행되었다.
  * 이유 : 논리적으로 select 절에 d.location_id 을 사용하거나, l.location_id 사용하는 것은 같으나, DE에 해당 기능은 제공되지 않는다.
* 조인된 컬럼을 사용할 것인지, 참조되는 컬럼을 사용할 것인지 신중하게 결정해야 한다.
* department 쪽 PK Constraint 를 Drop 하고 Unique index 만 존재한다면 DE가 발생하지 않는다. ( Null 데이터가 중복될 수 있기 때문 )
  *  not null 제약조건을 추가하면 DE가 발생한다. ( 1:N 관계에서 N쪽은 Not Null 조건이 필요하다 )
```sql
SELECT /*+ GATHER_PLAN_STATISTICS USE_NL(l) */ distinct d.department_id, d.location_id
    FROM department d, location l
 WHERE d.location_id = l.location_id ;
 
SELECT * FROM 
TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL, 'allstats last -rows +alias +outline +predicate'));  
```
```sql
PLAN_TABLE_OUTPUT
-------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                | Name             | Starts | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
-------------------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT         |                  |      1 |     27 |00:00:00.01 |      12 |       |       |          |
|   1 |  HASH UNIQUE             |                  |      1 |     27 |00:00:00.01 |      12 |  1270K|  1270K| 1340K (0)|
|   2 |   NESTED LOOPS           |                  |      1 |     27 |00:00:00.01 |      12 |       |       |          |
|   3 |    VIEW                  | index$_join$_001 |      1 |     27 |00:00:00.01 |       8 |       |       |          |
|*  4 |     HASH JOIN            |                  |      1 |     27 |00:00:00.01 |       8 |  1096K|  1096K| 1449K (0)|
|   5 |      INDEX FAST FULL SCAN| DEPT_ID_PK       |      1 |     27 |00:00:00.01 |       4 |       |       |          |
|   6 |      INDEX FAST FULL SCAN| DEPT_LOCATION_IX |      1 |     27 |00:00:00.01 |       4 |       |       |          |
|*  7 |    INDEX UNIQUE SCAN     | LOC_ID_PK        |     27 |     27 |00:00:00.01 |       4 |       |       |          |
-------------------------------------------------------------------------------------------------------------------------
 Predicate Information (identified by operation id):
---------------------------------------------------
 
   4 - access(ROWID=ROWID)
   7 - access("D"."LOCATION_ID"="L"."LOCATION_ID")
```

</br>

#### DE의 또 다른 유형 DEUI (Distinct Elimination using Unique Index)
* Unique 인덱스를 사용하여 INDEX UNIQUE SCAN Operation 이 나오는 경우 Distinct 를 제거한다.
```sql
explain plan for
SELECT DISTINCT d.department_id, l.city, l.country_id
  FROM department d, location l
 WHERE d.location_id = l.location_id
   AND d.department_id = 10 ;


select * from table(dbms_xplan.display);
```

```sql
PLAN_TABLE_OUTPUT
-------------------------------------------------------------------------------------------
| Id  | Operation                    | Name       | Rows  | Bytes | Cost (%CPU)| Time     |
-------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |            |     1 |    22 |     3  (34)| 00:00:01 |
|   1 |  NESTED LOOPS                |            |     1 |    22 |     2   (0)| 00:00:01 |
|   2 |   TABLE ACCESS BY INDEX ROWID| DEPARTMENT |     1 |     7 |     1   (0)| 00:00:01 |
|*  3 |    INDEX UNIQUE SCAN         | DEPT_ID_PK |     1 |       |     0   (0)| 00:00:01 |
|   4 |   TABLE ACCESS BY INDEX ROWID| LOCATION   |    23 |   345 |     1   (0)| 00:00:01 |
|*  5 |    INDEX UNIQUE SCAN         | LOC_ID_PK  |     1 |       |     0   (0)| 00:00:01 |
-------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   3 - access("D"."DEPARTMENT_ID"=10)
   5 - access("D"."LOCATION_ID"="L"."LOCATION_ID")
```

* Location 쪽 unique 컬럼이 select 절에 없음에도 Sort Unique, Hash Unique 가 실행되지 않는다.
  * department, location 두 테이블에 모두 index unique scan 을 사용하기 때문에 결과가 1건 혹은 0을 보장한다.
* 이는 DE기능이 아니다. DE 파라미터로 Control 할 수 없고 수행 로직이 다르기 때문.
* DE : Index unique scan 과 상관없이 select 절에 각 테이블 unique 컬럼만 있으면 수행
* DEUI : plan 상 operation 이 index unique scan 이면 언제든 수행 가능함. 그러나 Unique 인덱스 사용하지 않으면 DEUI는 수행되지 않음.

#### SQL에서 FULL 힌트를 추가할 경우
```sql 
explain plan for
SELECT /*+ FULL(d) */ DISTINCT d.department_id, l.city, l.country_id
  FROM department d, location l
 WHERE d.location_id = l.location_id
   AND d.department_id = 10 ;


select * from table(dbms_xplan.display);

PLAN_TABLE_OUTPUT
--------------------------------------------------------------------------------------------
| Id  | Operation                     | Name       | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT              |            |     1 |    22 |     5  (20)| 00:00:01 |
|   1 |  HASH UNIQUE                  |            |     1 |    22 |     5  (20)| 00:00:01 |
|   2 |   NESTED LOOPS                |            |       |       |            |          |
|   3 |    NESTED LOOPS               |            |     1 |    22 |     4   (0)| 00:00:01 |
|*  4 |     TABLE ACCESS FULL         | DEPARTMENT |     1 |     7 |     3   (0)| 00:00:01 |
|*  5 |     INDEX UNIQUE SCAN         | LOC_ID_PK  |     1 |       |     0   (0)| 00:00:01 |
|   6 |    TABLE ACCESS BY INDEX ROWID| LOCATION   |    23 |   345 |     1   (0)| 00:00:01 |
--------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   4 - filter("D"."DEPARTMENT_ID"=10)
   5 - access("D"."LOCATION_ID"="L"."LOCATION_ID")
```
* department 테이블을 FULL SCAN 하니 Plan 상에 HASH UNIQUE 가 발생.
### DE 수행 유무 - SELECT 절에 Unique 컬럼의 존재 유무에 따라 결정됨
#### DEUI 수행 유무 - Access Path 가 INDEX UNIQUE SCAN 인지 아닌지에 따라 결정됨
> "원인 없는 결과는 없다"
  > 서브쿼리 Unnesting 되어 Driving 집합이되면, 메인쿼리 집합 보존을 위해 default 로 distinct 가 추가되는데,
이 때 DE나 DEUI 기능이 수행되면 distinct 가 제거된다.  
  > (서브쿼리 Unnesting 은 서브쿼리를 인라인뷰로 바꿔 정상적으로 조인으로 만드는 기능.)
* DE는 10.2.0.4 에서도 수행되나, _optimizer_distinct_elimination 파라미터가 없는 것이 11g와 다른 점이다.

</br>
## 2.7 CNT (Count(column) To Count(*))
#### Count(컬럼) 사용시 해당 컬럼이 Not Null인 경우 Count( * )로 대체하라
* CNT 기능 : Not Null 컬럼임에도 Count(컬럼) 을 사용하는 경우가 있는데, Logical OPtimizer는 Count( * ) 를 SQL로 바꾼다.
```sql
CREATE INDEX EMP_IDX_02 ON EMPLOYEE (JOB_ID, DEPARTMENT_ID);

SELECT /*+ GATHER_PLAN_STATISTICS */ e.department_id, COUNT (e.last_name) cnt
  FROM employee e
 WHERE e.job_id = 'ST_CLERK'
 GROUP BY e.department_id;

SELECT * FROM 
TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL,NULL, 'allstats last -rows +alias +outline +predicate'));  
```
* 인덱스 : JOB_ID + DEPARTMENT_ID
* Count 함수에서 LAST_NAME 컬럼을 사용하여 테이블 액세스가 있어야 하나, CNT 기능으로 count( * ) 로 바뀌어 인덱스만 Scan 하였다
```sql
PLAN_TABLE_OUTPUT
------------------------------------------------------------------------------------
| Id  | Operation            | Name       | Starts | A-Rows |   A-Time   | Buffers |
------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT     |            |      1 |      1 |00:00:00.01 |       1 |
|   1 |  SORT GROUP BY NOSORT|            |      1 |      1 |00:00:00.01 |       1 |
|*  2 |   INDEX RANGE SCAN   | EMP_IDX_02 |      1 |     20 |00:00:00.01 |       1 |
------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   2 - access("E"."JOB_ID"='ST_CLERK')
```
#### 10053 trace
```sql
CNT:   Considering count(col) to count(*) on query block SEL$1 (#0)
*************************
Count(col) to Count(*) (CNT)
*************************
CNT:     Converting COUNT(LAST_NAME) to COUNT(*).
CNT:     COUNT() to COUNT(*) done.
```
#### CNT 기능은 항상 동작하는 것인가 ?
* FIRST_NAME 컬럼이 Null허용 컬럼이기 때문에 CNT 가 작동하지 않음
* 물리모델링 시 Not Null 컬럼이라면 Constraint 를 명시적으로 지정해야 한다.
* 가능한한 Count( * ) 를 사용하라. Not null constraint 가 있는 컬럼을 count 에 사용하지 않아야 한다.
#### CNT 기능이 있으므로 별 상관이 없다고 생각할 수 있지만 가장 좋은 것은 SQL이 완벽하여 Transpormation 기능이 추가적으로 실행되지 않도록 하는 것이다.
#### 또한 Count(컬럼)은 어떠한 경우에서도 Count( * )보다 빠를 수 없다.
```sql
explain plan for
SELECT e.department_id, COUNT (e.first_name) cnt
  FROM employee e
 WHERE e.job_id = 'ST_CLERK'
 GROUP BY e.department_id;

select * from table(dbms_xplan.display);

PLAN_TABLE_OUTPUT
-------------------------------------------------------------------------------------------
| Id  | Operation                    | Name       | Rows  | Bytes | Cost (%CPU)| Time     |
-------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |            |     1 |    19 |     2   (0)| 00:00:01 |
|   1 |  SORT GROUP BY NOSORT        |            |     1 |    19 |     2   (0)| 00:00:01 |
|   2 |   TABLE ACCESS BY INDEX ROWID| EMPLOYEE   |     1 |    19 |     2   (0)| 00:00:01 |
|*  3 |    INDEX RANGE SCAN          | EMP_IDX_02 |    20 |       |     1   (0)| 00:00:01 |
-------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   3 - access("E"."JOB_ID"='ST_CLERK')
```

## 2.8 FPD (Filter Push Down)
#### 조건절을 뷰 내부로 이동시켜라
* FPD는 뷰, 인라인뷰 밖에서 뷰 내부 컬럼을 이용하여 조건절을 사용할 때, 조건절이 뷰 내부로 침투되는 현상.
  * 조건절의 a.job_id = 'MK_REP' 구문이 인라인뷰 조건절로 들어가는 것
```sql
explain plan for
SELECT /*+ GATHER_PLAN_STATISTICS */ a.employee_id, a.first_name, 
       a.last_name, a.email, 
       b.department_name
  FROM (SELECT /*+ NO_MERGE */
               employee_id, first_name, last_name, job_id, email,
               department_id
          FROM employee) a,
       department b
 WHERE a.department_id = b.department_id 
   AND a.job_id = 'MK_REP';   

select * from table(dbms_xplan.display);

PLAN_TABLE_OUTPUT
Plan hash value: 1518082028
 
---------------------------------------------------------------------------------------------
| Id  | Operation                      | Name       | Rows  | Bytes | Cost (%CPU)| Time     |
---------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT               |            |     1 |    74 |     3   (0)| 00:00:01 |
|   1 |  NESTED LOOPS                  |            |       |       |            |          |
|   2 |   NESTED LOOPS                 |            |     1 |    74 |     3   (0)| 00:00:01 |
|   3 |    VIEW                        |            |     1 |    58 |     2   (0)| 00:00:01 |
|   4 |     TABLE ACCESS BY INDEX ROWID| EMPLOYEE   |     1 |    39 |     2   (0)| 00:00:01 |
|*  5 |      INDEX RANGE SCAN          | EMP_IDX_02 |     1 |       |     1   (0)| 00:00:01 |
|*  6 |    INDEX UNIQUE SCAN           | DEPT_ID_PK |     1 |       |     0   (0)| 00:00:01 |
|   7 |   TABLE ACCESS BY INDEX ROWID  | DEPARTMENT |     1 |    16 |     1   (0)| 00:00:01 |
---------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   5 - access("JOB_ID"='MK_REP')
   6 - access("A"."DEPARTMENT_ID"="B"."DEPARTMENT_ID")
```


* 인라인뷰에 NO_MERGE 힌트 사용 이유 : SVM (Simple View Merging) 을 피하기 위해.
SVM 발생 시 뷰가 해체되므로, 인라인 뷰로 파고드는 Filter 를 관찰할 수 없기 때문이다.  
* a.job_id = 'MK_REP' 조건이 뷰 내부로 파고들었다. 그 결과 employee 테이블의 job_id 인덱스를 사용하였다.

#### 10053 trace
* 쿼리블럭 SEL$2 (인라인뷰) 에 FPD를 고려, 인라인뷰 내부에 조건절을 생성하였다.
* predicate Move-Around Title 에서 FPD 발생 확인할 수 있다. ( 특정한 Title 이 없기 때문에 PM 이 끝나고 FPD 가 실행됨 )

```sql
**************************
Predicate Move-Around (PM)
**************************
(생략)
FPD: Considering simple filter push in query block SEL$2 (#0)
"EMPLOYEE"."JOB_ID"='MK_REP'
try to generate transitive predicate from check constraints for query block SEL$2 (#0)
finally: "EMPLOYEE"."JOB_ID"='MK_REP'
```

* Transformer 가 SQL을 아래처럼 변경함
```sql
explain plan for
SELECT a.employee_id, a.first_name, a.last_name, a.email, b.department_name
  FROM (SELECT /*+ NO_MERGE */
               employee_id, first_name, last_name, job_id, email,
               department_id
          FROM employee
         WHERE job_id = 'MK_REP') a,
       department b
 WHERE a.department_id = b.department_id;

select * from table(dbms_xplan.display);
```

#### 위 SQL에서 인라인뷰 내에서 Group By 나 Distinct 를 사용하지 않음 ( 이러한 인라인 뷰를 Simple View 라고 함 )
* Simple view 에 적용된 FPD = Simple FPD
* 10053 Trace 에서 "simple filter push" 항목으로 확인할 수 있다.
* Group by, 집계함수 사용 시 이러한 Transformation 발생 = Cost Based Predicate Push Down 이라 한다. ( 3.1장 )
* Simple view 라 하더라도, 인라인뷰 내부에서 rownum 사용 / rank 분석함수 사용하면 FPD기능 사용할 수 없다.
```sql
explain plan for
SELECT a.employee_id, a.first_name, a.last_name, a.email, b.department_name,
       a.salary_rank
  FROM (SELECT /*+ NO_MERGE */
               employee_id, first_name, last_name, job_id, email, department_id, 
               RANK () OVER (ORDER BY salary) salary_rank
          FROM employee) a,
       department b
 WHERE a.department_id = b.department_id 
   AND a.job_id = 'MK_REP';
   
select * from table(dbms_xplan.display);
```
```sql
PLAN_TABLE_OUTPUT
--------------------------------------------------------------------------------------------
| Id  | Operation               | Name             | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT        |                  |   104 | 10608 |     7  (29)| 00:00:01 |
|*  1 |  HASH JOIN              |                  |   104 | 10608 |     7  (29)| 00:00:01 |
|   2 |   VIEW                  | index$_join$_003 |    27 |   432 |     3  (34)| 00:00:01 |
|*  3 |    HASH JOIN            |                  |       |       |            |          |
|   4 |     INDEX FAST FULL SCAN| DEPT_ID_PK       |    27 |   432 |     1   (0)| 00:00:01 |
|   5 |     INDEX FAST FULL SCAN| DEPT_NAME_IDX    |    27 |   432 |     1   (0)| 00:00:01 |
|*  6 |   VIEW                  |                  |   107 |  9202 |     4  (25)| 00:00:01 |
|   7 |    WINDOW SORT          |                  |   107 |  4601 |     4  (25)| 00:00:01 |
|   8 |     TABLE ACCESS FULL   | EMPLOYEE         |   107 |  4601 |     3   (0)| 00:00:01 |
--------------------------------------------------------------------------------------------
 
Predicate Information (identified by operation id):
---------------------------------------------------
 
   1 - access("A"."DEPARTMENT_ID"="B"."DEPARTMENT_ID")
   3 - access(ROWID=ROWID)
   6 - filter("A"."JOB_ID"='MK_REP')    
```
#### 인라인 뷰 내부에서 분석함수 (rank) 사용 시 FPD 기능 수행되지 않음을 확인할 수 있다.
* a.job_id = 'MK_REP' 조건이 view 외부로 밀려남

</br>

## TP(Transitive Predicate or Transitive Closure)
#### 조인절을 이용하여 다른 테이블에 상수조건을 생성시켜라

```sql
SELECT /*+ gather_plan_statistics */
       e.*
  FROM employee e, department d
 WHERE e.department_id = d.department_id                  --DEPARTMENT_ID로 조인
   AND e.department_id = 50;                              --d.department_id ='50'으로 바꾸어도 논리적으로 지장이없다.

select * from table(dbms_xplan.display_cursor(null,null,'allstats last'));
------------------------------------------------------------------------------------------------------------
| Id  | Operation                    | Name              | Starts | E-Rows | A-Rows |   A-Time   | Buffers |
------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |                   |      1 |        |     45 |00:00:00.01 |      14 |
|   1 |  NESTED LOOPS                |                   |      1 |     45 |     45 |00:00:00.01 |      14 |
|*  2 |   INDEX UNIQUE SCAN          | DEPT_ID_PK        |      1 |      1 |      1 |00:00:00.01 |       1 |
|   3 |   TABLE ACCESS BY INDEX ROWID| EMPLOYEE          |      1 |     45 |     45 |00:00:00.01 |      13 |
|*  4 |    INDEX RANGE SCAN          | EMP_DEPARTMENT_IX |      1 |     45 |     45 |00:00:00.01 |       6 |
------------------------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

2 - access("D"."DEPARTMENT_ID"=50)
4 - access("E"."DEPARTMENT_ID"=50)
```
* Predicate Information에 (D.DEPARTMENT_ID=50)조건 생성

</br>

#### 변환된 SQL
```sql
SELECT /*+ gather_plan_statistics */
       e.*
  FROM employee e, department d
 WHERE e.department_id = d.department_id
   AND e.department_id = '50'
   AND d.department_id='50';
```

#### 10053 trace
```sql
**************************
Predicate Move-Around (PM)
**************************
PM:     PM bypassed: Outer query contains no views.
PM:     PM bypassed: Outer query contains no views.
query block SEL$1 (#0) unchanged
FPD: Considering simple filter push in query block SEL$1 (#0)
"E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID" AND "E"."DEPARTMENT_ID"=50
try to generate transitive predicate from check constraints for query block SEL$1 (#0)
finally: "E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID" AND "E"."DEPARTMENT_ID"=50 AND "D"."DEPARTMENT_ID"=50

FPD:   transitive predicates are generated in query block SEL$1 (#0)
"E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID" AND "E"."DEPARTMENT_ID"=50 AND "D"."DEPARTMENT_ID"=50 --D.DEPARTMENT_ID=50조건이 추가적으로 생성
```

#### _optimizer_transitivity_retain를 False로 적용할 경우 Join절이 없어진다.
* _optimizer_transitivity_retain : Transitive Closure에 의해 조인 조건이 없어지는 현상의 방지여부 지정
```sql
ALTER SESSION SET "_optimizer_transitivity_retain" = FALSE;

ALTER SESSION SET EVENTS '10053 trace name context forever, level 1';

select /*+ gather_plan_statistics */
       e.EMPLOYEE_ID, e.EMAIL, e.HIRE_DATE
  from EMPLOYEE e, DEPARTMENT  d
 where e.DEPARTMENT_ID = d.DEPARTMENT_ID
   and e.DEPARTMENT_ID = '50'

ALTER SESSION SET EVENTS '10053 trace name context off';

**************************
Predicate Move-Around (PM)
**************************
PM:     PM bypassed: Outer query contains no views.                       -->PM실패 no views 
PM:     PM bypassed: Outer query contains no views.
query block SEL$1 (#0) unchanged
FPD: Considering simple filter push in query block SEL$1 (#0)
"E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID" AND "E"."DEPARTMENT_ID"=50        -->조인조건이 존재함
try to generate transitive predicate from check constraints for query block SEL$1 (#0)
finally: "D"."DEPARTMENT_ID"=50 AND "E"."DEPARTMENT_ID"=50                -->조인절이 없어지고 상수 조건만 존재
```
* Operator Transformation이나 Tansive Predicate는 PM(Predicate Move Around)이 아니다.
* Predicate Move-Around(PM)자리에 Transitive Predicate가 발생했지만 PM은 발생하지 않았다.(no view) (2.15장에서 설명)
* PM은 2.15장에서 자세히 설명

</br>

## 2.10 SVM(Simple View Merging)
#### Simple View를 해체하여 메인 쿼리와 통합하라
* simple view : 뷰(혹은 인라인뷰) 내부에 Group By나 Distinct 등이 없는 것
* SVM : 오라클 Transformer가 simple view를 만났을때 view를 해체하고 메인 쿼리와 통합하는 작업
* CVM : view내부에 Distinct나 Group By를 사용하는 경우에도 View Merging(뷰의 해체 작업)이 일어남 (3.8장에서 설명)

```sql
ALTER SESSION SET EVENTS '10053 trace name context forever, level 1';

   
select A.EMPLOYEE_ID, A.FIRST_NAME, A.LAST_NAME, A.EMAIL, B.DEPARTMENT_ID
  from EMPLOYEE  a,
       (select B.DEPARTMENT_ID, B.DEPARTMENT_NAME
          from DEPARTMENT b
         where B.DEPARTMENT_ID = :v_deptno ) b
 where a.DEPARTMENT_ID = b.DEPARTMENT_ID;    


ALTER SESSION SET EVENTS '10053 trace name context off';

============
Plan Table
============
---------------------------------------------------------+-----------------------------------+
| Id  | Operation                     | Name             | Rows  | Bytes | Cost  | Time      |
---------------------------------------------------------+-----------------------------------+
| 0   | SELECT STATEMENT              |                  |       |       |     1 |           |
| 1   |  NESTED LOOPS                 |                  |    10 |   330 |     0 |           |
| 2   |   INDEX UNIQUE SCAN           | DEPT_ID_PK       |     1 |     3 |     0 |           |
| 3   |   TABLE ACCESS BY INDEX ROWID | EMPLOYEE         |    10 |   300 |     0 |           |
| 4   |    INDEX RANGE SCAN           | EMP_DEPARTMENT_IX|    10 |       |     0 |           |
---------------------------------------------------------+-----------------------------------+

*** 2013-06-11 14:52:17.132
Predicate Information:
----------------------
2 - access("B"."DEPARTMENT_ID"=TO_NUMBER(:V_DEPTNO))
4 - access("A"."DEPARTMENT_ID"=TO_NUMBER(:V_DEPTNO))
```
* Plan상에서 View가 사라졌다. Transformer가 SQL을 아래처럼 바꿈

</br>

#### SQL 변환
```sql
SELECT a.employee_id, a.first_name, a.last_name, a.email, b.department_id
  FROM employee a, department b
 WHERE a.department_id = b.department_id
   AND b.department_id = :v_deptno
   AND a.department_id = :v_deptno;  -- View Merging이 발생함으로써 Transitive Predicate(조건절 전이)가 발생하였다.
```
* 인라인뷰 b 내부의 DEPATMENT_NAME을 Select하려면 DEPARTMENT 테이블을 Scan하는 Operation이 필요한데 그에 해당하는 Operation이 존재하지 않는다.
* 이러한 현상은 SVM 과정에서 인라인뷰 외부에서 사용하지 않는 컬럼들을 Transformaer가 제거해 버리기 때문에 발생한다.
* 이기능은 SVM이 수행될 때 부가적으로 발생되는 기능이다.

</br>

#### Merge
```sql
outline Data:
     /*+
          BEGIN_Outline_DATA
               ... 중간생략
               MERGE(@"SEL$2")
               ...중간생략
          END_Outline_DATA
     */
```
* 오라클이 내부적으로 Merge 힌트를 사용한 것을 알 수 있다.
* 오라클은 이와같이 내부적인 힌트를 사용함으로써 Transformation을 수행한다.
* DBMS_XPLAN의 outline Data를 분석해보면 많은 경우에 오라클의 내부적인 힌트의 사용을 관찰할 수 있다.

</br>

#### Join Elimination (JE)
```sql
*************************
Join Elimination (JE)
*************************
... 중간생략
CVM: Considering view merge in query block SEL$1 (#0)
OJE: Begin: find best directive for query block SEL$1 (#0)
OJE: End: finding best directive for query block SEL$1 (#0)
CVM:   Checking validity of merging in query block SEL$2 (#0)
CVM: Considering view merge in query block SEL$2 (#0)
OJE: Begin: find best directive for query block SEL$2 (#0)
OJE: End: finding best directive for query block SEL$2 (#0)
CVM:   Merging SPJ view SEL$2 (#0) into SEL$1 (#0)        --SEL$2가 해체되어 SEL$1으로 통합
Registered qb: SEL$F5BB74E1 0x10902870 (VIEW MERGE SEL$1; SEL$2)
```

</br> 

#### Transitive Predicate
* SVM이 수행되면 후속작업으로 Transitive Predicate가 일어난다.
```sql
**************************
Predicate Move-Around (PM)
**************************
PM:     PM bypassed: Outer query contains no views.
PM:     PM bypassed: Outer query contains no views.
query block SEL$F5BB74E1 (#0) unchanged
FPD: Considering simple filter push in query block SEL$F5BB74E1 (#0)
"A"."DEPARTMENT_ID"="B"."DEPARTMENT_ID" AND "B"."DEPARTMENT_ID"=TO_NUMBER(:B1)
try to generate transitive predicate from check constraints for query block SEL$F5BB74E1 (#0)
finally: "A"."DEPARTMENT_ID"=TO_NUMBER(:B1) AND "B"."DEPARTMENT_ID"=TO_NUMBER(:B2)
```
* EMPLOYEE 테이블이 뷰와 조인하지 않고 DEPARTMENT 테이블과 직접 조인이 발생함으로써 Transitive Predicate가 발생될 수 있다

#### Transformer가 View Merge를 시도하는 이유
* View Merger에 의한 또 다른 형태의 Transformation을 발생시키기 위함. JE, OBYE, OJE, Transitive Predicate 등이 View Merge에 의해서 추가로 발생할 수 있기 때문에 이과정은 대단히 중요한 역할을 한다.
  * 뷰 바깥쪽 테이블과 관련된 이러한 변환들은 뷰가 해체되기 전에는 발생될 수가 없다.
* SQL의 복잡성을 회피하여 SQL의 Costing을 쉽게 하자는 것.
  * 복잡성회피 : 쿼리블럭의 개수를 줄여서 성능을 개선하자는 의미. 여러개의 쿼리블럭을 하나로 만들어 뷰를 해체하게 되면 뷰 바깥쪽에 존재하는 테이블과의 조인순서를 Physical Optimizer가 최적으로 조정할 수 있어 성능이 향상된다.











