---
sort: 6
---

# SQL Server 전체적 성능
대략적인 SQL Server 전체 상황을 모니터링 할수 있는 카운터들

```
Object(Instance)             Counter
---------------------------  --------------------------
SQLServer:Access Methods     FreeSpace Scans/sec
                             Full Scans/sec
                             Table Lock Escalations/sec
                             Worktables Created/sec
SQLServer:Latches            Total Latch Wait Time (ms)
SQLServer:Locks(_Total)      Lock Timeouts/sec
                             Lock Wait Time (ms)
                             Number of Deadlocks/sec
SQLServer:SQL Statistics     Batch Requests/sec
                             SQL Re-Compilations/sec
SQLServer:General Statistics Processes Blocked
                             User Connections
                             Temp Tables Creation Rate
                             Temp Tables for Destruction                            
```

## 6.1 인덱스 누락
```
Object(Instance)             Counter       
---------------------------  --------------
SQLServer:Access Methods     Full Scans/sec
```

### * Full Scans/sec
테이블이나 인덱스에서 발생하는 풀 테이블 스캔 수. 전체 테이블스캔(풀 테이블스캔)은 데이터량이 적을 경우에는 유리한 경우가 있어서 항상 나쁜것은 아니다. 하지만 대부분의 풀스캔은 문제의 원인이 되며 주요 원인은 다음과 같다.

  * 누락된 인덱스
  * 과도한 데이터 로우 수 요청. where절에서 적절하게 제한하지 않은 경우 등등
  * 선택도가 높지 않은 예측
  * 적절하지 않은 T-SQL
  * 데이터 분포 또는 수량이 seek 연산을 지원하지 않거나 초과하는 경우.

문제 쿼리를 알아내기 위해 확장 이벤트 사용 하는것이 좋다. 인덱스 누락되거나 요청 된 행이 너무 많거나 또는 T-SQL 형식이 잘못된 쿼리는 테이블 이나 인덱스의 풀 스캔을 유발한다. 이렇게 되면 과도한 CPU 타임과 논리적 읽기가 발생하기 때문에 문제가 된다.  

임시 테이블에 인덱스가 없어서 대부분의 시간동안 풀스캔을 하는 저장 프로 시저에서도 이 카운터가 도움이 된다.


### * DMO(Dynamic Management Objects)
동적 관리 뷰 sys.dm_db_missing_index_details를 이용해 누락된 인덱스를  확인할 수도 있다.
이 DMV는 실행중인 쿼리의 실행 계획을 바탕으로 인덱스 후보를 제안해 준다. 보통 "누락 인덱스 기능"이라고 부른다.

이 DMV는 캐시에 저장된 실행 계획에서 생성 된 데이터를 기반으로 하며 이 뷰를 사용해 인덱스를 작성할지 말지 결정할 수 있다. 누락 인덱스는 해당 쿼리에 대한 XML 실행 계획에도 표시 되지만 다음 장에서 더 자세히 다룰 것이다. 이러한 뷰는 신규 사용 가능한 인덱스를 제안하는 데 유용하지만 어떤 쿼리가 이 신규 인덱스를 사용하는 것이 맞는지에 대한 쿼리정보를 저장하지 않기 때문에 쿼리와 연결할 수 없다. 누락 인덱스를 특정 쿼리와 연결하려면 다음 장에서 설명하는 기술을 사용하는 것이 좋다. 다시 한번 말하지만 누락 모든 인덱스에 대한 제안이 정말로 효과적인지 판단하기 위해서 신중한 사전 테스트가 꼭 필요하다.

누락 된 인덱스에 대한 문제를 해결하지 전에 미 사용 인덱스를 먼저 해결하면 매우 좋다. sys.dm_db_index_usage_stats DMV를 사용해 이런 데이터를 조사할수 있는데 이 DVM 정보는 SQL Server 인스턴스를 마지막으로 다시 시작한 이후의 인덱스 사용 통계정보를 보여준다. 안타깝게도 이 DMV 내의 카운터를 재설정하거나 제거하는 방법은 여러 가지가 있으므로 인덱스 사용에 대한 100% 정확한 정보라고 할순 없다. 하지만 하위 수준 DMV 인 sys.dm_db_index_operational_stats(파티션에 대한 현재 하위수준 정보)를 볼 수 있다.
이로 인해 경합 또는 I/O로 인해 인덱스가 느려지는 위치를 표시하는 데 도움이됩니다. 이 두 가지에 대해서는 20 장에서 자세히 다룰 것이고 또한 Database Tuning Advisor (10 장에서 다룬)의 제안이 특정 쿼리에 대한 특정 인덱스를 사용하는 데 도움이 될 수 있습니다.


## 6.2 데이터베이스 동시성
블로킹 같은 동시성 문제를 알아내기 위하여 다음 카운터를 조사할 필요가 있다.
```
Object(Instance)             Counter
---------------------------  --------------------------
SQLServer:Latches            Total Latch Wait Time (ms)
SQLServer:Locks(_Total)      Lock Timeouts/sec
                             Lock Wait Time (ms)
                             Number of Deadlocks/sec
```
### * Total Latch Wait Time (ms)
래치는 테이블 행과 같은 데이터 구조의 무결성을 보호하기 위해 SQL Server에서 내부적으로 사용되며 사용자가 직접 제어할 수 없다. 이 카운터는 마지막 1 초 동안 기다려야했던 래치 요청에 대한 총 래치 대기 시간 (밀리 초)을 모니터링합니다. 이 카운터 값이 높으면 SQL Server가 내부 동기화 메커니즘을 기다리는데 너무 많은 시간을 소비하고 있음을 나타낸다.


### * Lock Timeouts/Sec and Lock Wait Time (Ms)
보통은 Lock Timeouts/sec는 0이고 Lock Wait Time (ms)은 매우 낮다. Lock Timeouts/sec 값이 0이 아니고 Lock Wait Time (ms) 값이 높으면 데이터베이스에서 과도한 차단이 발생하고 있음을 나타낸다. 이 경우 두 가지 접근 방식을 채택 할 수 있다.

  - SQL 프로필러의 데이터 또는 xevent 또는 sys.dm_exec_query_stats를 사용하여 현재 캐시에 있는 고 비용 쿼리를 식별 한 다음 적절하게 튜닝해야 함.
  - 차단 분석을 사용하여 과도한 차단의 원인을 진단 할 수 있다. 비용이 많이 드는 쿼리를 먼저 최적화하는 데 집중하는 것이 일반적으로 유리하다. 그러면 다른 사용자에 대한 차단이 줄어들 기 때문이며 뒤에서 차단을 분석하고 해결하는 방법을 알려준다.
  - 확장 이벤트는 차단 정보를 캡처하기 위해 임계 값을 활성화하고 설정할 수있는 blocked_process_report라는 차단 이벤트를 제공합니다.

어느 정도의 잠금은 시스템의 필수 부분임을 기억하십시오. 주어진 값이 우려의 원인인지 철저하게 추적하기 위해 기준을 설정하는 것이 좋습니다.


### * Number of Deadlocks/Sec
보통은 이 카운터에는 0이 표시되며 0이 아닌 값을 발생할 경우 희생된 쿼리를 재 실행하거나 교착 상태를 해결하고 해결하기 위해 무언가를 시도해야한다는 것이다. 21 장에서는이를 수행하는 방법을 보여준다.





## 6.3 재사용하지 않는 실행계획
저장 프로시저 쿼리에 대한 실행 계획을 생성하려면 CPU 파워가 필요하므로 실행 계획을 재사용하여 CPU에 대한 스트레스를 줄일 수 있다. 재 컴파일중인 저장 프로 시저의 수를 분석하려면 표 4-6의 카운터를 참조. 


```
Object(Instance)             Counter
---------------------------  -----------------------
SQLServer:SQL Statistics     SQL Re-Compilations/sec
```
저장 프로 시저를 다시 컴파일하면 프로세서에 오버 헤드가 추가된다. SOL Re-Compilations/sec 카운터에 대해 가능한 한 0에 가까운 값을보고 싶지만 결코 볼 수는 없다. 기준 측정 값에서 벗어나거나 급증하는 값이 일관되게 표시되는 경우 확장 이벤트를 사용하여 재 컴파일중인 저장 프로 시저를 추가로 조사해야합니다. 관련 저장 프로 시저를 식별 한 후에는
재 컴파일의 원인을 분석하고 해결합니다. 17 장에서는 재 컴파일의 다양한 원인을 분석하고 해결하는 방법을 배웁니다.

## 6.4 일반적인 활동
```
Object(Instance)             Counter
---------------------------  ----------------------
SQLServer:SQL Statistics     Batch Requests/sec
SQLServer:General Statistics User Connections
```

### * User Connections
여러 읽기 전용 SQL Server는로드 균형 조정 환경 (SQL Server가 여러 컴퓨터에 분산되어 있음)에서 함께 작동하여 많은 데이터베이스 요청을 지원할 수 있습니다. 이러한 경우 사용자 연결 카운터를 모니터링하여 여러 SQL Server 인스턴스에서 사용자 연결의 분포를 평가하는 것이 좋습니다. 사용자 연결은 정상적인 애플리케이션 동작으로 스펙트럼 전체에 걸쳐있을 수 있습니다. 이것은 기준선이 필수적인 곳입니다
예상되는 동작을 결정합니다. 이 기준을 설정하는 방법을 곧 알게 될 것입니다.

### * Batch Requests/sec
이 카운터는 SQL Server의 부하를 나타내는 좋은 지표입니다. 시스템 리소스 사용 수준 및 초당 일괄 요청 수를 기반으로 SQL Server가 리소스 병목 현상을 일으키지 않고 사용할 수있는 사용자 수를 추정 할 수 있습니다. 서로 다른로드주기에서이 카운터 값은 데이터베이스 연결 수와의 관계를 이해하는 데 도움이됩니다. 이는 또한 SQL Server와 초당 웹 요청의 관계, 즉 활성
Server Pages. Microsoft IIS (인터넷 정보 서비스) 및 ASP (Active Server Pages)를 사용하는 웹 응용 프로그램에 대한 요청 / 초. 이 모든 분석은 사용자 부하가 변경 될 때 시스템 동작을 더 잘 이해하고 예측하는 데 도움이됩니다.
이 카운터의 값은 일반적인 응용 프로그램 동작으로 광범위한 스펙트럼에 걸쳐있을 수 있습니다. 예상되는 행동을 결정하려면 정상적인 기준이 필수적입니다.
      