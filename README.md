# 

# SQLServerTuning

***
## 1. SSMS 사용. 쿼리 작성 클라이언트 툴
  MS에서 만들고 배포하는 SSMS(SQL Server Management Studio)를 사용하며   
  노트북을 지급받을 때 자동으로 깔려있다.  
  만약 설치가 안되어 있을 경우 파일서버의 다음 위치에서 설치본을 찾을수 있다.  
    파일서버\설치파일\SSMS\SSMS_Setup_ENU.exe
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