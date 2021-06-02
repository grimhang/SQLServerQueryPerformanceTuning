---
sort: 5
---

# Network Performance Analysis

## 5.1 Network 병목 분석    
SQL Server OLTP 프로덕션 환경에서 네트워크와 관련된 문제는 상대적으로 매우 적다. 네트워크와 관련된 대부분의 경우 하드웨어와 드라이버 또는 네트웍 장비에 관련된다. 
이러한 문제의 대부분은 네트워크 모니터 도구로 가장 잘 진단 할 수 있습니다. 그러나 성능 모니터에서도 아래와 같은 수집할 수 있는 항목이 있다.

### Network 분석 카운터

```
Object(Instance)                 Counter             Description                             Value
-------------------------------  --------------      -------------------------------------   --------------------------------------
Network Interface(Network card)  Bytes Total/sec     NIC에서 전송된 총 바이트                  50% < NIC 용량. 하지만 베이스라인 참고
Network Segment                  % Net Utilization   네트워크 세그먼트에서 사용중 대역폭 %      평균 < 80%. 하지만 베이스라인 참고
```


여기