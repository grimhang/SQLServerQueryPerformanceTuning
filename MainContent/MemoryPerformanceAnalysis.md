---
sort: 2
---

# Memory Performance Analysis
쿼리는 처리되기 전에 메모리에 데이터를 올려놓는다. 데이터를 수정하는 어떤 변경점도 해당 데이터가 맨 처음 메모리에 로드된다.  
그래서 많은 다른 연산 작업들은 메모리를 주로 이용하기 때문에 속도 상 유리한 점이 있다.  
이 때문에 메모리에 대한 이해가 필요하다.

이번장에서 다룰 주제

* 성능 모니터의 기본
* 몇몇 시스템 활동상황을 관찰하기 위해 사용되어지는 동적 관리 오브젝트
* 어떻게, 왜 하드웨어 리소스는 병목을 발생시키는지
* SQL Server 와 윈도우 OS에서 사용되어지는 메모리를 관찰하고 측정하는 방법
* 메모리 병목의 가능한 해결책

***
## 성능 모니터
CPU, Memory, Disk, Network의 자세한 활동 데이터 측정 가능.  
또는 SQL Server 2014에서는 추가적인 기능을 성능 모니터를 통해 사용 가능.  
단 VM에서 측정하는 값은 논리적인 VM 각각의 데이터이기 때문에 물리적 서버의 데이터가 아니다. 정확한 데이터가 아니기 때문에 주의.

시스템 실시간 활동데이터를 그래프로 바로 볼 수도 있고, data collector set 이라는 파일로 저장할 수도 있다.  
실서버에서는 파일로 저장하는게 오버헤드가 더 적기 때문에 보다 선호. 

명령도구(cmd.exe)에서 perfmon 이라고 치면 성능 모니터가 실행됨.

* 동적 관리 오브젝트
내부적으로 동적관리오브젝트(DMO)를 사용하는 동적관리뷰(DMV)와 동적 관리함수(DMF)라는 형태로 SQL Server에서도 성능모니터에서와 같이 실시간 스냅샷 데이터를 제공한다.

sys.dm_os_performance_counters dmv는 쿼리로 SQL Server의 카운터를 쿼리로 볼수 있게 해준다. 아래는 Login/sec 예제
```sql
SELECT cntr_value
    , cntr_type
FROM sys.dm_os_performance_counters
WHERE object_name = 'SQLServer:General Statistics'
    AND counter_name = 'Logins/sec';
```        
이때 cntr_value 는 현재까지 누적치이고 cntr_type은 각각의 카운터를 가르키는 정수값이다. 여기서 참조가능
[여기](https://docs.microsoft.com/ko-kr/windows/win32/wmisdk/wmi-performance-counter-types?redirectedfrom=MSDN) 서 참조 가능

sys.dm_os_wait_stats 는 다양한 대기 상태의 누적치. 대기상태를 아는 것은 병목의 원인을 알수있는 가장 쉬운 방법이다.
```sql
SELECT TOP (10) dows.*
FROM sys.dm_os_wait_stats AS dows
ORDER BY dows.wait_time_ms DESC;
```
다양한 대기 상태를 알수 있다.  
Microsoft에서 [대기상태](http://bit.ly/1e1I38f) 찾기.


## 성능 모니터
전통적으로 다음 4개의 하드웨어 리소스에 영향을 받는다.
    * 메모리
    * Disk I/O
    * CPU
    * Network

* 병목 알아내기
리소스 병목간에는 서로 관계를 맺고 있다. 예를 들면 CPU병목은 과도한 페이징(메모리 병목)이나 디스크 속도 저하(디스크병목)같은 증상이 발생한다.
또한 시스템에 메모리가 부족하면 과도한 페이징을 유발하고 느린 디스크가 된다. 이때 CPU를 더 빠른 것으로 교체하는것은 약간 좋은 해결책이 될ㅅ ㅜ있지만 최적의 솔루션이 아니다. 이 경우 메모리 증설이 디스크/CPU의 압박을 줄여주기 때문에 좀더 적절한 해결방법이다.
    
    병목 식별 방법
    - 가장좋은 방법 : 처리를 완료하기 위해 한 리소스가 다른 리소스를 기다리는지 알아내는 법
    - 두번째 방법 : 응답시간과 용량을 조사해 알아내기
         예를 들면 벤더가 제시한 대역폭과 용량을 알ㅁ기 disk sec/transfer.  그 이상을 넘으면 과도한 로드라고 할수 있다. 

    모든 리소스가 쿼리레벨에서 조회 가능한 특별한 카운터를 가지고 있는건 아니지만 리소스의 과도한 사용을 표시하는 카운터는 대부분 존재.

    예) 메모리는 그러한 카운터는 없지만 큰 숫자의 하드 페이지 폴트는 물리적 메모리의 한계가 부족하다는 것을 의미. (pages/sec, page faults/sec) 

    CPU나 디스크과 같은 다른 리소스들도 대부분 queuing 수치 카운터가 존재한다.
    예) SQL Server의 Page Life Expectancy는 디스크의 데이터를 메모리의 버퍼캐시로 올려놓고 유지하는 시간(단위:초)을 말하는데 이 값이 작다면 SQL Server의 메모리를 적절하게 이용하지 못한다는 얘기이고 메모리 병목이라고 판단할수 있는 것이다.

* 병목 해결 방법
일단 병목을 발견하면 다음 두가지 방법중 하나를 선택할 수 있다.
    - 하드웨어 리소스를 증설
    - 리소스를 사용하는 방법을 교정(쿼리 튜닝과 같은)



## 메모리 병목 분석
메모리 병목 현상은 시스템의 다른 리소스에도 문제를 발생.  
예) SQL Server가 버퍼 캐시가 부족하게 되면
    - SQL Server의 프로세스(lazy writer같은)는 충분한 여유 내부 메모리페이지를 유지하기 위하여 과도하게 작동한다.
    - 이는 과도한 CPU 사용률
    - 메모리 페이지를 디스크에 쓰려하는 추가적인 물리적 disk I/O를 발생

* SQL Server 메모리 관리
SQL Server의 메모리 구성
    - 데이터베이스용 메모리
    - 데이터용 메모리 요구사항
    - 쿼리 실행계획
    - 버퍼풀이라고 불리는 대량의 메모리 풀

메모리 풀은 8KB 버퍼들의 컬렉션.
데이터 페이지, 플랜 캐시 페이지, 프리 페이지와 같은 다양한 페이지 존재
SQL server는 동적으로 메모리 풀 크기를 늘리거나 줄임.

SSMS에서 세팅방법 서버등록정보/메모리 에서
![CI](image/MemoryConfig.png)

동적 메모리 범위는 두개의 구성 정보. Minimum server Memory(MB), Maximum server memory(MB)
    - Minimum(MB) : "min server memory"은 메모리 풀의 가장 낮은 값. 일단 메모리 풀이 최소값과 같은 크기에 도달하면 SQL Server는 메모리 풀의 페이지를 계속 커밋 할 수 있지만 최소값보다 작게는 축소 할수 없다. SQL Server는 min server memory 값으로 시작하지 않고 필요에 따라 덩적으로 메모리 커밋. 

    - Maximum(MB) : "max sserver memory" 메모리 풀의 최대 증가를 제한하는 상한 값 역할. 이러한 구성 설정은 즉시 적용되며 다시 시작할 필요가 없습니다. 

Microsoft는 동적 메모리 권장을 사용ㅏ도록 권장. min server memory는 0. Max server memory는 OS에 약간의 memory 허용치를 놔두게.
8~16GB 메모리 일 경우 OS메모리는 2~4GB 놔두게. 일반적으로는 os메모리가 16GB 늘때마다 4GB는 OS에 여유로 남긴다.

최소 서버 메모리가 0 인 SQL Server에 동적 메모리 구성을 사용하는 것이 좋습니다.
최대 서버 메모리는 시스템의 단일 인스턴스를 가정하여 운영 체제에 일부 메모리를 허용하도록 설정됩니다.
운영 체제의 메모리 양은 시스템 자체에 따라 다릅니다. 8GB –16GB의 대부분의 시스템
메모리의 경우 약 2GB-4GB를 OS에 남겨 두어야합니다. 서버의 메모리 양이 증가함에 따라
OS에 더 많은 메모리를 할당합니다. 일반적인 권장 사항은 전체 시스템의 32GB를 초과하는 16GB마다 4GB입니다.
기억. 시스템의 요구 사항과 메모리 할당에 따라이를 조정해야합니다. 당신은 실행해서는 안됩니다
SQL Server와 동일한 서버에있는 다른 메모리 집약적 응용 프로그램, 그러나 필요한 경우 먼저 예상치를 얻는 것이 좋습니다.
다른 응용 프로그램에 필요한 메모리 양을 확인한 다음 최대 서버 메모리 값을 설정하여 SQL Server를 구성하여 다른 응용 프로그램이 SQL Server의 메모리를 고갈시키지 않도록합니다. SQL Server가 실행중인 시스템
자신의 경우 최소 서버 메모리를 최대 값과 동일하게 설정하고 동적 관리로 간단히 발송하는 것을 선호합니다.
여러 SQL Server 인스턴스가있는 서버에서는 각 인스턴스에 다음과 같은 메모리 설정이 있는지 확인해야합니다.
적절한 가치. 운영 체제 및 외부 프로세스를위한 충분한 메모리가 남아 있는지 확인하십시오.

SQL Server 의 메모리는 크게 데이터페이지와 프리페이지가 있는 버퍼풀 메모리와 쓰레드, DLL들, 연결된 서버들 등등이 있는 비버퍼 메모리로 나눠진다.
대부분은 버퍼풀이 차지한다. 그러나 버퍼풀 그 너머 영역(private bytes라고 알려진)까지 얻을 수 있긴 하지만 일반적으로 버퍼풀 모니터링하는 정상적인 절차에 걸리지 않기 때문에 메모리 압박을 유발 할수도 있다.
이런 상황이 의심스럽다면 Process:sqlserver:Private Bytes 와 SQL Server: Buffer Manager: Total pages 를 비교해보자

sp_configure 를 이용해 min server memory와 max server memory를 설ㅓㅈㅇ할 수 있다. 
```sql
EXEC sp_configure 'show advanced options', 1;
GO
RECONFIGURE;
GO
EXEC sp_configure 'min server memory';
EXEC sp_configure 'max server memory';
```
| name                   | minimum  | maximum       |config_value   |run_value  |
|:---:                   |:----:    |:----          |:----          |:----      |
| min server memory (MB) | 0        | 2147483647    | 0             | 16        |

| name                  | minimum   | maximum       |config_value   |run_value  |
|:---:                  |:----:     |:----          |:----          |:----      |
| max server memory (MB)| 128       | 2147483647    | 102400        | 102400    |

min server memory의 값이 0MB 이고 max server memory가 2147483647MB인것에 주의
max server memory를 10GB, min server memory를 5GB 로 세팅하는 예제
```sql
USE master;
EXEC sp_configure 'show advanced option', 1;
RECONFIGURE;
exec sp_configure 'min server memory (MB)', 5120;
exec sp_configure 'max server memory (MB)', 10240;
RECONFIGURE WITH OVERRIDE;
```
show_advanced option을 1로 킨 다음에 세팅해야 정상적으로 완료됨.
sys.configuration 뷰를 통해서도 메모리 세팅 값을 조회할수 디다.


메모리를 분석할수 있는 성능 모니터 카운터

| 오브젝트                  | Counter                   | 설명                                  |값                     |
|:---:                      |:----                      |:----                                  |:----                  |
| Memory                    | Availble Bytes            | Free physical Memory                  | OS 여유메모레         |
|                           | Pages/sec                 | Rate of hard page faults              | 보통 평균 < 50. 베이스라인 참고   |
|                           | Page Faults/sec           | Rate of total page faults             | 베이스라인 참고       |
|                           | Page Input/sec            | Rate of input page faults             |                       |
|                           | Page Output/sec           | Rate of output page faults                |                       |
|                           | Paging File %Usage Peak   | Peak values in the memory paging file     |                       |
|                           | Paging File: %Usag        | Rate of usage of the memory paging file   |                       |
| SQLServer:Buffer Manager  | Buffer cache hit ratio    | Percentage of requests served out of buffer cache |                       |
|                           | Page Life Expectancy      | 버퍼캐시에 머무루는 시간(초)              | 베이스라인  비교      |


