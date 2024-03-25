---
sort: 25
comments: true
---

# 행 단위별 프로세싱
커서를 사용하여 한 번에 한 행씩 처리하는 데이터베이스 응용 프로그램을 찾는 것이 일반적입니다. 개발자는 행 단위 방식으로 데이터를 처리하는 것에 대해 생각하는 경향이 있습니다. Oracle은 고속 데이터 액세스 메커니즘으로 커서라는 것을 사용하기도 합니다. SQL Server의 커서는 다릅니다. SQL Server에서 커서를 통한 데이터 조작은 상당한 추가 오버헤드를 발생시키므로 데이터베이스 애플리케이션에서는 커서 사용을 피해야 합니다. T-SQL 및 SQL Server는 한 번에 한 행씩이 아니라 데이터 집합에서 가장 잘 작동하도록 설계되었습니다. Jeff Moden은 이러한 유형의 처리를 RBAR(“ree-bar”로 발음)라고 명명했는데, 이는 행 대 고통스러운 행을 의미합니다. 그러나 커서를 사용해야 하는 경우 비용이 가장 적은 커서를 사용하십시오.

이 기술을 채택할 때는 주의할게 있는데 적절한 양의 메모리를 갖춘 올바른 시스템에서만 인메모리의 테이블, 기본 저장 프로시저를 통해 매우 빠른 속도를 얻을 수 있다는 것이다.

이번 장에서는 다음과 같은 주제를 다룬다.

    * 커서 기초
    * 다른 커서 특성의 비용 분석
    * 커서에 비해 기본 결과 집합의 장점과 단점
    * 커서의 비용 오버헤드를 최소화하기 위한 권장 사항

## <font color='dodgerblue' size="6">1) 커서 기초</font>    
응용 프로그램에서 쿼리를 실행하면 SQL Server는 행으로 구성된 데이터 집합을 반환합니다. 일반적으로 애플리케이션은 여러 행을 함께 처리할 수 없습니다. 대신 SQL Server에서 반환된 결과 집합을 살펴보며 한 번에 한 행씩 처리합니다. 이 기능은 여러 행 결과 집합에서 한 번에 한 행씩 작업하는 메커니즘인 커서에 의해 제공됩니다.

T-SQL 커서 처리에는 일반적으로 다음 단계가 포함됩니다. 
    1. 커서를 선언하여 SELECT 문과 연결하고 커서의 특성을 정의합니다. 2. 커서를 열어 SELECT 문에서 반환된 결과 집합에 액세스합니다. 
    3. 커서에서 행을 검색합니다. 선택적으로 커서를 통해 행을 수정합니다. 
    4. 결과 집합의 추가 행으로 이동합니다. 
    5. 결과 집합의 모든 행이 처리되면 커서를 닫고 커서에 할당된 리소스를 해제합니다. T-SQL 문이나 SQL Server에 연결하는 데 사용되는 데이터 액세스 계층을 사용하여 커서를 만들 수 있습니다. 데이터 액세스 계층을 사용하여 생성된 커서를 일반적으로 클라이언트 커서라고 합니다. T-SQL로 작성된 커서를 서버 커서라고 합니다. 다음은 테이블의 쿼리 결과를 처리하는 서버 커서의 예입니다.

```sql
--Associate a SELECT statement to a cursor and define the
--cursor's characteristics
USE AdventureWorks2017;
GO
SET NOCOUNT ON
DECLARE MyCursor CURSOR /*<cursor characteristics>*/
FOR
SELECT adt.AddressTypeID,
 adt.Name,
 adt.ModifiedDate
FROM Person.AddressType AS adt;
--Open the cursor to access the result set returned by the
--SELECT statement
OPEN MyCursor;
--Retrieve one row at a time from the result set returned by
--the SELECT statement
DECLARE @AddressTypeId INT,
 @Name VARCHAR(50),
 @ModifiedDate DATETIME;
FETCH NEXT FROM MyCursor
INTO @AddressTypeId,
 @Name,
 @ModifiedDate;
WHILE @@FETCH_STATUS = 0
BEGIN
 PRINT 'NAME = ' + @Name;
 --Optionally, modify the row through the cursor
 UPDATE Person.AddressType
 SET Name = Name + 'z'
 WHERE CURRENT OF MyCursor;
 --Move through to additional rows in the data set
 FETCH NEXT FROM MyCursor
 INTO @AddressTypeId,
 @Name,
 @ModifiedDate;
END
--Close the cursor and release all resources assigned to the
--cursor
CLOSE MyCursor;
DEALLOCATE MyCursor;
```

커서의 오버헤드 중 일부는 커서 특성에 따라 달라집니다. SQL Server에서 제공하는 커서와 데이터 액세스 계층의 특성은 크게 세 가지 범주로 분류할 수 있습니다. • 커서 위치: 커서 생성 위치를 정의합니다. • 커서 동시성: 커서와 기본 콘텐츠의 격리 및 동기화 정도를 정의합니다. • 커서 유형: 커서의 특정 특성을 정의합니다.

커서의 비용을 살펴보기 전에 커서의 다양한 특징을 몇 페이지에 걸쳐 소개하겠습니다. 다음 쿼리를 사용하여 Person.AddressType 테이블에 대한 변경 사항을 실행 취소할 수 있습니다. UPDATE Person.AddressType SET Name = LEFT(Name, LEN(Name) - 1);

- ### a. 커서의 위치
    메모리 최적화 테이블이 가능한지 여부를 고려하기 전 먼저 몇 가지 표준 요구 사항을 충족해야 한다.
        * 모던 64bit 프로세스
        * 메모리에 넣으려는 데이터의 두배 디스크 공간
        * 당연하게도 많은 메모리
    
    분명히 대부분의 시스템에서 핵심은 많은 메모리입니다. 운영 체제와 SQL Server가 정상적으로 작동하려면 충분한 메모리가 필요합니다. 그런 다음에도 데이터 캐시를 포함하여 시스템의 메모리 최적화되지 않은 모든 요구 사항을 충족할 수 있는 메모리가 필요합니다. 마지막으로, 무엇보다도 메모리 최적화 테이블을 위한 메모리를 추가하게 됩니다. 최소 64GB 메모리를 갖춘 상당히 큰 시스템을 보고 있지 않다면 이 시스템을 옵션으로 고려하는 것조차 제안하지 않습니다. 더 작은 시스템은 시간과 노력을 들일 만큼 충분한 메모리 저장 공간을 제공하지 못합니다.
    
    SQL Server 2014에서만 SQL Server Enterprise 버전이 실행되고 있어야 합니다. 물론 SQL Server 2014에서도 Developer 에디션을 사용할 수 있지만 프로덕션 로드를 실행할 수는 없습니다. SQL Server 2014보다 최신 버전의 경우 Microsoft에서 게시한 버전에 따라 메모리 제한이 있습니다.    

- ### b. 기본 설정
    하드웨어 요구 사항 외에도 인메모리 테이블을 활성화하려면 데이터베이스에 대한 추가 작업을 수행해야 합니다. 설명을 위해 새 데이터베이스부터 시작하겠습니다.

    ```sql    
    CREATE DATABASE InMemoryTest
    ON PRIMARY (NAME = N'InMemoryTest_Data',
                FILENAME = N'D:\Data\InMemoryTest_Data.mdf',
                SIZE = 5GB)
    LOG ON (NAME = N'InMemoryTest_Log',
        FILENAME = N'L:\Log\InMemoryTest_Log.ldf');
    GO
    ```

    인메모리 테이블이 내구성을 유지하려면 메모리는 물론 메모리에도 기록해야 합니다. 왜냐하면 메모리는 전원이 소모되기 때문입니다. 내구성(관계형 데이터세트의 ACID 속성의 일부)은 트랜잭션이 커밋되면 커밋된 상태를 유지한다는 의미입니다. 내구성이 있는 인메모리 테이블이나 내구성이 없는 테이블을 가질 수 있습니다. 비영구 테이블을 사용하면 커밋된 트랜잭션이 있을 수 있지만 여전히 해당 데이터가 손실될 수 있습니다. 이는 SQL Server 내에서 표준 테이블이 작동하는 방식과 다릅니다. 내구성이 없는 데이터의 가장 일반적으로 알려진 용도는 세션 상태나 전자 장바구니와 같은 시간에 민감한 정보 등입니다. 어쨌든, 인메모리 저장소는 표준 관계형 테이블 내의 일반적인 저장소와 동일하지 않습니다. 따라서 별도의 파일 그룹과 파일을 생성해야 합니다. 이렇게 하려면 다음과 같이 데이터베이스를 변경하면 됩니다.

    ```sql    
    ALTER DATABASE InMemoryTest
        ADD FILEGROUP InMemoryTest_InMemoryData
        CONTAINS MEMORY_OPTIMIZED_DATA
    GO        
    ALTER DATABASE InMemoryTest
        ADD FILE (NAME = 'InMemoryTest_InMemoryData',
        FILENAME = 'D:\Data\InMemoryTest_InMemoryData.ndf')
        TO FILEGROUP InMemoryTest_InMemoryData
    GO
    ```

    실험 중인 AdventureWorks2017 데이터베이스만 변경하면 되지만, 인메모리 최적화 테이블에 대한 또 다른 고려 사항은 특수 파일 그룹이 생성되면 제거할 수 없다는 것입니다. 데이터베이스만 삭제할 수 있습니다. 그렇기 때문에 별도의 데이터베이스로 실험해 보겠습니다. 더 안전합니다. 이는 또한 인메모리 기술을 구현하는 방법과 위치에 대해 주의를 기울이게 만드는 원동력 중 하나입니다. 영구적으로 변경하지 않고는 프로덕션 서버에서 사용해 볼 수 없습니다.

    인메모리 OLTP를 사용하는 데이터베이스에 사용할 수 있는 기능에는 몇 가지 제한 사항이 있습니다.

    * DBCC CHECKDB: 일관성 검사를 실행할 수 있지만 메모리 최적화 테이블은 건너뜁니다. DBCC CHECKTABLE을 실행하려고 하면 오류가 발생합니다. 
    * AUTO_CLOSE: 지원되지 않습니다. 
    * 데이터베이스 스냅샷: 지원되지 않습니다. 
    * ATTACH_REBUILD_LOG: 이 역시 지원되지 않습니다.
    * 데이터베이스 미러링: MEMORY_ OPTIMIZED_DATA 파일 그룹이 있는 데이터베이스를 미러링할 수 없습니다. 그러나 가용성 그룹은 원활한 환경을 제공하고 장애 조치 클러스터링은 인메모리 테이블을 지원합니다(그러나 복구 시간에 영향을 미칩니다).

    이러한 수정이 완료되어야 시스템에서 인메모리 테이블 생성을 시작할 수 있습니다.

- ### c. 테이블 생성
    데이터베이스 설정이 완료되면 앞서 설명한 대로 메모리 최적화 테이블을 생성할 수 있습니다. 실제 구문은 매우 간단합니다. AdventureWorks2017의 Person.Address 테이블을 최대한 복제하겠습니다.

    ```sql    
    USE InMemoryTest;
    GO
    CREATE TABLE dbo.Address
        (AddressID INT IDENTITY(1, 1) NOT NULL PRIMARY KEY NONCLUSTERED HASH
        WITH (BUCKET_COUNT = 50000),
        AddressLine1 NVARCHAR(60) NOT NULL,
        AddressLine2 NVARCHAR(60) NULL,
        City NVARCHAR(30) NOT NULL,
        StateProvinceID INT NOT NULL,
        PostalCode NVARCHAR(15) NOT NULL,
        --[SpatialLocation geography NULL,
        --rowguid uniqueidentifier ROWGUIDCOL NOT NULL CONSTRAINT DF_
        Address_rowguid DEFAULT (newid()),
        ModifiedDate DATETIME NOT NULL
        CONSTRAINT DF_Address_ModifiedDate
        DEFAULT (GETDATE()))
        WITH (MEMORY_OPTIMIZED = ON, DURABILITY = SCHEMA_AND_DATA);
    ```

    이렇게 하면 데이터의 내구성 있는 복사본을 유지하기 위해 정의한 디스크 공간을 사용하여 시스템 메모리에 내구성 있는 테이블이 생성되므로 정전 시 데이터가 손실되지 않습니다. 일반 SQL Server 테이블과 마찬가지로 IDENTITY 값인 기본 키가 있습니다. 그러나 SEQUENCE 대신 IDENTITY를 사용하려면 이 버전의 (1,1)을 제외한 다른 것으로 정의를 설정하는 기능을 포기해야 합니다. SQL 서버). 인덱스 정의는 클러스터링되지 않습니다. 대신 비클러스터형 해시입니다. 다음 섹션에서는 인덱싱 및 BUCKET_COUNT와 같은 항목에 대해 설명하겠습니다. 또한 SpatialLocation과 rowguid라는 두 개의 열을 주석 처리해야 했다는 점을 참고하세요. 이는 인메모리 테이블에서 사용할 수 없는 데이터 유형을 사용하고 있습니다. 마지막으로 WITH 문은 MEMORY_ OPTIMIZED=ON을 정의하여 SQL Server가 이 테이블을 배치할 위치를 알 수 있도록 합니다. DURABILITY=SCHEMA_ONLY를 사용하도록 WITH 절을 수정하면 더욱 빠른 테이블을 만들 수 있습니다. 이렇게 하면 데이터가 손실될 수 있지만 디스크에 아무 것도 기록되지 않으므로 테이블이 더욱 빨라집니다. 인메모리 테이블을 활용하지 못하게 할 수 있는 지원되지 않는 데이터 유형이 많이 있습니다.

    * XML
    * ROWVERSION
    * SQL_VARIANT
    * HIERARCHYID
    * DATETIMEOFFSET
    * GEOGRAPHY/GEOMETRY
    * User-defined data types

    데이터 유형 외에도 다른 제한 사항이 있습니다. "인메모리 인덱스" 섹션에서 인덱스 요구 사항에 대해 설명하겠습니다. SQL Server 2016부터 외래 키, 검사 제약 조건 및 고유 제약 조건에 대한 지원이 추가되었습니다. 인메모리 테이블이 생성되면 일반 테이블처럼 액세스할 수 있습니다. 지금 쿼리를 실행하면 어떤 행도 반환되지 않지만 작동할 것입니다.

    ```sql    
    SELECT a.AddressID
    FROM dbo.Address AS a
    WHERE a.AddressID = 42;
    ```

    따라서 데이터베이스의 실제 데이터를 실험하려면 AdventureWorks2017의 Person.Address에 저장된 정보를 이 새 데이터베이스의 메모리에 저장된 새 테이블에 로드하세요.

    ```sql    
    CREATE TABLE dbo.AddressStaging
    (
        AddressLine1 NVARCHAR(60) NOT NULL,
        AddressLine2 NVARCHAR(60) NULL,
        City NVARCHAR(30) NOT NULL,
        StateProvinceID INT NOT NULL,
        PostalCode NVARCHAR(15) NOT NULL
    );

    INSERT dbo.AddressStaging
    (
        AddressLine1,
        AddressLine2,
        City,
        StateProvinceID,
        PostalCode
    )

    SELECT a.AddressLine1,
        a.AddressLine2,
        a.City,
        a.StateProvinceID,
        a.PostalCode
    FROM AdventureWorks2017.Person.Address AS a;

    INSERT dbo.Address (AddressLine1, AddressLine2, City, StateProvinceID, PostalCode)

    SELECT a.AddressLine1,
        a.AddressLine2,
        a.City,
        a.StateProvinceID,
        a.PostalCode
    FROM dbo.AddressStaging AS a;

    DROP TABLE dbo.AddressStaging;
    ```

    그는 약 19,000개의 행을 준비 테이블에 넣은 다음 이를 인메모리 테이블에 로드합니다. 이는 성능 예시의 일부가 아니지만 표준 테이블에 데이터를 삽입하는 데 거의 850ms가 걸렸고 동일한 데이터를 내 시스템의 인메모리 테이블에 로드하는 데 2ms밖에 걸리지 않았다는 점은 아무 가치가 없습니다. 하지만 데이터가 있으면 그림 24-1과 같이 쿼리를 다시 실행하여 실제로 결과를 볼 수 있습니다.

    그림26-1 ![인메모리 테이블에 첫번째쿼리](image/26/26_01_FirstQueryInMemoryTable.png)  

    물론 이것은 별로 흥미롭지 않습니다. 따라서 의미 있는 작업을 수행하기 위해 몇 가지 다른 테이블을 만들어서 더 많은 쿼리 동작을 화면에서 볼 수 있도록 하겠습니다.

    ```sql    
    CREATE TABLE dbo.StateProvince 
    (
        StateProvinceID INT IDENTITY(1, 1) NOT NULL PRIMARY KEY NONCLUSTERED HASH
                                                                WITH (BUCKET_COUNT = 10000),
        StateProvinceCode   NCHAR(3) COLLATE Latin1_General_100_BIN2 NOT NULL,
        CountryRegionCode   NVARCHAR(3)     NOT NULL,
        Name                VARCHAR(50)     NOT NULL,
        TerritoryID         INT             NOT NULL,
        ModifiedDate        DATETIME NOT    NULL
        CONSTRAINT          DF_StateProvince_ModifiedDate DEFAULT (GETDATE())
    )
        WITH (MEMORY_OPTIMIZED = ON);

    CREATE TABLE dbo.CountryRegion
    (
        CountryRegionCode NVARCHAR(3) NOT NULL,
        Name VARCHAR(50) NOT NULL,
        ModifiedDate DATETIME NOT NULL
        CONSTRAINT DF_CountryRegion_ModifiedDate
        DEFAULT (GETDATE()),
        CONSTRAINT PK_CountryRegion_CountryRegionCode
        PRIMARY KEY CLUSTERED
        (
            CountryRegionCode ASC
        )
    );
    ```

    이는 추가 메모리 최적화 테이블과 표준 테이블입니다. 또한 여러분이 더 흥미로운 쿼리를 할 수 있도록 여기에 데이터를 로드하겠습니다.

    ```sql    
    SELECT sp.StateProvinceCode,
    sp.CountryRegionCode,
    sp.Name,
    sp.TerritoryID
    INTO dbo.StateProvinceStaging
    FROM AdventureWorks2017.Person.StateProvince AS sp;
    INSERT dbo.StateProvince (StateProvinceCode,
    CountryRegionCode,
    Name,
    TerritoryID)
    SELECT StateProvinceCode,
    CountryRegionCode,
    Name,
    TerritoryID
    FROM dbo.StateProvinceStaging;
    DROP TABLE dbo.StateProvinceStaging;
    INSERT dbo.CountryRegion (CountryRegionCode,
    Name)
    SELECT cr.CountryRegionCode,
    cr.Name
    FROM AdventureWorks2017.Person.CountryRegion AS cr;


    ```

    데이터가 로드되면 다음 쿼리는 단일 행을 반환하고 그림 24-2와 같은 실행 계획을 갖습니다.

    ```sql    
    SELECT a.AddressLine1,
        a.City,
        a.PostalCode,
        sp.Name AS StateProvinceName,
        cr.Name AS CountryName
    FROM dbo.Address AS a
        JOIN dbo.StateProvince AS sp
            ON sp.StateProvinceID = a.StateProvinceID
        JOIN dbo.CountryRegion AS cr
            ON cr.CountryRegionCode = sp.CountryRegionCode
    WHERE a.AddressID = 42;
    ```

    그림26-2 ![인메모리테이블과 일반테이블의 실행계획](image/26/26_02_ExecPlanBothInMemoryAndNormalTable.png)  
    
    보시다시피 인메모리 테이블을 사용하더라도 정상적인 실행 계획을 얻는 것이 전적으로 가능합니다. 운영자도 똑같습니다. 이 경우 세 가지 다른 인덱스 탐색 작업이 있습니다. 그 중 두 개는 인메모리 테이블을 사용하여 생성한 비클러스터형 해시 인덱스에 대한 것이고, 다른 하나는 표준 테이블에 대한 표준 클러스터형 인덱스 탐색입니다. 또한 이 계획의 예상 비용을 합하면 최대 101%가 된다는 점을 참고할 수도 있습니다. 최적화 프로그램을 통한 비용이 일반 테이블과 근본적으로 다르기 때문에 인메모리 테이블을 처리하는 이러한 이상 현상을 가끔 볼 수 있습니다.

    주요 성능 향상은 잠금 및 래칭이 없기 때문에 대량 삽입 및 업데이트가 가능하고 동시에 쿼리가 가능하다는 것입니다. 그러나 쿼리도 더 빠르게 실행됩니다. 이전 쿼리의 결과는 그림 24-3에 표시된 실행 시간 및 읽기 결과였습니다.

    그림26-3 ![인메모리 테이블의 쿼리 수치](image/26/26_03_QueryMetricForInMemoryTable.png)  

    AdventureWorks2017 데이터베이스에 대해 유사한 쿼리를 실행하면 그림 26-4에 표시된 동작이 발생합니다.

    그림26-4 ![일반테이블의 쿼리 수치](image/26/26_03_QueryMetricForInMemoryTable.png)  

    인메모리 테이블을 사용하면 실행 시간이 훨씬 더 좋다는 것은 분명하지만, 읽기가 처리되는 방식은 명확하지 않습니다. 그러나 인메모리 페이지나 디스크 페이지가 아니라 해시 인덱스를 사용하여 인메모리 저장소에서 읽는 것에 대해 이야기하고 있으므로 성능 측정 측면에서 상황이 완전히 다릅니다. 이전과 동일한 측정값을 모두 사용하지 않고 대신 실행 시간에 의존하게 됩니다. 이 경우 읽기는 시스템 활동을 측정한 것이므로 값이 높을수록 데이터에 대한 액세스가 더 많아지고 값이 낮을수록 더 적은 액세스를 의미할 것으로 예상할 수 있습니다. 테이블이 준비되고 삽입 및 선택 모두에 대한 성능이 향상되었다는 증거를 바탕으로 인메모리 테이블과 함께 사용할 수 있는 인덱스와 표준 인덱스와의 차이점에 대해 이야기해 보겠습니다.

- ### d. 인메모리 인덱스    