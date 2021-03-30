# MS SQL Server Query Performance Tuning


이 문서는 Grant Fritchey의 SQL Server Query Performance Tuning을 번역한것 것을 바탕으로 하였다.  
아무래도 SQL Server 2014 버전으로 쓰여져 있어서 그 중 오래된 내용을 삭제하고 최신 내용으로 대체하였으며 일부는 나와 견해가 틀린것들이 있었는데 내 생각을 우선으로 대체하였다. 한 원본 책 내용 80%, 오래되거나 내것으로 바꾼게 20%쯤.

매우 좋은 책이다.  
내가 20년동안 머리속에서만 생각하고 정리하지 않았던 것들을 저자가 대신 모두 정리해 준 느낌이다.  
아무에게도 얘기하지 않았던 내 머리속의 생각/지식들을 이 책에서 정리/나열한 느낌이어서 전율이 일어났다.  
또한 내 기술 수준이 높은 단계에 이르렀단 것은 알았지만 실제 어느정도인지 객관적으로 검증해 주거나 설명해준 사람이 없었는데(당연하지만), 이 책의 저자한테 내 수준이 충분히 높다는걸 검증 받은 것 같았다.

하지만 기술도 기술이지만 이 책에서 제일 높이 평가하고 싶은 점은 이해하기 쉬운 영어로 쓰여졌단 것이다. 저자가 유머와 이상한 비유를 들어서 설명하는 책이 많은데 같은 언어권에 사는 사람이었으면 위트있는 좋은 책이라고 느꼈을 지라도 나처럼 외국인의 경우 이해하기 어려울 수 밖에 없다. 하지만 이책은 그런 것들을 최대한 포기하고 쉬운 문법으로 쓰여져 있기 때문에 더욱 더 존경받을 만 하다.


정말 좋은 책이지만 내가 저작권을 어길 수 없기 때문에 아쉽게도 정리본 정도로만 기록할 것이고 혹시나 나중에 문제 되면 말없이
삭제 예정이다.


SQL Server Query Performance Tuning 4th Edition [book link](https://www.amazon.com/SQL-Server-Query-Performance-Tuning-ebook/dp/B01JC6P8MC)

## 책의 목차 안내

1. SQL Query Performance Tuning
2. Memory Performance Analysis
3. Disk Performance Analysis
4. CPU Performance Analysis
5. Creating a Baseline
6. Query Performance Metrics
7. Analyzing Query Performance
8. Index Architecture and Behavior 
9. Index Analysis
10. Database Engine Tuning Advisor
11. Key Lookups and Solutions
12. Statistics, Data Distribution, and Cardinality
13. Index Fragmentation
14. Execution Plan Generation
15. Execution Plan Cache Behavior
16. Parameter Sniffing
17. Query Recompilation
18. Query Design Analysis
19. Reduce Query Resource Use
20.  Blocking and Blocked Processes
21. Causes and Solutions for Deadlocks
22. Row-by-Row Processing
23. Memory-Optimized OLTP Tables and Procedures
24. Database Performance Testing
25. Database Workload Optimization
26.  SQL Server Optimization Checklist

