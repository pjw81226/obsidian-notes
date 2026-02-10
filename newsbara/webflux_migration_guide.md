# RestTemplate → WebClient 전환 가이드

> ⚠️ **주의**: 이 프로젝트는 **비동기 전환을 하지 않았습니다**.  
> WebClient를 사용하지만 `block()`으로 **동기 호출**합니다.

---

## 📋 핵심 요약

| 항목 | 이전 | 현재 |
|-----|-----|------|
| HTTP 클라이언트 | RestTemplate | WebClient + `block()` |
| 호출 방식 | 동기 | **동기 (변경 없음)** |
| Service 반환 타입 | 일반 객체 | 일반 객체 (Mono 아님) |
| Controller 반환 타입 | 일반 객체 | 일반 객체 (Mono 아님) |

**결론: WebClient를 쓰지만 코드 흐름은 동기입니다.**

---

## 🤔 왜 RestTemplate → WebClient로 바꿨나?

### 1. RestTemplate은 유지보수 모드 (deprecated 예정)
```
Spring 5.0부터 RestTemplate은 "maintenance mode"
Spring에서 공식적으로 WebClient 사용 권장
```

### 2. WebClient의 장점 (동기로 써도)
| 항목 | RestTemplate | WebClient |
|-----|-------------|-----------|
| Connection Pool | 수동 설정 필요 | **자동 관리** |
| 타임아웃 설정 | 복잡 | **간단** |
| 비동기 전환 | 코드 재작성 | **block() 제거만 하면 됨** |
| 에러 처리 | try-catch | 체이닝 가능 |

### 3. 미래 확장성
```java
// 현재: 동기
webClient.get()
    .retrieve()
    .bodyToMono(Response.class)
    .block();  // 여기서 블로킹

// 나중에 비동기 전환 시: block()만 제거
webClient.get()
    .retrieve()
    .bodyToMono(Response.class);  // Mono 그대로 반환
```

---

## 🚫 왜 비동기로 전환 안 했나?

### 1. JPA와의 호환성 문제
```
JPA/Hibernate = 블로킹 라이브러리
Mono/Flux 안에서 JPA 호출하면 → Carrier Thread 블로킹
비동기 장점 사라짐
```

**진짜 비동기 하려면:**
- JPA → R2DBC 전환 필요 (큰 공수)
- 모든 Repository 재작성

### 2. 복잡도 증가
```java
// 동기 (현재): 읽기 쉬움
User user = userRepository.findById(id);
News news = naverClient.search(keyword);
return new Response(user, news);

// 비동기: 복잡
return userRepository.findById(id)
    .flatMap(user -> naverClient.search(keyword)
        .map(news -> new Response(user, news)));
```

### 3. 학습 비용
- Reactive Programming 학습 필요
- 디버깅 어려움
- 팀 전체 학습 필요

### 4. 트래픽 규모
- 현재 트래픽으로는 동기 방식으로 충분
- Virtual Thread로 동시성 문제 해결

---

## ✅ 대신 Virtual Thread 활성화

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

**효과:**
- 동기 코드 그대로 유지
- Thread.sleep(), 블로킹 I/O에서 효율적 동작
- 코드 변경 없이 동시 처리량 증가

---

## 📁 변경된 파일

### 삭제
| 파일 | 이유 |
|-----|------|
| `RestTemplateConfig.java` | WebClient로 대체 |
| `ReactiveRedisConfig.java` | 비동기 Redis 사용 안 함 |
| `thumbnail/` 디렉토리 | 중복 코드 |

### 수정
| 파일 | 변경 내용 |
|-----|----------|
| `NaverNewsClient.java` | RestTemplate → WebClient |
| `KeywordNewsService.java` | 분산 락 로직 개선 (RedisLockUtil) |
| `application.yaml` | Virtual Thread 활성화 |

---

## 🔮 언제 진짜 비동기로 전환하나?

### 전환 시점
- 동시 요청 100+ 이상
- 외부 API 호출이 매우 빈번
- I/O 대기 시간이 긴 작업 다수

### 전환 시 필요한 작업
1. JPA → R2DBC 전환
2. Redis → ReactiveRedisTemplate 전환
3. Controller 반환 타입 `Mono<>` / `Flux<>`
4. Service 반환 타입 `Mono<>` / `Flux<>`
5. 테스트 코드 전체 재작성

---

## 📊 현재 아키텍처

```
[Controller] - 동기
     ↓
[Service] - 동기, JPA 사용
     ↓
[NaverNewsClient] - WebClient.block() (동기)
     ↓
[네이버 API]
```

**핵심: 전체가 동기, WebClient만 도구로 사용**
