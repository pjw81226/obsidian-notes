# G1GC (Garbage First GC)

## 특징
- 목표는 예측 가능한 짧은 pause: 처리량도 챙기지만, 기본 철학은 **지연시간 제어**
- 힙을 Region으로 쪼개서 관리(연속된 Young/Old 구분 대신 리전 단위로 유연하게 구성)
- Garbage First = 쓰레기가 많은 리전부터 우선 수거해서 효율을 올림
- Old 영역 정리는 Concurrent Marking 을 활용해서 Full GC 빈도를 낮추는 쪽


## 동작 방식
### Region 기반 메모리 관리
- 힙을 동일 크기의 리전으로 나눔 (대략 1MB~32MB 범위에서 JVM이 자동 결정)
- 각 리전은 상황에 따라 Eden / Survivor / Old 역할을 그때그때 맡음  


### 컬렉션 흐름 핵심
1. Young GC (Evacuation Pause, STW)
    - Eden/Survivor 리전의 살아있는 객체를 다른 리전으로 복사하며 정리
    - STW지만 pause 목표(MaxGCPauseMillis) 에 맞춰 작업량(수거할 리전/객체)을 조절하려고 함
        
2. Concurrent Marking (Old 추적, 대부분 동시)
    - Old에 어떤 객체가 살아있는지 표시(mark)를 애플리케이션과 동시에 진행
    - 중간에 짧은 STW 구간(Initial Mark, Remark)이 끼지만 CMS 대비 목표는 관리 가능한 수준
        
3. Mixed GC (STW, Young + 일부 Old 리전)
    - 마킹 결과를 바탕으로 “가비지가 많은 Old 리전”을 골라 Young GC에 섞어서 함께 수거
    - 여기서 “Garbage First”가 본격적으로 발동: 회수 효율 높은 Old 리전부터 처리
        
4. (가능하면) Full GC 회피
    - G1은 “동시 + 리전 선택 + 복사”로 Full GC를 최대한 피하려고 하지만 메모리 압박/단편화/휴먼거스이슈로 Full GC가 날 수도 있음

- **Remembered Set(RSet)**: 리전 간 참조를 추적해, 필요한 리전만 스캔해서 pause를 줄임(대신 오버헤드 존재)
    
- **Humongous Object(대형 객체)**: 리전 크기의 절반 이상인 큰 객체는 “휴먼거스”로 취급되어 별도 방식으로 할당/회수 
	- 즉, 큰 배열/버퍼를 자주 만들면 G1 성능이 흔들릴 수 있음


## 사용 시나리오

- API 서버/일반 웹서비스처럼 가끔 긴 멈춤이 치명적이고, 평균/최대 지연시간을 관리하고 싶은 경우
- 힙이 커지면서 Parallel GC에서 Full GC pause가 길어지는 상황의 대안
- “튜닝은 최소로, 기본값으로도 안정적으로” 가고 싶을 때(특히 Java 11/17/21 등 LTS)

반대로 주의할 경우
- 힙이 매우 작거나(오버헤드 대비 이득 적음), 초고처리량 배치만 최우선이면 Parallel이 더 단순/유리할 때도 있음
- 대형 객체(큰 배열/버퍼) 할당이 매우 빈번하면 휴먼거스 이슈로 튜닝이 필요할 수 있음

## JVM 옵션
`-XX:+UseG1GC`

pause 목표/시작 트리거
- `-XX:MaxGCPauseMillis=200` : 목표 pause(ms). “보장”이 아니라 **목표치**
 - `-XX:InitiatingHeapOccupancyPercent=45` : 힙 사용률이 이 정도면 동시 마킹을 시작

리전/Young 비율(필요 시)
- `-XX:G1HeapRegionSize=8m` : 리전 크기(고정). 보통은 자동이 낫지만, 특수 케이스에 사용
- `-XX:G1NewSizePercent=20` / `-XX:G1MaxNewSizePercent=60` : Young 비율 범위

여유 공간/혼합 수거 튜닝(문제 있을 때만)
- `-XX:G1ReservePercent=10` : evacuation 실패 방지를 위한 “여유 리전” 비율
- `-XX:G1MixedGCCountTarget=8` : Mixed GC를 몇 번에 걸쳐 Old를 처리할지 목표
    
스레드/부가 옵션
- `-XX:ParallelGCThreads=<n>` / `-XX:ConcGCThreads=<n>`
- `-XX:+ParallelRefProcEnabled` : 참조 처리 병렬화로 pause 감소에 도움
- `-XX:+UseStringDeduplication` : 문자열 중복 제거(메모리 절약, CPU 약간 사용)
    
로그(자바 9+)
- `-Xlog:gc*,gc+heap=info,gc+pause=info,safepoint`

## 장단점

장점
- pause 예측/관리에 강함 **(길게 한 방을 줄이는 방향)**
- Old를 동시 마킹 + Mixed GC로 나눠 처리해서 Full GC 빈도/충격을 줄이기 쉬움    
- 최신 JVM에서 기본으로 많이 쓰여 운영 사례/가이드가 풍부

단점
- RSet 등으로 인해 메모리/CPU 오버헤드가 있음(특히 쓰기(write) 많은 워크로드)
- 휴먼거스 객체(큰 배열/버퍼) 패턴에 민감할 수 있음
- 목표 pause를 너무 낮게 잡으면(예: 10ms) 처리량이 떨어지거나, 오히려 흔들릴 수 있음
- 최악의 경우엔 여전히 Full GC가 발생할 수 있음(운영에서 로그로 감시 필요)

## 관련 문서
- [[../GC]]

