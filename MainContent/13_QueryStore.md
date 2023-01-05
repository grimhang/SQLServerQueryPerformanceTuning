---
sort: 13
comments: true
---

# 쿼리 저장소

쿼리 저장소는 원래 2015년에 Azure SQL Database에 처음 소개되었으며 SQL Server에는 2016버전에 도입되었다.

쿼리 저장소는 세 가지 주요한 기능을 제공한다.  
    1. 쿼리의 성능수치와 실행 계획 등을 데이터베이스의 별도 저장공간에 영구적으로 저장하며 여러 유연한 방식으로 가져와 쉽게 성능 정보를 확인 할 수 있다.  
    2. 쿼리 저장소는 실행 계획 동작을 직접 제어할수 있게 하는데 이전에는 매우 어렵게 수행했어야 하지만 매우 쉽게 가능하도록 하였다.  
    3. 쿼리 저장소는 데이터베이스 업그레이드 시 시스템을 보호할 수 있는 위한 새로운 안전하며 보고기능이 있는 방식을 사용하여 쉽게 파악할 수 있도록 한다.  

이번 장에서는 다음과 같은 것을 설명할 예정이다.

    - 쿼리 저장소의 작동 방식 및 수집하는 정보 
    - 쿼리 저장소 동작에 대해 Management Studio를 통해 노출되는 보고서 및 메커니즘 
    - 실행계획 강제 적용과 같은 SQL Server 및 Azure SQL 데이터베이스에서 사용되는 실행 계획을 제어하는 방법
    - 시스템 활동을 보호하기 위한 업그레이드 방법

확장 이벤트 세션은 쿼리의 세부문제까지 조사하기 위한 좋은 수단이지만 제대로 운영하려면 많은 지식이 필요하다. 이에 비해 쿼리 저장소는 대략적인 쿼리의 성능정보를 가지고 있지만 대부분의 시스템에서는 이정도면 충분하기 때문에 우선적으로 사용하는 것이 좋을 것이다.

## <font color='dodgerblue' size="6">13.1 쿼리 저장소 함수와 디자인</font>

쿼리 저장소는 시스템에 미치는 영향 측면에서는 아마도 가장 가벼운 메커니즘일 것이며 시스템의 쿼리 성능을 제대로 이해하는 데 필요한 핵심 정보를 제공한다. 또한 쿼리 저장소에 대한 모든 작업은 시스템 뷰를 이용해 수행되기 때문에 T-SQL을 사용하여 우리가 직접 쿼리할수 있다. 쿼리 에 대한 성능수치를 모니터링하기 위한 메커니즘으로 DMO, 추적 이벤트 및 어느 정도 확장 이벤트를 사용하는 것은 쿼리 저장소의 도입으로 인해 구식으로 간주될 수 있다.

- ### a. 쿼리 저장소 활동
쿼리 저장소는 두 가지 정보를 수집하는데 첫째로는 시스템에서 각 쿼리 동작의 집계된 정보를 수집한다. 둘째, 쿼리 저장소는 기본적으로 쿼리당 최대 계획 수(기본적으로 200개)까지 시스템에서 생성된 모든 실행 계획을 캡처하며 데이터베이스별로 쿼리 저장소를 켜고 끌 수 있다. 켜져 있으면 쿼리 저장소가 그림 13-1과 같이 작동합니다.

    ![XE시작화면](image/13/13_01_QueryStoreCollectionData01.png)  

    그림 13-1 데이터를 수집하는 쿼리 저장소 활동

쿼리 최적화 프로세스는 일반적으로 수행된다. 쿼리가 일단 시스템에 제출되면 실행 계획이 생성되고(자세한 내용은 15장 참조) 계획 캐시에 저장된다.(16장 참조). 이러한 프로세스가 완료되면 쿼리 저장소는 계획 캐시에서 실행 계획을 캡처하는 비동기적 작업을 수행한다. 처음에는 이러한 계획을 임시 저장을 위해 별도의 메모리에 기록한다. 그러면 다른 비동기 프로세스가 이러한 실행 계획을 데이터베이스의 쿼리 저장소에 기록한다. 이 모든 것은 시스템 내의 다른 프로세스에 미치는 영향이 0은 아니지만 최소화되도록 하는 비동기식 프로세스이며 이 프로세스의 흐름에 대한 유일한 예외는 이 장의 뒷부분에서 다룰 계획 강요입니다.    

그런 다음 다른 쿼리와 마찬가지로 쿼리 실행이 발생합니다. 쿼리 실행이 완료되면 읽기 수, 쓰기 수, 쿼리 기간 및 대기 통계와 같은 쿼리 런타임 메트릭이 다시 비동기식으로 별도의 메모리 공간에 기록됩니다. 나중에 다른 비동기 프로세스가 해당 정보를 디스크에 기록합니다. 수집되어 디스크에 기록된 정보가 집계됩니다. 기본 집계 시간은 60분 간격입니다. 쿼리 저장소 시스템 테이블 내에 저장된 모든 정보는 쿼리 저장소가 활성화된 데이터베이스에 영구적으로 기록됩니다. 쿼리 메트릭과 쿼리에 대한 실행 계획은 데이터베이스에 보관됩니다. 데이터베이스와 함께 백업되고 데이터베이스와 함께 복원됩니다. 시스템이 오프라인 상태가 되거나 장애 조치되는 경우 아직 메모리에 있고 아직 디스크에 기록되지 않은 일부 쿼리 저장소 정보가 손실될 수 있습니다. 디스크에 쓰는 기본 간격은 15분입니다. 이것이 집계 데이터라는 점을 고려하면 프로덕션 수준 데이터로 간주되지 않아야 하는 일부 쿼리 저장소 데이터 손실 가능성에 대해 나쁜 간격이 아닙니다. 쿼리 저장소에서 정보를 쿼리하면 메모리 데이터와 디스크에 기록된 데이터가 모두 결합됩니다. 해당 정보에 액세스하기 위해 추가 작업을 수행할 필요가 없습니다. 이 장의 나머지 부분을 계속하기 전에 일부 코드 및 처리를 따라하려면 데이터베이스 중 하나에서 쿼리 저장소를 활성화해야 합니다. 이 명령은 다음을 수행합니다.

```sql
ALTER DATABASE AdventureWorks2017 SET QUERY_STORE = ON;
```

따라갈 때 쿼리 저장소에 쿼리가 있는지 확인하기 위해 이 저장 프로시저를 사용해 보겠습니다.

```sql
CREATE OR ALTER PROC dbo.ProductTransactionHistoryByReference (
    @ReferenceOrderID int
)
AS
BEGIN
    SELECT p.Name,
        p.ProductNumber,
        th.ReferenceOrderID
    FROM Production.Product AS p
        JOIN Production.TransactionHistory AS th
            ON th.ProductID = p.ProductID
    WHERE th.ReferenceOrderID = @ReferenceOrderID;
END
```

이 세 가지 파라메터 값으로 저장 프로시저를 실행하고 매번 캐시에서 제거하면 실제로 세 가지 다른 실행 계획을 갖게 됩니다.

```sql
DECLARE @Planhandle VARBINARY(64);

EXEC dbo.ProductTransactionHistoryByReference @ReferenceOrderID = 0;

SELECT @Planhandle = deps.plan_handle
FROM sys.dm_exec_procedure_stats AS deps
WHERE deps.object_id = OBJECT_ID('dbo.ProductTransactionHistoryByReference');

IF @Planhandle IS NOT NULL
BEGIN
    DBCC FREEPROCCACHE(@Planhandle);
END

EXEC dbo.ProductTransactionHistoryByReference @ReferenceOrderID = 53465;

SELECT @Planhandle = deps.plan_handle
FROM sys.dm_exec_procedure_stats AS deps
WHERE deps.object_id = OBJECT_ID('dbo.ProductTransactionHistoryByReference');

IF @Planhandle IS NOT NULL
BEGIN
    DBCC FREEPROCCACHE(@Planhandle);
END

EXEC dbo.ProductTransactionHistoryByReference @ReferenceOrderID = 3849;
```

이로써 쿼리 저정소 안에 있는 정보를 얻을 수 있다는 것을 확인했다.

- ### b. 쿼리 저장소가 수집하는 정보
    쿼리 저장소는 상당히 좁지만 매우 풍부한 데이터 집합을 수집합니다. 그림 11-2는 시스템 테이블과 그 관계를 나타냅니다.

    ![XE시작화면](image/13/13_02_QueryStoreSystemViews.png)  

    그림 13-2 쿼리 저장소의 시스템 뷰

    쿼리 저장소에 저장된 정보는 두 가지 기본 집합으로 나뉩니다. 쿼리 텍스트, 실행 계획 및 쿼리 컨텍스트 설정을 포함하여 쿼리 자체에 대한 정보가 있습니다. 그런 다음 런타임 간격, 대기 통계 및 쿼리 런타임 통계로 구성된 런타임 정보가 있습니다. 쿼리에 대한 정보부터 시작하여 정보의 각 섹션에 개별적으로 접근할 것입니다.

    1. 쿼리 정보
        쿼리 저장소의 핵심 데이터는 쿼리 자체입니다. 쿼리는 저장 프로시저 또는 일괄 처리의 일부일 수 있지만 독립적입니다. 기본 쿼리 텍스트와 주어진 쿼리를 식별할 수 있는 query_hash 값(쿼리 텍스트의 해시)으로 귀결됩니다. 그런 다음 이 데이터는 쿼리 계획 및 실제 쿼리 텍스트와 결합됩니다. 그림 11-3은 기본 구조와 일부 데이터를 보여줍니다.

        ![쿼리저장소의저장되는정보](image/13/13_03_QueryInformationInQS.png)  

        그림 13-3 쿼리저장소에 저장되는 정보

        쿼리 저장소가 활성화된 데이터베이스의 기본 파일 그룹에 저장된 시스템 테이블입니다. Management Studio 인터페이스에 내장된 우수한 보고서가 있지만 고유한 쿼리를 작성하여 쿼리 저장소의 정보에 액세스할 수 있습니다. 예를 들어 이 쿼리는 실행 계획과 함께 지정된 저장 프로시저에 대한 모든 쿼리 문을 검색할 수 있습니다.

        ```sql
        SELECT qsq.query_id,
            qsq.object_id,
            qsqt.query_sql_text,
            CAST(qsp.query_plan AS XML) AS QueryPlan
        FROM sys.query_store_query AS qsq
            JOIN sys.query_store_query_text AS qsqt
                ON qsq.query_text_id = qsqt.query_text_id
            JOIN sys.query_store_plan AS qsp
                ON qsp.query_id = qsq.query_id
        WHERE qsq.object_id = OBJECT_ID('dbo.ProductTransactionHistoryByReference');
        ```

        각각의 개별 쿼리 문이 쿼리 저장소 내에 저장되는 동안 object_id도 가져오므로 내가 정보를 검색하기 위해 수행한 것처럼 OBJECT_ID()와 같은 함수를 사용할 수 있습니다. 또한 query_plan 열에서 CAST 명령을 사용해야 했습니다. 이는 쿼리 저장소가 이 열을 XML이 아닌 텍스트로 올바르게 저장하기 때문입니다. SQL Server의 XML 데이터 형식에는 두 개의 열이 필요한 중첩 제한이 있습니다. 요구 사항을 충족하는 열은 XML이고 그렇지 않은 열은 NVARCHAR(MAX)입니다. 쿼리 저장소를 구축할 때 설계상 해당 문제를 해결했습니다. 실행 계획을 보기 위해 그림 11-4와 유사하게 결과를 클릭할 수 있으려면 이전에 했던 것처럼 CAST를 사용해야 합니다.

        ![쿼리저장소의저장되는정보](image/13/13_04_InfoInQS_UsingTSql.png)  

        그림 13-4 T-SQL을 이용해 쿼리 저장소의 정보 알아내기

        이 경우 단일 쿼리인 query_id = 75는 단일 문 저장 프로시저이며 세 가지 다른 plan_id 값으로 식별되는 세 가지 실행 계획이 있습니다. 이 계획은 잠시 후에 살펴보겠습니다. 쿼리 저장소의 결과에서 주목해야 할 또 다른 사항은 텍스트가 저장되는 방식입니다. 이 문은 매개 변수가 있는 저장 프로시저의 일부이므로 T-SQL 텍스트에 사용되는 매개 변수 값이 정의됩니다. 다음은 쿼리 저장소 내에서 명령문의 모습입니다(형식은 그대로 유지).

        ```sql
        (@ReferenceOrderID int)SELECT p.Name,       p.ProductNumber,
                    th.ReferenceOrderID FROM Production.Product
        AS p    JOIN Production.TransactionHistory AS
        th          ON th.ProductID = p.ProductID   WHERE
        th.ReferenceOrderID = @ReferenceOrderID
        ```

        문의 시작 부분에 있는 매개변수 정의에 유의하십시오. 앞에서 언급한 대로 실제 저장 프로시저 정의는 다음과 같습니다.

        ```sql
        CREATE OR ALTER PROC dbo.ProductTransactionHistoryByReference (
        @ReferenceOrderID int
        )
        AS
        BEGIN
            SELECT p.Name,
                    p.ProductNumber,
                    th.ReferenceOrderID
            FROM Production.Product AS p
            JOIN Production.TransactionHistory AS th
                ON th.ProductID = p.ProductID
            WHERE th.ReferenceOrderID = @ReferenceOrderID;
        END
        ```

        프로시저 내의 문과 쿼리 저장소에 저장된 문은 다릅니다. 이로 인해 쿼리 저장소 내에서 특정 쿼리를 찾으려고 시도할 때 몇 가지 문제가 발생할 수 있습니다. 다음과 같은 다른 예를 살펴보겠습니다.

        ```sql
        SELECT a.AddressID,
            a.AddressLine1
        FROM Person.Address AS a
        WHERE a.AddressID = 72;
        ```

        저장 프로시저가 아닌 일괄 처리입니다. 이것을 처음 실행하면 앞에서 설명한 프로세스를 사용하여 쿼리 저장소에 로드됩니다. 다음과 같이 이 명령문에 대한 정보를 검색하기 위해 일부 T-SQL을 실행하면 아무 것도 반환되지 않습니다.

        ```sql
        SELECT qsq.query_id,
        qsq.query_hash,
        qsqt.query_sql_text
        FROM sys.query_store_query AS qsq
        JOIN sys.query_store_query_text AS qsqt
        ON qsqt.query_text_id = qsq.query_text_id
        WHERE qsqt.query_sql_text = 'SELECT a.AddressID,
        a.AddressLine1
        FROM Person.Address AS a
        WHERE a.AddressID = 72;';
        ```

        이 명령문은 매우 단순했기 때문에 옵티마이저는 단순 매개변수화라는 프로세스를 수행할 수 있었습니다. 다행히 쿼리 저장소에는 자동 매개 변수화를 처리하는 기능인 sys.fn_stmt_sql_handle_from_sql_stmt가 있습니다. 이 기능을 사용하면 다음과 같이 쿼리에서 정보를 찾을 수 있습니다.

        ```sql
        SELECT qsq.query_id,
        qsq.query_hash,
        qsqt.query_sql_text,
        qsq.query_parameterization_type
        FROM sys.query_store_query_text AS qsqt
        JOIN sys.query_store_query AS qsq
        ON qsq.query_text_id = qsqt.query_text_id
        JOIN sys.fn_stmt_sql_handle_from_sql_stmt(
        'SELECT a.AddressID,
        a.AddressLine1
        FROM Person.Address AS a
        WHERE a.AddressID = 72;',
        2) AS fsshfss
        ON fsshfss.statement_sql_handle = qsqt.statement_sql_handle;
        ```

        이것이 작동하려면 형식과 공백이 모두 동일해야 합니다. 하드 코딩된 값은 변경될 수 있지만 나머지는 모두 동일해야 합니다. 쿼리를 실행하면 그림 11-5와 같이 표시됩니다.

        

        ![그림](image/13/13_05_ResultSimpleParam.png)  

        그림 13-5 간단한 파라메터화 결과

        단순 매개변수화를 위한 매개변수 값이 저장 프로시저와 마찬가지로 텍스트에 추가된 query_sql_text 열에서 볼 수 있습니다. 나쁜 소식은 sy.fn_stmt_sql_handl_from_sql_stmt가 현재 자동 매개변수화에서만 작동한다는 것입니다. 다른 소스에서 매개변수화된 명령문을 찾는 데 도움이 되지 않습니다. 해당 정보를 검색하려면 LIKE 명령을 사용하여 텍스트를 검색하거나 이전에 했던 것처럼 저장 프로시저에서 쿼리에 object_id를 사용해야 합니다.

    2. 쿼리 런타임 데이터
        쿼리 및 계획에 대한 정보를 검색한 후 다음으로 원하는 것은 런타임 메트릭을 보는 것입니다. 런타임 메트릭을 이해하는 데는 두 가지 핵심이 있습니다. 첫째, 메트릭은 쿼리가 아닌 계획에 다시 연결됩니다. 각 계획은 서로 다른 조인 유형 및 나머지 모든 인덱스에 대해 서로 다른 작업으로 다르게 작동할 수 있으므로 런타임 데이터 및 대기 통계를 캡처한다는 것은 다시 계획에 연결된다는 것을 의미합니다. 둘째, 런타임 및 대기 통계가 집계되지만 런타임 간격으로 집계됩니다. 런타임 간격의 기본값은 60분입니다. 즉, 각 런타임 간격에 대한 각 계획에 대해 서로 다른 메트릭 집합을 갖게 됩니다.

## <font color='dodgerblue' size="6">13.2 쿼리 저장소 리포팅</font>

일부 작업에서는 T-SQL을 사용하여 쿼리 저장소를 직접 제어하고 시스템 테이블을 사용하여 쿼리 저장소에 대한 데이터를 검색하는 것이 선호되는 접근 방식이 될 것입니다. 그러나 정상적인 작업 비율을 위해 쿼리 저장소로 작업할 때 기본 제공 보고서와 해당 동작을 활용할 수 있습니다. 이러한 보고서를 보려면 Management Studio의 개체 탐색기 창에서 데이터베이스를 확장하기만 하면 됩니다. 쿼리 저장소가 활성화된 모든 데이터베이스의 경우 그림 11-10과 같이 보고서가 표시되는 새 폴더가 있습니다.

![그림](image/13/13_10_QSReportAdventureWorks2017.png)  

그림 13-10 AdventureWorks2017 데이터베이스의 쿼리 저장소 보고서

이 리포트는 다음 정보를 포함한다.

    - 회귀된 쿼리: 시간이 지남에 따라 성능이 부정적인 방식으로 변경된 쿼리를 볼 수 있습니다.
    - 전체 리소스 소비량: 이 보고서는 정의된 기간 동안 다양한 쿼리의 리소스 소비를 보여줍니다. 기본값은 지난 달입니다.
    - 리소스를 가장 많이 사용하는 쿼리: 여기에서 기간에 관계없이 가장 많은 리소스를 사용하는 쿼리를 찾을 수 있습니다.
    - 강제 계획이 포함된 쿼리: 강제 계획이 있는 것으로 정의한 모든 쿼리가 이 보고서에 표시됩니다.
    - 고변형 쿼리: 이 보고서는 런타임 통계에서 변동이 심한 쿼리를 표시하며 실행 계획이 둘 이상인 경우가 많습니다.
    - 쿼리 대기 통계 :
    - 추적된 쿼리: 쿼리 저장소를 사용하면 관심 있는 쿼리를 정의할 수 있으며 다른 보고서에서 추적을 시도하는 대신 쿼리를 표시하고 여기에서 찾을 수 있습니다.
    
    이러한 각 보고서는 고유하며 각기 다른 목적에 유용하지만 모든 보고서를 자세히 살펴볼 시간과 공간이 없습니다. 대신 리소스를 많이 사용하는 쿼리 중 하나의 동작에 초점을 맞추겠습니다. 이는 일반적으로 다른 모든 쿼리의 동작을 나타내고 상당히 자주 사용할 가능성이 높기 때문입니다. 보고서를 열면 그림 13-11과 유사한 내용이 표시됩니다.

![그림](image/13/13_11_Top25ResConsumerReport.png)  

그림 13-11 쿼리 저장소의 Top 25 리소스 소비자

보고서에는 세 개의 창이 있습니다. 왼쪽 상단의 첫 번째는 query_id 값으로 집계된 쿼리를 보여줍니다. 오른쪽의 두 번째 창에는 시간 경과에 따른 다양한 쿼리 동작과 해당 쿼리에 대한 다양한 계획이 표시됩니다. 첫 번째 창에서 강조 표시된 첫 번째 쿼리에 세 가지 다른 실행 계획이 있음을 알 수 있습니다. 이러한 계획 중 하나를 클릭하면 화면 하단의 세 번째 창에서 해당 계획이 열립니다. 기본 동작으로 제한되지 않습니다. 기본적으로 기간별로 집계된 쿼리를 표시하는 첫 번째 창은 다른 두 개를 구동합니다. 그림 13-12와 같이 화면 상단에 현재 13개의 선택 항목을 제공하는 드롭다운이 있습니다.

![그림](image/13/13_12_Top25ResConsumerReport.png)  

그림 13-12 Top 25 리소스 소비자 리포트의 다른 집계 기준

그 중 하나를 선택하면 보고서에 대해 집계되는 값이 변경됩니다. 다른 드롭다운을 사용하여 보고서가 집계되는 방식을 변경할 수도 있습니다. 이 목록에는 평균, 최소값, 최대값, 총계 및 표준 편차가 포함됩니다. 첫 번째 창의 추가 기능에는 그리드 형식으로 변경하고, 나중에 추적할 쿼리를 표시하고(추적된 쿼리 보고서에서) 보고서를 새로 고치고 쿼리 텍스트를 보는 기능이 포함됩니다. 이 모든 것은 성능 문제를 확인하기 위해 작업할 때 시간을 할애할 쿼리를 식별하는 데 유용합니다.

다음 창에는 첫 번째 창에서 선택한 항목의 성능 메트릭이 표시됩니다. 각 점은 특정 시점과 특정 실행 계획을 모두 나타냅니다. 그림 11-13의 정보는 오전 8시 45분부터 오전 9시 45분까지 쿼리 성능이 어떻게 변했는지, 그리고 해당 시간 프레임 동안 쿼리의 성능 및 실행 계획이 어떻게 변경되었는지 보여줍니다.

![그림](image/13/13_13_DiffPerfDiffPlanOneSql.png)  

그림 13-13 하나의 쿼리가 다른 성능과 다른 실행계획을 가지는 경우

각 점의 크기는 주어진 시간 프레임 내에서 주어진 계획의 실행 횟수에 해당합니다. 주어진 점 위로 마우스를 가져가면 해당 시점에 대한 추가 정보가 표시됩니다. 그림 11-14는 오전 9시 45분 시간대에 화면 상단에 있는 점 plan_id = 76에 대한 정보를 보여줍니다.

![그림](image/13/13_14_DetailInfoOnePlan.png)  

그림 13-14 주어진 플랜에 대한 상세한 정보

해당 쿼리에 대한 특정 계획에 대한 실행 수 및 기타 메트릭을 볼 수 있습니다. 어떤 점을 클릭하든 최종 창에서 해당 점에 대한 실행 계획을 볼 수 있습니다. 표시된 실행 계획은 Management Studio 내의 다른 그래픽 계획과 마찬가지로 작동하므로 여기에서는 동작을 자세히 설명하지 않겠습니다. 여기에 표시되는 추가 기능 중 하나는 계획을 강제 실행하는 기능입니다. 그림 11-15와 같이 실행 계획 창의 오른쪽 상단에 두 개의 버튼이 표시됩니다.

![그림](image/13/13_15_ForceUnforcePlan.png)  

그림 13-15 리포트로부터 계획 강제 적용 및 해제

보고서에서 직접 계획을 강제 적용하거나 강제 해제할 수 있습니다. 계획 강요에 대해서는 다음 섹션에서 자세히 다루겠습니다.