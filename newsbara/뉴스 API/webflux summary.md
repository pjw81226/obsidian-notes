# WebFlux 비동기 전환 요약본

## 🎯 핵심 요약

### 왜 비동기로 전환하는가?

현재 시스템은 RestTemplate과 Thread.sleep()을 사용하는 **동기 블로킹 방식**입니다. 이는 다음과 같은 문제를 야기합니다:

1. **스레드 고갈**: 동시 요청 100개면 100개의 스레드가 필요하며, 각 스레드는 약 1MB의 메모리를 점유합니다.
2. **비효율적 대기**: 네이버 API 응답을 기다리는 동안 스레드가 idle 상태로 낭비됩니다.
3. **처리량 제한**: Tomcat의 기본 스레드 풀(200개)이 상한선이 되어 확장성이 제한됩니다.

비동기 전환 시, 동일한 작업을 **10~20개의 스레드**로 처리할 수 있으며, 메모리 사용량도 1/5 수준으로 감소합니다.

---

## 📊 Before vs After

### 현재 (동기)
- **서버**: Tomcat (Servlet Container)
- **HTTP 클라이언트**: RestTemplate
- **Redis**: StringRedisTemplate (동기)
- **스레드 모델**: 요청당 1 스레드
- **대기 방식**: Thread.sleep() - 스레드 블로킹
- **동시 처리**: 스레드 풀 크기만큼 (기본 200개)

### 목표 (비동기)
- **서버**: Netty (Event Loop 기반)
- **HTTP 클라이언트**: WebClient
- **Redis**: ReactiveRedisTemplate
- **스레드 모델**: Event Loop (CPU 코어 * 2개)
- **대기 방식**: 논블로킹 - delayElements()
- **동시 처리**: 수천~수만 개 (리소스 허용 한도까지)

---

## 🔄 Reactive Streams 핵심 개념

### Mono와 Flux
- **Mono**: 0~1개의 결과를 비동기로 반환하는 Publisher
- **Flux**: 0~N개의 결과를 스트림으로 반환하는 Publisher

### 구독(Subscription) 방식
Reactive는 **선언형 프로그래밍**입니다. 메서드 호출만으로는 실행되지 않고, 누군가 구독(subscribe)해야 비로소 실행됩니다. Spring WebFlux는 Controller에서 Mono나 Flux를 반환하면 자동으로 구독합니다.

### 비동기 파이프라인
데이터 변환과 처리를 연결(chaining)하여 파이프라인을 구성합니다:
- **map**: 동기 변환 (String을 Integer로)
- **flatMap**: 비동기 변환 (Mono를 다른 Mono로)
- **filter**: 조건부 필터링
- **switchIfEmpty**: 값이 없을 때 대체
- **onErrorResume**: 에러 발생 시 복구

---

## 🛠️ 전환 과정 (5단계)

### Phase 1: 준비
**의존성 추가**를 통해 WebFlux와 Reactive Redis를 프로젝트에 포함합니다. Spring Boot는 Web과 WebFlux를 동시에 지원하므로 점진적 전환이 가능합니다.

**설정 파일 작성**으로 WebClient와 ReactiveRedis의 Connection Pool, 타임아웃 등을 설정합니다.

### Phase 2: 외부 API 클라이언트
**NaverNewsClient**를 RestTemplate에서 WebClient로 전환합니다. 
- 반환 타입이 NaverSearchResponse에서 Mono<NaverSearchResponse>로 변경됩니다.
- 에러 처리는 try-catch 대신 onErrorMap()을 사용합니다.
- Connection Pool은 Reactor Netty가 자동으로 관리하며, 최대 연결 수, Idle 타임아웃 등을 설정할 수 있습니다.

### Phase 3: Reactive Redis 및 분산 락
**StringRedisTemplate을 ReactiveStringRedisTemplate로 전환**합니다. 모든 Redis 작업이 Mono 또는 Flux를 반환하게 됩니다.

**RedisLockUtil도 비동기로 전환**하여 락 획득/해제가 논블로킹으로 동작합니다. 핵심은 Thread.sleep()을 제거하고 delayElements()로 대체하는 것입니다. 재시도 로직도 명시적 for 루프 대신 repeatWhenEmpty() 연산자를 사용합니다.

### Phase 4: 서비스 레이어
**KeywordNewsService**를 완전히 비동기로 전환합니다. 모든 메서드가 Mono나 Flux를 반환하도록 변경합니다.

**JPA는 여전히 블로킹**이므로 subscribeOn(Schedulers.boundedElastic())를 사용하여 별도 스레드 풀에서 실행합니다. 진정한 논블로킹을 원한다면 R2DBC로 전환해야 하지만, 당장은 필수가 아닙니다.

**캐시 조회 → 락 획득 → API 호출 → 캐시 저장** 흐름을 flatMap 체이닝으로 구성합니다. 락 해제는 doFinally()를 사용하여 성공/실패/취소 모든 경우에 실행되도록 보장합니다.

### Phase 5: Controller
Controller의 반환 타입을 Mono 또는 Flux로 변경합니다. Spring WebFlux는 이를 자동으로 구독하고 HTTP 응답으로 변환합니다.

---

## ⚠️ 주의사항

### 절대 하지 말아야 할 것

**1. block() 호출 금지**
Reactive 체인 중간에서 block()을 호출하면 스레드가 블로킹되어 비동기의 이점이 사라집니다. 특히 Reactor 스레드(reactor-http-nio)에서 block()을 호출하면 IllegalStateException이 발생합니다.

**2. ThreadLocal 사용 금지**
작업이 여러 스레드를 넘나들기 때문에 ThreadLocal은 의도대로 작동하지 않습니다. Context를 사용하여 데이터를 전파해야 합니다.

**3. 구독 누락**
Mono나 Flux를 생성만 하고 구독하지 않으면 아무 일도 일어나지 않습니다. Controller에서 반환하거나 명시적으로 subscribe()를 호출해야 합니다.

### 꼭 알아야 할 것

**1. map vs flatMap**
- map은 동기 변환(Mono를 Mono 로)에 사용
- flatMap은 비동기 변환(Mono를 다른 Mono로)에 사용
- flatMap 안에서 또 다른 Mono를 반환할 때 사용

**2. Scheduler 선택**
- parallel: CPU 집약적 작업 (코어 수만큼 스레드)
- boundedElastic: I/O 블로킹 작업 (JPA, 파일 I/O 등, 최대 10 * 코어 수)
- single: 순차적으로 실행해야 하는 작업

**3. 에러 처리**
- onErrorResume: 에러 발생 시 대체 Mono 반환
- onErrorReturn: 에러 발생 시 기본값 반환
- onErrorMap: 예외를 다른 예외로 변환
- retry: 재시도

---

## 🎓 핵심 패턴

### 캐시 확인 후 처리
캐시를 먼저 확인하고, 없으면 실제 작업을 수행하는 패턴입니다. switchIfEmpty()를 사용하여 간결하게 표현할 수 있습니다.

### 분산 락 with 재시도
락을 획득하고, 작업을 수행한 후, 반드시 락을 해제해야 합니다. doFinally()를 사용하면 어떤 경우에도 락 해제가 보장됩니다.

### 블로킹 작업 격리
JPA처럼 피할 수 없는 블로킹 작업은 Mono.fromCallable()로 감싸고 subscribeOn(Schedulers.boundedElastic())를 사용하여 별도 스레드 풀에서 실행합니다.

### 병렬 처리
여러 독립적인 작업을 동시에 수행하려면 Mono.zip()이나 Flux.parallel()을 사용합니다. 순차 실행보다 월등히 빠릅니다.

---

## 🔍 성능과 트레이드오프

### 성능 향상
- **처리량**: 3~10배 증가 (I/O 대기가 많은 경우)
- **메모리**: 1/3~1/5 감소 (스레드 수 감소)
- **응답 시간**: 동시 부하 시 크게 개선

### 비용
- **코드 복잡도**: 2~3배 증가
- **디버깅 난이도**: 상승 (스택 트레이스가 복잡)
- **학습 곡선**: 가파름 (팀 전체가 Reactive 이해 필요)

### 언제 전환해야 하는가?
- 동시 요청이 많은 서비스 (100+ concurrent)
- 외부 API 호출이 빈번한 서비스
- I/O 대기 시간이 긴 서비스

### 언제 전환하지 말아야 하는가?
- 트래픽이 적은 서비스 (동시 요청 < 50)
- CPU 집약적 작업이 주를 이루는 경우
- 팀의 Reactive 경험이 전혀 없는 경우
- 단순 CRUD 중심의 서비스

---

## 🧩 Backpressure (배압)

### 배압이란?
생산자(Publisher)가 데이터를 빠르게 생성하는데, 소비자(Subscriber)가 처리 속도를 따라가지 못하는 상황입니다. 이를 방치하면 메모리 부족(OutOfMemoryError)이 발생합니다.

### 해결 방법
- **buffer**: 일정 개수를 모아서 배치 처리
- **sample**: 일정 시간마다 최신 값만 가져옴
- **onBackpressureBuffer**: 버퍼 크기 제한, 초과 시 에러
- **onBackpressureDrop**: 처리 못하는 데이터는 버림

우리 시스템에서는 대량의 뉴스 항목을 처리할 때 onBackpressureBuffer를 사용하여 최대 버퍼 크기를 제한할 수 있습니다.

---

## 🔧 트러블슈팅

### 자주 발생하는 에러

**IllegalStateException: block() are blocking**
Reactor 스레드에서 block()을 호출했을 때 발생합니다. 모든 메서드를 Mono/Flux 반환으로 변경해야 합니다.

**Subscription not happening**
Mono를 생성만 하고 구독하지 않아서 실행되지 않습니다. Controller에서 반환하거나 subscribe()를 명시적으로 호출해야 합니다.

**Memory Leak**
무한 스트림(Flux.interval)을 구독 해제하지 않으면 메모리 누수가 발생합니다. Disposable을 저장해두고 나중에 dispose() 호출하거나, take()로 개수를 제한해야 합니다.

### 디버깅 팁
- **log()**: 모든 Reactive 이벤트를 로깅
- **checkpoint()**: 에러 발생 지점을 스택 트레이스에 표시
- **Hooks.onOperatorDebug()**: 전역 디버그 모드 활성화 (성능 저하 있음)

---

## 📅 실전 마이그레이션 로드맵

### Week 1: 준비
팀 교육, 문서 학습, 의존성 추가, 기본 설정 작성

### Week 2: 외부 API
NaverNewsClient를 WebClient로 전환, 단위 테스트 작성

### Week 3: 인프라
ReactiveRedis 설정, ReactiveRedisLockUtil 구현, 락 동작 검증

### Week 4: 서비스
KeywordNewsService 전환, JPA 블로킹 코드 격리, 로직 검증

### Week 5: 통합
Controller 전환, E2E 테스트, 성능 벤치마크

### Week 6: 운영 준비
모니터링, 알람, 롤백 계획, 스테이징 배포

### Week 7: 배포
카나리 배포, 성능 모니터링, 이슈 대응, 점진적 트래픽 전환

---

## 💡 FAQ

**Q: WebFlux로 전환하면 무조건 빠른가요?**
아니요. I/O 대기가 많은 경우에만 효과적입니다. CPU 집약적 작업은 오히려 느릴 수 있습니다.

**Q: JPA를 R2DBC로 전환해야 하나요?**
필수는 아닙니다. subscribeOn(Schedulers.boundedElastic())로 JPA를 별도 스레드에서 실행 가능합니다. 하지만 진정한 논블로킹을 원한다면 R2DBC 고려하세요.

**Q: 기존 코드를 한 번에 전환해야 하나요?**
아니요. Spring Boot는 Web과 WebFlux 공존을 지원합니다. 점진적 전환이 가능하고 권장됩니다.

**Q: 디버깅이 너무 어려운데요?**
log(), checkpoint(), Reactor DevTools를 활용하세요. 초기에는 학습 곡선이 있지만 익숙해지면 괜찮습니다.

**Q: 언제 비동기 전환을 하지 말아야 하나요?**
트래픽이 적은 서비스, 팀의 Reactive 경험이 전혀 없는 경우, CRUD 중심의 단순한 서비스에서는 오버엔지니어링일 수 있습니다.

---

## ✅ 최종 체크리스트

### 전환 전
- 현재 병목 지점 파악 및 측정
- 비동기 전환의 명확한 목표와 기대 효과 설정
- 팀 교육 및 Reactive 학습 완료
- 충분한 테스트 환경과 롤백 계획 준비

### 전환 중
- 각 Phase마다 단위 테스트와 통합 테스트 작성
- 성능 벤치마크로 개선 효과 측정
- 메모리 프로파일링으로 누수 확인
- 에러 처리와 타임아웃 검증

### 전환 후
- 설정한 성능 목표 달성 여부 확인
- 모니터링 대시보드와 알람 설정
- 운영 가이드와 트러블슈팅 문서 작성
- 팀 회고로 개선점 도출 및 공유

---

## 🎯 핵심 메시지

비동기 전환은 **은총알(Silver Bullet)이 아닙니다**. 복잡도가 증가하는 만큼, 명확한 이득이 있을 때만 시도해야 합니다.

**I/O 대기가 많고, 동시 요청이 많은 시스템**에서는 큰 효과를 볼 수 있습니다. 하지만 **팀의 학습 비용, 유지보수 복잡도, 디버깅 난이도**를 충분히 고려해야 합니다.

**점진적 전환**을 권장합니다. 한 번에 모든 것을 바꾸지 말고, 병목이 되는 부분부터 차근차근 전환하세요. 각 단계마다 테스트하고, 측정하고, 검증하세요.

**모니터링은 필수**입니다. 비동기 시스템은 문제가 발생했을 때 원인 파악이 어렵습니다. 충분한 로깅, 메트릭, 트레이싱을 준비하세요.

마지막으로, **팀원 모두가 Reactive를 이해해야** 합니다. 한 사람만 알고 있으면 그 사람이 병목이 됩니다. 충분한 교육과 문서화에 투자하세요.

---

**비동기 전환은 도전적이지만, 제대로 구현하면 시스템의 확장성과 효율성을 크게 향상시킬 수 있습니다. 천천히, 확실하게 진행하세요!** 🚀
