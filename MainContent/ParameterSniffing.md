---
sort: 18
---

# 파라메터 스니핑
이전장에서 실행 계획을 캐시로 가져오는 방법과 캐시에서 재사용하는 방법에 대해 논의했다. 이는 칭찬할 만한 목표이며 시스템의 전반적인 성능을 향상시키는 여러 가지 방법 중 하나이다. 실행계획 재사용을 보장하는 가장 좋은 메커니즘 중 하나는 저장 프로시저, 준비된 구문 또는 sp_executesql을 통해 쿼리를 매개 변수화하는 것이다. 이러한 모든 메커니즘은 계획을 생성할 때 하드 코딩된 값 대신 사용되는 매개변수를 생성합니다. 실행 계획을 생성할 때 포함된 값을 사용하기 위해 최적화 프로그램에서 이러한 매개변수를 샘플링하거나 스니핑할 수 있다. 대부분의 경우와 마찬가지로 이것이 잘 작동하면 보다 정확한 계획의 이점을 얻을 수 있지만 잘못된 매개변수 스니핑이 되면 심각한 성능 문제를 야기 할 수 있다.

이번 장에서는 다음과 같은 주제를 다룬다.

    * 파라메터 스니핑 뒤의 유용한 메카니즘
    * 어떻게 파라메터 스니핑이 문제를 발생시키는지
    * 나쁜 파라메터 스니핑을 발생시키는 메카니즘

## <font color='dodgerblue' size="6">1) 파라메터 스니핑</font>    
매개변수화된 쿼리가 옵티마이저로 전송되고 캐시에 기존 계획이 없으면 옵티마이저는 T-SQL 문에서 요청한 대로 데이터를 조작하기 위한 실행 계획을 생성한다. 이 매개변수화된 쿼리가 호출되면 매개변수 값은 프로그램을 통해 또는 매개변수 정의의 기본값을 통해 설정된다. 어느 쪽이든 거기에는 가치가 있습니다. 옵티마이저는 이것을 알고 있습니다. 그래서 그 사실을 이용하여 매개변수의 값을 읽습니다. 이것은 매개변수 스니핑으로 알려진 프로세스의 "스니핑" 측면입니다. 이러한 값을 사용할 수 있는 경우 옵티마이저는 해당 특정 값을 사용하여 매개변수가 참조하는 데이터의 통계를 확인합니다. 특정 값과 정확한 통계 세트를 사용하면 더 나은 실행 계획을 얻을 수 있습니다. 매개변수 스니핑의 이 유익한 프로세스는 기본값이 변경되지 않는다고 가정하고 매개변수가 있는 모든 쿼리에 대해 출처에 관계없이 항상 자동으로 실행됩니다.  
지역 변수를 스니핑할 수도 있습니다. 그러나 진행하기 전에 T-SQL 문 내에서 동일하게 보일 수 있으므로 지역 변수와 매개변수를 구분해 보겠습니다. 이 예에서는 지역 변수와 매개변수를 모두 보여줍니다.

```sql
CREATE PROCEDURE dbo.ProductDetails (@ProductID INT)
AS
DECLARE @CurrentDate DATETIME = GETDATE();
SELECT p.Name,
    p.Color,
    p.DaysToManufacture,
    pm.CatalogDescription
FROM Production.Product AS p
    JOIN Production.ProductModel AS pm
        ON pm.ProductModelID = p.ProductModelID
WHERE p.ProductID = @ProductID
    AND pm.ModifiedDate < @CurrentDate;
GO
```

이전 쿼리의 매개변수는 @ProductID입니다. 지역 변수는 @CurrentDate입니다. 매개변수는 저장 프로시저(또는 이 경우 준비된 명령문)로 정의됩니다. 지역 변수는 코드의 일부입니다. WHERE 절에 도달하면 완전히 동일하게 보이기 때문에 이들을 구별하는 것이 중요합니다.

지역 변수를 사용하는 명령문을 다시 컴파일하면 매개변수를 스니핑하는 것과 동일한 방식으로 최적화 프로그램에서 해당 변수를 스니핑할 수 있습니다. 이것만 알아두세요. 재컴파일과 관련된 이러한 고유한 상황을 제외하고 로컬 변수는 계획을 컴파일할 때 옵티마이저가 알 수 없는 양입니다. 일반적으로 매개변수만 스니핑할 수 있습니다.

매개변수 스니핑이 작동하는지 확인하고 유용하다는 것을 보여주기 위해 다른 절차부터 시작하겠습니다.
```sql
CREATE OR ALTER PROC dbo.AddressByCity @City NVARCHAR(30)
AS
SELECT a.AddressID,
    a.AddressLine1,
    AddressLine2,
    a.City,
    sp.Name AS StateProvinceName,
    a.PostalCode
FROM Person.Address AS a
    JOIN Person.StateProvince AS sp
        ON a.StateProvinceID = sp.StateProvinceID
WHERE a.City = @City;
GO
```
프로시저를 생성한 후 파라메터를 사용해 실행했다.
```sql
EXEC dbo.AddressByCity @City = N'London';
```
이것은 다음과 같은 결과를 초래할 것입니다.
그림 추가>

옵티마이저는 런던 값을 스니핑하고 주소 테이블의 통계 내에서 런던 시가 나타내는 데이터 분포를 기반으로 계획에 도달했습니다. 해당 쿼리 또는 테이블의 인덱스에 다른 조정 기회가 있을 수 있지만 계획은 런던 값 및 기존 데이터 구조에 최적입니다. 이와 같이 로컬 변수를 사용하여 동일한 쿼리를 작성할 수 있습니다.
```sql
DECLARE @City NVARCHAR(30) = N'London';

SELECT a.AddressID,
    a.AddressLine1,
    AddressLine2,
    a.City,
    sp.[Name] AS StateProvinceName,
    a.PostalCode
FROM Person.Address AS a
    JOIN Person.StateProvince AS sp
        ON a.StateProvinceID = sp.StateProvinceID
WHERE a.City = @City;
```
이 쿼리를 실행하면 I/O와 실행시간이 다르다.  
그림 추가>  
실행 시간이 증가했고 총 219개의 읽기에서 1084개로 이동했습니다. 이것은 그림에 표시된 새 실행 계획을 살펴봄으로써 다소 설명됩니다.
그림 추가> 
옵티마이저가 지역 변수 값을 샘플링하거나 스니핑할 수 없었기 때문에 통계에서 평균 행 수를 사용해야 했습니다. Index Scan 연산자의 속성에서 예상 행 수를 보면 알 수 있습니다. 34.113을 보여줍니다. 그러나 반환된 데이터를 보면 실제로 London 값에 대해 434개의 행이 있습니다. 간단히 말해서 옵티마이저는 434개의 행을 검색해야 한다고 생각하면 병합 조인을 사용하여 계획을 만들고 219개만 읽습니다. 그러나 약 34개 행만 반환한다고 생각하는 경우 중첩 루프 조인이 있는 계획을 사용합니다. 중첩 루프의 특성에 따라 상위 데이터 집합의 각 값에 대해 하위 값을 한 번씩 찾는 중첩 루프의 특성으로 인해 1,084개의 읽기 및 느린 성능.

즉 매개변수 스니핑이 실행되어 성능이 향상됩니다. 이제 매개변수 스니핑이 나빠지면 어떻게 되는지 봅시다.

## <font color='dodgerblue' size="6">2) 악성 파라메터 스니핑</font>  
매개변수 스니핑은 통계에 문제가 있을 때 문제를 만듭니다. 매개변수에 전달된 값은 통계 내의 데이터 및 데이터 분포를 나타낼 수 있습니다. 이 경우 좋은 실행 계획을 볼 수 있습니다. 그러나 전달된 매개변수가 테이블의 나머지 데이터를 나타내지 않으면 어떻게 될까요? 이 상황은 데이터가 평균이 아닌 방식으로 배포되기 때문에 발생할 수 있습니다. 예를 들어, 통계에 있는 대부분의 값은 몇 개의 행(예: 6)만 반환하지만 일부 값은 수백 개의 행을 반환합니다. 많은 양의 데이터가 공통적으로 분포되어 있고 흔하지 않은 작은 값 집합이 있는 것과 같은 방식으로 작동합니다. 이 경우 대표성이 없는 데이터를 기반으로 실행 계획이 생성되지만 대부분의 쿼리에는 유용하지 않습니다. 이 상황은 가장 자주 갑자기, 때로는 매우 심각한 성능 저하를 통해 스스로를 드러냅니다. 재컴파일 이벤트가 매개변수에 전달되는 더 나은 대표 데이터 값을 허용할 때 겉보기에 무작위로 보일 수도 있습니다.

통계가 최신이 아니거나, 스캔되는 대신 샘플링되기 때문에 정확하지 않거나(일반적인 통계에 대한 자세한 내용은 13장 참조), 완벽하게 구성되어 매우 들쭉날쭉한 경우( 데이터의 이상한 분포). 그럼에도 불구하고 상황은 유용하지 않은 계획을 만들어 캐시에 저장합니다. 예를 들어 다음 저장 프로시저를 사용합니다.
```sql
CREATE OR ALTER PROC dbo.AddressByCity @City NVARCHAR(30)
AS
SELECT a.AddressID,
    a.AddressLine1,
    AddressLine2,
    a.City,
    sp.Name AS StateProvinceName,
    a.PostalCode
FROM Person.Address AS a
    JOIN Person.StateProvince AS sp
        ON a.StateProvinceID = sp.StateProvinceID
WHERE a.City = @City;
GO
```
이전에 생성된 저장 프로시저인 dbo.AddressByCity가 이번에는 다른 매개변수로 다시 실행되면 다른 I 집합으로 반환됩니다.
```sql
EXEC dbo.AddressByCity @City = N'Mentor';
Reads: 218
Duration: 2.8ms
```
동일한 실행 계획이 재사용되기 때문에 I/O는 거의 동일합니다. 더 적은 수의 행이 반환되기 때문에 실행 시간이 더 빠릅니다. sys.dm_exec_query_stats의 출력을 보면 계획이 재사용되었는지 확인할 수 있습니다(그림 17-3 참조).
```sql
SELECT dest.text,
    deqs.execution_count,
    deqs.creation_time
FROM sys.dm_exec_query_stats AS deqs
    CROSS APPLY sys.dm_exec_sql_text(deqs.sql_handle) AS dest
WHERE dest.text LIKE 'CREATE PROC dbo.AddressByCity%';
```
<그림추가>
얼마나 나쁜 매개변수 스니핑이 발생할 수 있는지 보여주기 위해 프로시저 실행 순서를 반대로 할 수 있습니다. 먼저 DBCC FREEPROCCACHE를 실행하여 버퍼 캐시를 플러시합니다. 이 작업은 캐시에서 단일 실행 계획만 제거하는 여기에 표시된 작업을 주의 깊게 수행하지 않는 한 프로덕션 시스템에 대해 실행해서는 안 됩니다.
```sql
DECLARE @PlanHandle VARBINARY(64);

SELECT @PlanHandle = deps.plan_handle
FROM sys.dm_exec_procedure_stats AS deps
WHERE deps.object_id = OBJECT_ID('dbo.AddressByCity');

IF @PlanHandle IS NOT NULL
BEGIN
    DBCC FREEPROCCACHE(@PlanHandle);
END
GO
```
여기서 또 다른 옵션은 ALTER DATABASE SCOPED CONFIGURATION CLEAR PROCEDURE CACHE를 통해 지정된 데이터베이스에 대한 계획만 플러시하는 것입니다.
이제 역순으로 쿼리를 다시 실행합니다. 매개변수 값 Mentor를 사용하는 첫 번째 쿼리의 결과는 다음과 같습니다.
그림 추가>
그림 17-4는 그림 17-2와 같은 실행 계획이 아닙니다. 읽기 횟수는 약간 줄어들지만 실행 시간은 거의 동일하게 유지됩니다. 런던을 매개변수 값으로 사용하는 두 번째 실행은 다음과 같은 결과를 가져옵니다.
```
Reads:1084
Duration:97.7ms
```
이번에는 지역 변수를 사용했을 때와 달리 읽기가 훨씬 더 빨라지고 실행 시간이 늘어났습니다. 런던 매개변수를 사용하여 프로시저를 처음 실행할 때 생성된 계획은 데이터베이스에서 해당 기준과 일치하는 434개 행을 검색하는 데 가장 적합한 계획을 생성합니다. 그런 다음 매개변수 값 Mentor를 사용하는 프로시저의 다음 실행은 첫 번째 실행에서 생성된 동일한 계획을 사용하여 충분히 잘 수행되었습니다. 순서가 바뀌면 Mentor 값에 대해 새로운 실행 계획이 생성되었지만 London 값에는 전혀 효과가 없었습니다. 이 예에서 나는 실제로 약간의 속임수를 썼습니다. 해당 통계의 데이터 분포를 살펴보면 반환되는 평균 행 수가 약 34개인 반면 런던의 434개는 이상값임을 알 수 있습니다. 절차가 런던에 대해 컴파일되었을 때 보았던 약간 더 나은 성능은 다른 계획이 필요했다는 사실을 반영합니다. 그러나 Mentor와 같은 가치에 대한 성능은 런던에 대한 계획으로 약간 감소했습니다. 그러나 Mentor의 개선된 계획은 런던과 같은 가치에 절대적으로 비참했습니다. 이제 어려운 부분이 나옵니다. 시스템 부하에 맞는 계획을 결정해야 합니다. 한 계획은 평균 값에 대해 약간 더 나쁩니다. 반면 다른 계획은 평균 값에 더 낫지만 이상값에 심각한 피해를 줍니다. 문제는 가능한 모든 데이터 세트에 대해 다소 느린 성능을 갖고 이상값의 더 나은 성능을 지원하는 것이 더 낫습니까? 아니면 더 자주 호출될 수 있기 때문에 더 큰 데이터 단면을 지원하기 위해 이상값이 고통을 받도록 하는 것입니까? 이것은 자신의 시스템에서 파악해야 합니다.

- ### a. 악성 파라메터 스니핑 식별하기