---
sort: 1
---

# SQL Server 성능 튜닝
```
역자 의견
1장은 숲에 관한 이야기이다. 각각의 나무를 보는 법은 뒷장부터 소개하지만 여기서는 전체적인 숲을 보면서
쿼리 성능의 문제를 해결하는 큰 길을 보여주는 것이다.

초보가 볼때는 1장을 보면서 머리만 뒤죽박죽 될 수도 있을 것이다. 하지만 본인의 기술과 경력이 늘어 날수록 이 1장을 보는
시간이 점점 더 늘어날 것이기 때문에 계속 보고 또 보고 해야 할 장이다.
정 이해 안될때는 다음 장부터 보는것도 좋은 방법이다. 나중에 필요할 때만 1장을 찾아보자
```

SQL Server 버전 업그레이드는 내부적인 쿼리 엔진의 성능을 가져오기 때문에 많은 도움이 된다.

또한 오늘날은 VM 또는 Azure, AWS와 같은 가상 환경에서 작동되는 경우도 많고 기존과 다르게 개런티되지 않는 환경이 많이 존재. 하지만 그럴수록 기본은 더욱 중요.

이 책에서 다룰 쿼리 최적화 시 주요 작업
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
## 1.1 성능 튜닝 절차
성능 병목 식별 -> 이슈 우선 순위 정하기 -> 원인 트러블슈팅 -> 다른 해결책 적용 -> 성능 향상 수치화
  이 절차 반복    그리고 약간의 창의성 필요.

* 핵심 절차  
다음 질문을 스스로에게 해보자  
    - 같은 서버에 SQL Server와 리소스 경쟁하는 다른 어플리케이션이 존재하는지?  
        SQL Server보다 우선순위가 높은지 확인. ex) 작업관리자 -> 자세히 -> 컬럼 선택 -> Basic priority 체크
        기본적으로 sqlservr.exe는 노멀 우선순위인데 작업관리자(taskmgr.exe) 프로세스는 높음 우선순위이다.
        우리는 덜 중요한 어플리케이션이나 서비스가 SQL Server와 같은 머신에서 실행되며 자원 사용에 영향을 미치는지 조사 필요.
    - 하드웨어의 용량이 최대 워크로드를 견딜수 있는 상황인지?  
        CPU, 메모리, 디스크, 네트워크
    - SQL Server 구성은 정확한가?  
        sys.configuration 확인
        대부분 디폴트 값이 좋지만 일부분 성능에 영향을 미치는 부분은 언급할것. database 구성 옵션도 model DB에 적용하면 대부분 그 다음에는 만들어지는 것도 그것을 따라감.
    - VM같은 리소스 공유 환경에 내가 영향을 끼쳐 구성을 변경할수 있는지?
    - SQL Server 와 응용프로그램간에 연결은 효율적인지?
        적절한 네트워크 대역폭이 필요. 특히 클라우드 호스팅 환경이면 더 중요.
    - 데이터베이스 디자인은 최적화인지?
        디자인을 수정하는 것은 항상 가능하지 않지만 설계된 디자인을 잘 이해하는 것은 매우 도움이 된다.
    - SQL 쿼리를 구성하는 사용자 워크로드는 SQL Server 로드를 줄이게 최적화 되었는지?
        인덱스와 같은 것들이 적절하게 구성되어야
    - 어떤 프로세스들이 시스템 성능 저하를 일으키는지?
    - 워크로드가 동시성 필요레벨을 지원하는지?  
            
* 절차 반복(Iterating the Process)  
    성능 튜닝은 반복 작업
    - 주요 병목 식별하여 해결하고 변화에 대한 임팩트를 측정 그리고 다시 처음부터. 성능개선활동이 종료될때까지.
    - 해결책을 적용할때는 골든 룰이 있다. 가능한 1번에 1가지 변경만 적용하라. 어떤 변경사항이 시스템의 다른 부분에 영향을 미치는 일이 많기 때문.
    예) 특정 쿼리 성능 문제를 해결하기 위해 인덱스를 추가하면 그로 인해 다른 쿼리들이 느려질 수도.
    - 그렇기에 테스트 서버에서 충분히 테스트를 수행해야 한다.
    - 데이터와 사용량이 증가하면 기존의 쿼리의 실행계획 등이 변경되어 또 다시 나빠질 수 있기 때문에 주기적으로 성능 측정과 튜닝작업을 반복 필요.
    (그림 존재)

<br> 
## 1.2 성능 대 비용
일부 작업들은 약간의 성능 향상을 얻기 위해 많은 시간과 돈이 소비될 수도 있기 때문에 좀더 효율적인 방법을 선택할 수 있어야 한다.

    - 얼마만큼의 성능이면 받아들일 수 있는지
    - 이 작업이 성능향상의 가치가 있는지?
    
* 성능 목표  
    최대 효율을 얻기 위해서 성능 목표치를 정해야 함.  
    베스트 프랙티스가 이미 많이 존재하기 때문에 우리에 맞게 적용하기 전 얼마만큼의 수치면 만족할 수 있을지 정해야 함?
    보통 개발자나 비지니스 수행자들에게 목표를 정하게 하는게 좋은 자세.  
    쿼리 튜닝 같은 개선 활동도 적은 비용으로 효과를 얻고 하드웨어 리소스 투자 같은 방법도 빠르게 성능 향상을 얻을 수 있다.
    단 쿼리 튜닝은 적은 노력으로 큰 효과를 얻지만 전문가가 필요하고 하드웨어 투자는 빠른 효과를 얻지만 큰 비용을 든다. 잘 판단. 경험치 능력자가 매우 중요.
    
* "이정도면 충분" 튜닝
    80:20 법칙이 여기서도 통함. 상위 몇개의 악성 쿼리들만 해결하면 80%의 문제가 해결된다. 이 때 노력은 20%만 소요.  최대 효율을 얻기 위해서는 목표치를 세우고 20%의 노력으로 80%의 문제점을 해결하는 방향으로 하려면 "이정도면 충분" 방법으로 가야.
    
    들인 비용과 성능 개선과의 관계는 반비례 그래프와 같다.
    초기에는 적은 노력으로 큰 향상을 얻지만 나중에는 큰 노력과 비용으로도 적은 결과만 얻게 되니 중간에 "이정도면 충분" 지점에서 끊어야 함.

<br> 
## 1.3 성능 베이스라인
성능 분석의 주요 목표는 여러가지 하드웨어와 소프트웨어 상에서 시스템의 사용과 부하 패턴을 이해하는 것이다.

* 이로 인해 아래의 도움을 얻을 수 있다.
    - 리소스 병목 분석 가능
    - 이전에 측정된 베이스라인과 비교하여 얼마만큼 사용량이 증가하여 트러블슈팅할때 이득
    - 리소스 사용량 계획과 하드웨어 업그레이드 주기 정확한 측정 가능
    - 피크타임을 피하여 데이터베이스 관리 활동하는 시간을 정할수 있음.
    - 특정 수치는 이전 값과 비교할 때만 의미가 있을 수 있다. 예) PLE
    
* 베이스라인이 있으면 가능한 일
    - 현재 성능을 측정하고 어플리케이션 성능 목표치를 세우기 가능
    - 다른 하드웨어와 소프트웨어 조합을 비교 가능
    - 어떻게 사용량과 데이터가 변화하는지 측정
    - 현재 값이 이상하게 튀는 데 진짜 이상 상황인지 아니면 평소와 같은 정상 수치인지
    - 어플리케이션의 피크와 비 피크 사용 패턴 이해. 효과적인 데이터베이스 관리작업 가능. 예를 들면 비 피크타임 일때 백업이나 조각모음을 한다든지
    
    SQL Server의 하드웨어 소프트웨어 리소스 사용량 베이스라인 만들기 위해 성능모니터 이용할 수 있다.  
    또는 DMV나 DMF Xevent를 사용하여 쿼리 부하치 
    또는 기타 툴 MS의 SQLIO, SQLIOSim(시뮬레이션) or Distributed Replay.

<br> 
## 1.4 어디에 노력을 집중 해야?
하드웨어와 OS쪽은 얻는 결과가 작으나 큰 비용이 들며 인덱스나, 쿼리 코딩쪽은 작은 비용이 소요되지만 가장 큰 효과를 본다.    
    
| 항목 | 개선효과 | 응답자 |
|:---:|:----:|:----|
| CPU 전원 절약                | 2% | 6 |
| 다른 하드웨어나 OS 이슈      | 2% | 7 |
| 가상화                       | 2% | 7 |
| SQL Server/데이터베이스 구성 | 3% | 10 |
| 오래되거나 누락된 통계        | 9% | 31 |
| 데이터베이스/테이블 디자인    | 10% | 38 |
| 어플리케이션 코드             | 12% | 43 |
| I/O 서브시스템 문제           | 16% | 60 |
| 잘못된 인덱스 전략            | 19% | 68 |
| T-SQL 코드                   | 26% | 94 |

T-SQL 코드 재작성 이나 인덱스를 적용하는게 실제 가장 큰 효과를 볼 수 있다.  
또한 가장 적은 비용으로 할 수 있고 위쪽으로 갈수록 비용이 많이 든다. 그렇기 때문에 우리는 아래쪽 방법에서 점차 위로 접근해 가장 좋은 솔루션을 선택해야 하는 것이다.

SQL Server 데이터베이스를 이용하는 어플리케이션 쪽에서 2단계로 스트레스 분석해야 한다.
* 고수준 : 각각의 하드웨어 자원과 SQL Server 설치에 걸쳐 얼마나 많은 스트레스가 있는지 측정. 가장 좋은 것은 다양한 대기 상태를 측정하는 것.
            이로 인해 두가지 방법으로 도움을 얻는다.
    - 첫째, 나쁜 성능의 SQL Server 어플리케이션의 어느 부분에 집중해야 하는지 알수 있고
    - 둘째, 적절하지 않은 구성 식별 도움.

    이로 인해 성능 모니터를 사용하여 어플리케이션을 튜닝할 수 없을 경우 어떤 하드웨어 자원을 업그레이드해야 결정 할때 도움.
            
* 저수준 : 범인이 되는 쿼리 식별. Xevent와 DMV 를 통해 알수 있다.

<br> 
## 1.5 SQL Server 성능 저하 주범들
* 주요 SQL Server 성능 저하 요인들
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




