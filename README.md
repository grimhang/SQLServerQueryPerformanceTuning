# MS SQL Server Query Performance Tuning


이 문서는 Grant Fritchey의 SQL Server Query Performance Tuning을 번역한 것을 바탕으로 하였다.  
원본책이 SQL Server 2014 버전으로 쓰여져 있어서 하다가 중간에 2017버전으로 교체했고 내용도 중간에 약간 버전업되었으며 그래도 오래된 내용이나 나와 견해가 틀린 일부는 내 생각 위주로 대체하였다.  
원본 책 내용 80%, 내가 바꾼거나 추가한 게 20%쯤.

매우 좋은 책이다.  
내가 20년동안 머리속에서만 생각하고 정리하지 않았던 것들을 저자가 대신 모두 정리해 준 느낌이다.  
아무에게도 얘기하지 않았던 내 머리속의 생각/지식들을 이 책에서 정리/나열한 느낌이어서 전율이 일어났다.  
또한 내 기술 수준이 높은 단계에 이르렀단 것은 알았지만 실제 어느정도인지 객관적으로 검증해 주거나 설명해준 사람이 없었는데(당연하지만), 이 책의 저자한테 검증 받은 것 같았다.

하지만 이 책에서 제일 높이 평가하고 싶은 점은 이해하기 쉬운 영어로 쓰여졌단 것이다. 저자가 유머와 이상한 비유를 들어서 설명하는 책이 많은데 같은 언어권에 사는 사람이었으면 위트있는 좋은 책이라고 느꼈을 지라도 나처럼 외국인의 경우 이해하기 어려울 수 밖에 없다. 하지만 이책은 그런 것들을 최대한 포기하고 쉬운 문법으로 쓰여져 있기 때문에 더욱 더 존경받을 만 하다.


정말 좋은 책이지만 내가 저작권을 어길 수 없기 때문에 아쉽게도 정리본 정도로만 기록할 것이고 혹시나 나중에 문제 되면 말없이
삭제 예정이다.

문의 : 박성출 grimhan골뱅이daum.net

SQL Server 2017 Query Performance Tuning: Troubleshoot and Optimize Query Performance 5th ed. Edition [book link](https://www.amazon.com/Server-2017-Query-Performance-Tuning/dp/1484238877)

## 책의 목차 안내

1. SQL Query 성능 튜닝
2. Memory 성능 분석
3. Disk 성능 분석
4. CPU 성능 분석
5. Network 성능 분석
6. SQL Server 전체 성능
7. Baseline 생성
8. Query 성능 수집
9. Query 성능 분석
10. Index Architecture and Behavior 
11. Index Analysis
12. Database Engine Tuning Advisor
13. Key Lookups and Solutions
14. Statistics, Data Distribution, and Cardinality
15. Index Fragmentation
16. Execution Plan Generation
17. Execution Plan Cache Behavior
18. Parameter Sniffing
19. Query Recompilation
20. Query Design Analysis
21. Reduce Query Resource Use
22.  Blocking and Blocked Processes
23. Causes and Solutions for Deadlocks
24. Row-by-Row Processing
25. Memory-Optimized OLTP Tables and Procedures
26. Database Performance Testing
27. Database Workload Optimization
28.  SQL Server Optimization Checklist

