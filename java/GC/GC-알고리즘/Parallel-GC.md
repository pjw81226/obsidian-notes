# Parallel GC

## 특징
- HotSpot의 Throughput(처리량) 우선 GC(Parallel/Throughput Collector 라고도 부름)
    
- Stop-The-World(STW) 구간에서 여러 GC 스레드를 써서 GC 시간을 줄이는 방식
    
- Young GC뿐 아니라(기본) Old/Full GC도 병렬로 수행 가능(Parallel Old)
    
- **지연시간(응답시간) 보장보다는 전체 처리량 최대화에 초점**

## 동작 방식

**Young 영역(Minor GC)**
- 보통 “복사(카피) 기반”으로 Eden -> Survivor, 혹은 Old로 승격
- STW 동안 **여러 스레드가 병렬로** 객체 스캔, 복사, 정리

**Old 영역(Full GC / Major GC)**
- Old가 차거나 단편화가 심해지면 Full GC가 발생
- Mark(표시) -> Compact(압축) 계열 작업을 **STW 동안 병렬로** 수행(Parallel Old 사용 시)

핵심: 동시(concurrent)로 애플리케이션과 같이 도는 단계가 아니라, **멈춘 상태에서 빨리 끝내는** 쪽

## 사용 시나리오

- **배치 작업, ETL, 데이터 처리**처럼 “응답시간보다 총 처리량”이 중요한 경우
- **코어 수가 많고 CPU 여유가 있는 서버**에서 GC를 병렬화해 이득이 큰 경우    
- 가끔 멈춰도 괜찮고, 대신 평균 성능이 중요한 내부 서비스, 백오피스성 워크로드

반대로, 아래에는 비추천
- 지연시간 민감(짧은 pause를 안정적으로 원함)한 API 서버, 실시간 처리
- Full GC가 가끔 길게 터지는 것이 치명적인 서비스

## JVM 옵션

- `-XX:+UseParallelGC` : Parallel GC 사용(Young 중심, Old도 환경에 따라 병렬)
- `-XX:+UseParallelOldGC` : Old/Full GC도 병렬로(대개 함께 사용)

## 장단점

장점
- 멀티코어에서 GC 처리량이 좋고 평균 성능이 잘 나오는 편
- 튜닝 없이도 “대충 잘 돌아가는” 경우가 많음(Ergonomics가 잘 맞는 편)
- Old도 병렬로 돌리면 Full GC 시간이 줄어드는 경우가 있음

단점
- STW pause가 길어질 수 있음(특히 Full GC)
- 지연시간 목표를 “안정적으로” 맞추기 어렵고, 변동 폭이 커질 수 있음
- 코어를 많이 쓰기 때문에 CPU 경쟁이 생기면 애플리케이션 성능이 흔들릴 수 있음


## 관련 문서
- [[../GC]]

