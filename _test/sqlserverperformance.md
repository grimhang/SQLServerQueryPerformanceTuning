---
sort: 1
---

# SQL Server Performance Tuning
SQL Server 업그레이드는 쿼리 처리 엔진이 바뀌어 더 나은 성능향상이 있기 때문에 중요하다.

* 문제 있는 SQL 쿼리 식별
* 쿼리 실행 계획 분석
* 현재 인덱스 효율 평가
* 북마크 룩업 피하기
* 현재 통계 효율 분석
* 파라메터 스니핑 이해와 해결
* 조각화 분석과 해결
* 실행 계획 캐싱 최적화
* 구문 재컴파일 분석과 피하기
* 블로킹과 데드락의 최소화
* 커서 사용의 효율 분석
* 메모리 테이블 저장소 적용 과 프로시저 실행
* SQL 부하를 최적화 하기 위한 튜닝 절차와 도구, 기술들을 적용하기

이책에서 커버하는 범위
* 성능 튜닝 절차
* 성능과 비용 사이
* 성능 가이드라인
* 튜닝할때 어디에 초점을 맞출지
* 탑 13 SQL Server 성능 킬러들

커버하지 않는 범위
* 하드웨어 선택
* 어플리케이션 코딩 방법론
* 서버 구성(쿼리 튜닝에 영향을 미치는 것 제외)
* SSIS
* SSAS
* SSRS
* 파워쉘

***
## 성능 튜닝 절차
  성능 병목 식별 -> 이슈 우선 순위 정하기 -> 원인 트러블슈팅 -> 다른 해결책 적용 -> 성능 향상 수치화
  이 절차 반복    그리고 약간의 창의성 필요.

    * 핵심 절차
        다음 질문을 스스로에게 해보자
        a. 같은 서버에 SQL Server와 리소스 경쟁하는 다른 어플리케이션이 존재하는지?
            SQL Server보다 우선순위가 높은지 확인. ex) 작업관리자 -> 자세히 -> 컬럼 선택 -> Basic priority 체크
            기본적으로 sqlservr.exe는 노멀 우선순위인데 작업관리자(taskmgr.exe) 프로세스는 높음 우선순위이다.
            우리는 덜 중요한 어플리케이션이나 서비스가 SQL Server와 같은 머신에서 실행되며 자원 사용에 영향을 미치는지 조사 필요.
        b. 하드웨어의 용량이 최대 워크로드를 견딜수 있는 상황인지?
            CPU, 메모리, 디스크, 네트워크
        c SQL Server 구성은 정확한가?
            sys.configuration 확인
            대부분 디폴트 값이 좋지만 일부분 성능에 영향을 미치는 부분은 언급할것. database 구성 옵션도
            model DB에 적용하면 대부분 그 다음에는 만들어지는 것도 그것을 따라감.
        d VM같은 리소스 공유 환경에 내가 영향을 끼쳐 구성을 변경할수 있는지?
        e SQL Server 와 응용프로그램간에 연결은 효율적인지?
            적절한 네트워크 대역폭이 필요. 특히 클라우드 호스팅 환경이면 더 중요.
        f 데이터베이스 디자인은 최적화인지?
            디자인을 수정하는 것은 항상 가능하지 않지만 설계된 디자인을 잘 이해하는 것은 매우 도움이 된다.
        g SQL 쿼리를 구성하는 사용자 워크로드는 SQL Server 로드를 줄이게 최적화 되었는지?
            인덱스와 같은 것들이 적절하게 구성되어야
        h 어떤 프로세스들이 시스템 성능 저하를 일으키는지?
        i 워크로드가 동시성 필요레벨을 지원하는지?
            
        * 절차 반복(Iterating the Process)
            성능 튜닝은 반복 작업 - 주요 병목 식별하여 해결하고 변화에 대한 임팩트를 측정 그리고 다시 처음부터. 성능개선활동이 종료될때까지.
            해결책을 적용할때는 골든 룰이 있다. 가능한 1번에 1가지 변경만 적용하라. 어떤 변경사항이 시스템의 다른 부분에 영향을 미치는 일이 많기 때문.
            예) 특정 쿼리 성능 문제를 해결하기 위해 인덱스를 추가하면 그로 인해 다른 쿼리들이 느려질 수도.
            그렇기에 테스트 서버에서 충분히 테스트를 수행해야 한다.
            데이터와 사용량이 증가하면 기존의 쿼리의 실행계획 등이 변경되어 또 다시 나빠질 수 있기 때문에 주기적으로 성능 측정과 튜닝작업을 반복 필요.
            (그림 존재)
        
## 성능 대 비용 =    
    약간의 성능 향상을 얻기 위해 많이 시간과 돈을 소비할 수도 있기 때문에 목적을 세워야한다.
    - 얼마만큼의 성능이면 받아들일 수 있는지
    - 이 작업이 성능향상의 가치가 있는지?
    
    * 성능 목표
    최대 효율을 얻기 위해서 성능 목표치를 정해야 함. 베스트 프랙티스가 이미 많이 존재하기 때문에 우리에 맞게 적용하기 전 얼마만큼의 수치면 만족할 수 있을지?
    보통 개발자나 비지니스 수행자들들에게 목표를 정하게 하는게 좋음 자세.
    쿼리 튜닝 같은 개선 활동도 적은 비용으로 효과를 얻지만 하드웨어 리소스 투자 같은 방법도 빠르게 성능 향상을 얻을 수 있다.
    단 쿼리 튜닝은 적은 노력으로 큰 효과를 얻지만 전문가가 필요하고 하드웨더 투자는 큰 비용을 들지만 빠른 효과를 얻을수 있다. 잘 판단. 경험치 능력자가 매우 중요.
    
    * "이정도면 충분" 튜닝
    80:20 법칙이 여기서도 통함. 상위 몇개의 악성 쿼리들만 해결하면 80%의 문제가 해결된다. 이 때 노력은 20%만 소요.  최대 효율을 얻기 위해서는 목표치를 세우고
    20%의 노력으로 80%의 문제점을 해결하는 방향으로 하려면 이정도면 충분 방법으로 가야.
    
    들인 비용과 성능 개선과의 관계는 반비례 그래프와 같다.
    초기에는 적은 노력으로 큰 향상을 얻지만 나중에는 큰 노력과 비용으로도 적은 결과만 얻게 되니 중간에 "이정도면 충분" 지점에서 끊어ㄷ야 함.
    
## 성능 베이스라인
    성능 분석의 주요 목표는 여러가지 하드웨어와 소프트웨어 위에서 시스템의 사용과 부하 패턴을 이해하는 것이다.
    이로 인해 아래의 도움을 얻을 수 있다.
    - 리소스 병목 분석 가능
    - 이전에 측정된 베이스라인과 비교하여 얼마만큼 사용량이 증가하여 트러블슈팅할때 이득
    - 리소스 사용량 계획과 하드웨어 업그레이드 주기 정확한 측정 가능
    - 피크타임을 피하여 데이터베이스 관리 활동하는 시간을 정할수 있음.
    - 특정 수치는 이전 값과 비교할 때만 의미가 있을 수 있다. 예) PLE
    
    베이스라인이 있으면 가능한 일
    - 현재 성능을 측정하고 어플리케이션 성능 목표치를 세울수
    - 다른 하드웨어와 소프트웨어 조합을 비교 가능
    - 어떻게 사용량과 데이터가 변화하는지 측정
    - 현재 값이 이상하게 튀는 데 진짜 이상 상황인지 아니면 평소와 같은 정상 수치인지
    - 어플리케이션의 피크와 비 피크 사용 패턴 이해. 효과적인 데이터베이스 관리작업 가능. 예를 들면 비 피크타임 일때 백업이나 조각모음을 한다든지
    
    SQL Server의 하드웨어 소프트웨어 리소스 사용량 베이스라인 만들기 위해 성능모니터 이용할 수 있다. 또는 DMV나 DMF
    Xevent를 사용하여 쿼리 부하치 
    또는 기타 툴 MS의 SQLIO, SQLIOSim(시뮬레이션) or Distributed Replay.
    
## 어디에 노력을 집중 해야? =
    하드웨어와 OS쪽은 얻는 결과가 작고 큰 비용이 들며 인덱스나, 쿼리 코딩쪽은 가장 큰 효과를 본다.    
    (그림존재)
    
    SQL Server 데이터베이스 사용하고 있는 어플리케이션 스트레스를 2단계로 분석해야 한다.
        - 고수준 : 각각의 하드웨어 자원과 SQL Server 설치에 걸쳐 얼마나 많은 스트레스가 있는지 측정. 가장 좋은 것은 다양한 대기 상태를 측정하는 것.
                 이로 인해 두가지 방법으로 도움을 얻는다.
                 첫째, 나쁜 성능의 SQL Server 어플리케이션의 어느 부분에 집중해야 하는지 알수
                 둘째, 적절하지 않은 구성 식별 도움.
                 이로 인해 성능 모니터를 사용하여 어플리케이션을 튜닝할 수 없을 경우 어떤 하드웨어 자원을 업그레이드해야 결정 할때 도움.
                 
        - 저수준 : 범인이 되는 쿼리 식별. Xevent와 DMV 를 통해 알수 있다.
        
## SQL Server 성능 저하 범인 =
    주요 SQL Server 성능 저하 요인들
    - 불충분 인덱스
        보통 가장 큰 원인. CPU, Memory, Disk에 부하 가중.
    - 정확하지 않은 통계
    - 부적절한 쿼리 디자인
    - 나쁜 실행계획
        효율적인 프로시서는 재컴파일하지 않고 생성, 재사용되는데 몇 몇 케이스 동일 메카니즘으로 반복 실행되어도 기대에 어긋나는 결과 발생.
        나쁜 실행계획은 큰 문제 가능. 나쁜 실행계획은 파라메터 스니핑 문제를 발생. 쿼리 옵티마이저가 통계를 바탕으로 실행계획을 만드는 과정에서 발생.
        통계와 실행계획이 어떻게 만들어지는지 이해해야 하고 어떻게 컨트롤 할수 있느지 필요.
    - 과도한 블로킹과 데드락
    - 집합 기준이 아닌 연산. 예 TSQL 커서같은
    - 부적절한 데이터베이스 디자인
    - 과도한 조각화
    - 재사용 불가 실행계획
    - 빈번한 쿼리 재컴파일
    - 부적절한 커서 사용
    - 데이터베이스 트랜잭션 로그의 부적절한 구성
    - tempdb의 과도한 사용 및 부적절한 구성






> 최신 버전 18.2
* #### 1.1 한글 전환 문제
  sql을 작성중 다른 화면으로 이동했다 돌아오면 한글 상태로 자판이 자동 변경되는 문제가 있다.  
  15년 넘은 버그로 해결이 안되고 있다.  
  다음과 같이 임시 조치 가능  
    > 메뉴 / 쿼리 / 쿼리 옵션 창에서 영문으로 자판을 바꾸면 그 다음부터는 영문이 기본값.  

## 2. 기본규칙
* a. 들여쓰기 크기는 4이며 공백4개 기본으로 한다.  
   SMS에 도구/옵션/텍스트편집기/Transact-SQL/탭에서 공백삽입으로 바꾸면됨
* b. 괄호로 AND 와 OR는 명시적으로 지정
* c. 여러 sql 문장을 작성할때는 꼭 1칸의 빈 공백 라인을 준다
* d. 애스터리스크(*) 는 쓰지 않는다. 항상 명시적으로 컬럼 리스트를 지정
* e. MARS(쿼리 1개로 resultset(데이터셋) 을 여러개 받는것) 는 쓰지 않는다.
* f. 되도록 힌트를 쓰지 않는다. (최근에는 DBMS가 예전보다 똑똑해져서 대부분 사람보다 낫다)
    

## 3. 키워드
* a. 모든 SQL 키워드는 대문자  
  예) SELECT, FROM, WHERE, INSERT  
* b. 예약어를 변수명으로 사용하지 않는다.
* c. 모든 sql은 ; (세미콜론)으로 구분

## 4. 별칭
* a. 항상 AS 를 지정  
* b. 별칭은 단어들의 첫째 글자로  
   예) PRODUCT_BUY_CUSTOMER  ⇒ PBC

## 5. DB Object 명명 규칙
* #### a. 테이블
* #### b. PK ==> PK_테이블명   
  예)  PK_OKRB_CTRT

* #### c. FK : FK_테이블명   
  예)  FK_OKRB_CTRT

* #### d. 인덱스 : IX_테이블명_일련번호  
  예) IX_OKRB_CTRT_01

* #### e. 뷰 : VW_기준테이블명_일련번호  
  예) VW_OKRB_CTRT_01
* #### f. 저장 프로시저 : P_의미있는이름_일련번호  
  예) P_OKRB_CTRT_BATCH_01

* #### g. 함수 : FN_의미있는이름_01  
  예) FN_OKRB_CTRT_GET_NAME_01

* #### h. 트리거 : TR_테이블명_01
  예) TR_OKRB_CTRT_01

* #### i. 파티션 : PT_테이블명_파티션키
  예) PT_OKRB_CTRT__PLCY_NO    

* #### j. 시노님 : 테이블명과 동일

* #### k. 시퀀스 : SQ_테이블명
  예) SQ_OKRB_CTRT

## 6. 주석
* #### 6.1 한줄 주석
  대시 두번이면 같은 줄의 이후의 내용은 주석처리됨

* #### 6.2 복수줄 주석
  대량의 내용을 주석으로 하고 싶을때
```sql
    /**************************************************************
    -- Title        : 테스트
    -- Author 	    : 박성출
    -- Create date  : 2012-09-08
    -- Description  : 초기 작성
    
        DATE         	Developer        Change
        ----------   	---------------- --------------------------
        2012-09-08      박성출         	  처음 작성
    **************************************************************/
```

## 7. SELECT
기본 스타일은 다음과 같습니다.
```sql
SELECT MEMBER_NO, MEMBER_ID, MEMBER_NAME, BASIC_SYSTEM_NO, CREATE_DTM
	, MEMBER_ID, MEMBER_NAME
FROM TB_MEMBER
WHERE MEMBER_GROUP_CODE = 'A'
```
  SELECT, FROM, WHERE 절은 다음줄에 배치     
  SELECT절에 한줄의 컬럼 갯수는 5개까지만 하고 6개부터 줄바꿈  

* #### 7.1 SELECT 절의 사용자 정의 함수
    * SELECT에는 가급적 사용자 정의 함수를 사용하지 않는다. 성능 이슈 존재.  
    * 해당 로직을 함수 밖으로 꺼낸다.  
```sql
    -- 나쁜 예
    SELECT MEMBER_NAME, dbo.FN_GET_AGE(MEMBER_NO) AS MEM_AGE
    FROM TB_MEMBER
    WHERE MEMBER_GROUP_CODE = 'A'

    -- 좋은 예
    SELECT MEMBER_NAME, 100 - SUBSTRING(JUMIN_NO AS, 2)  MEM_AGE
    FROM TB_MEMBER
    WHERE MEMBER_GROUP_CODE = 'A'
```

* #### 7.2  컬럼과 컬럼사이의 콤마 규칙 
```sql   
    -- 한줄일때
    SELECT PROD_NO, CUST_NAME

    -- 여러줄일때
    SELECT PROD_NO
        , CUST_NAME
```

* #### 7.3 SELECT, FROM, WHERE, GROUP BY 는 각각 다른 줄에  


* #### 7.4 WHERE 조건은 다른줄에
```sql    
    SELECT *
    FROM TABLE1
    WHERE PROD_NO = 1
        AND CUST_NO = 55552
        AND REGION = ‘서울’
```

* #### 7.6 SELECT 구문의 가로길이는 컬럼 5개  
```sql
    SELECT PROD_NO, CUST_NAME, CUST_NO, CUST_ADDR, CUST_TELNO
        , CUST_ALIAS_NAME
```        
* #### 7.7 변수에 값 할당
  SET을 이용해 변수에 값 할당. 단 테이블에서 데이터를 읽어와 값을 변수에 할당할 경우는 SELECT

```sql
    -- 권장방식
    DECLARE @TEMP_NAME VARCHAR(50)
    SET @TEMP_NAME = '홍길동'
    
    -- 비권장
    DECLARE @TEMP_NAME VARCHAR(50)
    SELECT @TEMP_NAME = '홍길동'

    -- 테이블에서 값을 변수에 할당할때만 SELECT 사용
    DECLARE @TEMP_NAME VARCHAR(50)
    SELECT @TEMP_NAME = MEM_NAME
    FROM MEMBER
    WHERE CUST_NO = 5524
```       


## 8. UPDATE
기본 스타일은 다음과 같습니다.

```SQL
UPDATE T MEMBER
SET MEMBER_NAME = '홍길동'
    , MEMBER_GROUP_CODE = 'C'
WHERE MEMBER_NO = 5
    AND MEMBER_GROUP_CODE = 'A'
```

* #### a. UPDATE, SET, WHERE 는 다른줄에 배치
* #### b. SET절과 WHERE절에서 컬럼들 다른줄에 배치


## 9. INSERT
    컬럼 리스트를 항상 명시
```SQL    
-- 나쁜 예
INSERT INTO TABLE1 VALUES (1, getdate());

-- 좋은 예
INSERT INTO TABLE1 (NO, TODAY_DATE) VALUES (1, getdate());
```

## 10. DELETE
기본 스타일은 다음과 같습니다.
```SQL
DELETE ADDRESS
WHERE ADDRESS_ID = 297;
```
* #### a. DELETE와 WHERE 구문은 다른줄에


## 11. 에러처리
* GOTO 논리를 사용하지 않는다

* null값 비교는 무조건 IS NULL 또는 IS NOT NULL 

* 오류처리는 TRY CATCH 사용
```SQL
DECLARE @TEMP_NAME VARCHAR(50)
BEGIN TRY
    SELECT @TEMP_NAME = MEM_NO
    FROM MEMBER
    WHERE MEM_NO = 23532
END TRY
BEGIN CATCH
    PRINT @ERROR
END CATCH
```

## 12. 조인
조인은 ANSI 조인을 사용
```SQL
SELECT PROD_NO
FROM TABLE1 AS T1
    JOIN TABLE2 AS T2         	ON T1.TNO = T2.TNO
    JOIN CUST_BUY_LIST AS CBL 	ON T2.CUST_NO = CBL.CUST_NO
```  
* #### a. INNER JOIN 은 INNER를 생략
* #### b. LEFT OUTER JOIN, RIGHT OUTER JOIN은 OUTER를 생략  ⇒ LEFT JOIN, RIGHT JOIN, FULL JOIN
* #### c. 조인의 ON 절은 탭으로 1번 들여쓰기
* #### d. JOIN은 1탭 들여쓰기한다.

```SQL
    SELECT PROD_NO
    FROM TABLE1 AS T1
        JOIN TABLE2 AS T2		ON T1.TNO = T2.TNO
        LEFT JOIN TABLE3 AS T3  ON T2.TNAME = T3.TNAME
```

* #### e.  여러 테이블 조인시 항상 최상위 테이블부터. 단 성능 이슈발생시 드라이빙 테이블부터
```SQL
    SELECT C.NAME
    FROM CUST AS C
        JOIN CUST_BUY_LIST AS CBL	ON 
```

## 13. 피해야 할 점
텍스트 형태의 데이터 타입 CHAR, NCHAR, VARCHAR, NVARCHAR 에는 항상 크기를 지정
```SQL
/* 권장 */
 DECLARE @myGoodVarchareVariable  VARCHAR(50);
 DECLARE @myGoodNVarchareVariable NVARCHAR(90);
 DECLARE @myGoodCharVariable      CHAR(7);
 DECLARE @myGoodNCharVariable     NCHAR(10);
 
 /* 비권장 */
 DECLARE @myBadVarcharVariable  VARCHAR;
 DECLARE @myBadNVarcharVariable NVARCHAR;
 DECLARE @myBadCharVariable     CHAR;
 DECLARE @myBadNCharVariable    NCHAR;
```


## 14. 권장
* #### 14.1 TOP으로 결과 제한
  SELECT의 결과셋을 제한할때 TOP을 사용  
```SQL
    SELECT TOP 5 PROJECT_NO, PROJECT_NAME        -- 상위 5개만 가져오는다
    FROM T_PROJECT;

    SELECT TOP 50 PERCENT PROJECT_NO, PROJECT_NAME  -- 결과중 상위 50%만 가져오기
    FROM T_PROJECT;
```

* #### 14.2 서브쿼리
```SQL
    SELECT MEMBER_NO, MEMBER_ID
    FROM TB_MEMBER
    WHERE MEMBER_NO = (SELECT TOP 1 MEMBER_NO
                                            FROM TB_MEMBER
                                            WHERE MEMBER_GROUP_CODE = 'A')


    SELECT M.MEMBER_NO, M.MEMBER_ID, M.MEMBER_NAME
    FROM TB_MEMBER M
        JOIN
        (
            SELECT MEMBER_NO, MEMBER_NAME, MEMBER_GROUP_CODE
            FROM TB_MEMBER
            WHERE MEMBER_GROUP_CODE = 'A'
        ) A    ON M.MEMBER_NO = A.MEMBER_NO
    WHERE M.MEMBER_GROUP_CODE NOT IN ('A')
```

* #### 14.3 @@IDENTITY 사용 금지
  Azure DW는 시퀀스가 지원 안됨. identity를 써야 하는데 @@IDENTITY 대신 SCOPE_IDENTITY() 사용  

* #### 14.4 NOLOCK 힌트 사용 금지
    * Azure SQL Database          
        * read commited snapshot isolation가 기본.
        * 오라클과 같은 MVCC라서 NOLOCK의미 없음

    * Azure SQL Datawarehouse
        * read uncommited가 기본값. nolock과 같은 말


* #### 14.5 데이터 존재 파악
  데이터 존재 여부 파악을 위해 COUNT(*) 또는 SELECT * FROM 을 사용하는 대신 EXSITS/TOP 구문 사용.  
  또한 IN, NOT IN 대신에 가능하면 EXISTS, NOT EXISTS를 사용
```sql
    -- 예제 1. 첫번째 한건만 확인
    IF (EXISTS(SELECT 1 FROM DBO.TB_CUSTID WHERE CUSTNAME LIKE '박%'))

    -- 예제 2. 첫번째 한건의 컬럼만 확인
    IF (EXISTS(SELECT TOP 1 CUSTID FROM DBO.TB_CUSTID WHERE CUSTNAME LIKE '박%'))

    -- 나쁜 예.
    IF (SELECT COUNT(*) FROM DBO.TB_CUSTID WHERE CUSTNAME LIKE ‘박%’)) -- 전체 ROW 를 모두 COUNT
 ```

 * #### 14.6 페이징은 OFFSET FETCH 를 사용
 ```sql
    SELECT PRODUCT_NAME, LIST_PRICE 
    FROM PRODUCT
    ORDER BY LIST_PRICE, PRODUCT_NAME
    OFFSET 10 ROWS 
    FETCH NEXT 10 ROWS ONLY;
 ```

 * #### 14.7 VARCHAR와 NVARCHAR 사이 조인 금지
   VARCHAR쪽 테이블의 모든 로우를 NVARCHAR로 먼저 암시적으로 변환 한다음 조인을 하기 때문에  
   엄청난 성능 저하가 발생  
    > JAVA쪽은 이와 관련되어 기본적으로 쿼리가 UNICODE로 입력되기에 VARCHAR로 테이블을  
    > 만들었을 경우 암시적 변환때문에 성능 저하 발생.  MyBatis가 N'apple' 이렇게 데이터를 보냄  
    > 이에 대한 예방책으로 JDBC 드라이버의 sendStringParametersAsUnicode 값을 false로 하는 방법도 존재  
    > 문제 설명 참고  
    >  --> https://dream6.tistory.com/entry/인덱스를-이용-못하는-경우자바-유니코드-문제?category=582795