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
