# Spring WebFlux 비동기 전환 가이드

## 📋 목차

1. [현재 시스템 분석](#1-현재-시스템-분석)
2. [비동기 전환 목표](#2-비동기-전환-목표)
3. [아키텍처 변경 개요](#3-아키텍처-변경-개요)
4. [단계별 전환 과정](#4-단계별-전환-과정)
5. [코드 예시](#5-코드-예시)
6. [주의사항 및 트레이드오프](#6-주의사항-및-트레이드오프)
7. [테스트 전략](#7-테스트-전략)
8. [성능 측정](#8-성능-측정)

---

## 1. 현재 시스템 분석

### 1.1 현재 아키텍처

```
클라이언트 요청
    ↓
KeywordController (Servlet 스레드)
    ↓
KeywordNewsService (블로킹)
    ↓
├─ Redis 캐시 조회 (블로킹)
├─ 분산 락 획득 시도 (블로킹 + Thread.sleep)
└─ NaverNewsClient (RestTemplate - 블로킹)
       ↓
   네이버 API (최대 5초 대기)
```

### 1.2 현재 문제점

#### 🚨 스레드 블로킹

```java
// KeywordNewsService.java
private void waitForRetry(int attempt) {
    try {
        Thread.sleep(RETRY_DELAY_MS);  // 💥 스레드 블로킹!
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
}
```

**문제:**
- Tomcat 스레드 풀 고갈 위험
- 동시 요청 100개 → 100개 스레드 필요
- 각 스레드는 메모리 점유 (약 1MB)

#### 🚨 RestTemplate 동기 호출

```java
// NaverNewsClient.java
ResponseEntity<NaverSearchResponse> response = restTemplate.exchange(
    uri, HttpMethod.GET, entity, NaverSearchResponse.class);
// 네이버 API 응답 대기 중 스레드 차단됨 (최대 5초)
```

**문제:**
- API 응답 대기 중 스레드 idle
- Connection Pool 미설정 시 매번 새 연결
- 네트워크 I/O 블로킹

#### 🚨 Redis 동기 호출

```java
String jsonValue = redisTemplate.opsForValue().get(cacheKey);  // 블로킹
```

**문제:**
- Redis 응답 대기 중 스레드 차단
- 네트워크 레이턴시만큼 대기

### 1.3 성능 영향 분석

**시나리오: 동시 요청 100개**

| 항목 | 현재 (동기) | 목표 (비동기) |
|------|------------|--------------|
| 필요 스레드 수 | 100개 | 10~20개 |
| 메모리 사용 | ~100MB | ~20MB |
| 처리 시간 | 순차 처리 | 병렬 처리 |
| CPU 사용률 | 낮음 (대기 시간 多) | 높음 (효율적) |

---

## 2. 비동기 전환 목표

### 2.1 핵심 목표

1. **논블로킹 I/O**
   - 모든 외부 호출을 비동기로 전환
   - 스레드 블로킹 완전 제거

2. **리소스 효율성**
   - 적은 스레드로 많은 요청 처리
   - 메모리 사용량 감소

3. **응답성 향상**
   - 동시 요청 처리 능력 향상
   - 평균 응답 시간 감소

### 2.2 전환 범위

#### ✅ 전환 대상

- `NaverNewsClient`: RestTemplate → WebClient
- `KeywordNewsService`: 동기 → Reactive
- `RedisTemplate`: StringRedisTemplate → ReactiveRedisTemplate
- `RedisLockUtil`: 동기 락 → Reactive 락
- Controller 반환 타입: `ResponseEntity<T>` → `Mono<ResponseEntity<T>>`

#### ❌ 전환 제외

- JPA (Spring Data JPA는 블로킹, R2DBC는 별도 고려)
- OAuth2 인증 (기존 유지)

---

## 3. 아키텍처 변경 개요

### 3.1 Before (현재)

```
┌─────────────────────────────────────────┐
│  Servlet Container (Tomcat)            │
│  - Thread Pool: 200개 (기본)           │
│  - 각 요청 = 1 스레드                   │
└─────────────────────────────────────────┘
            ↓ 블로킹
┌─────────────────────────────────────────┐
│  RestTemplate (동기)                    │
│  - 응답 대기 중 스레드 차단              │
└─────────────────────────────────────────┘
            ↓
┌─────────────────────────────────────────┐
│  외부 API / Redis                       │
└─────────────────────────────────────────┘
```

### 3.2 After (비동기)

```
┌─────────────────────────────────────────┐
│  Netty (Reactive)                       │
│  - Event Loop: 8개 (CPU 코어 * 2)       │
│  - 논블로킹 I/O                         │
└─────────────────────────────────────────┘
            ↓ 논블로킹
┌─────────────────────────────────────────┐
│  WebClient (비동기)                     │
│  - Mono/Flux 반환                       │
│  - Reactor Netty 사용                   │
└─────────────────────────────────────────┘
            ↓
┌─────────────────────────────────────────┐
│  외부 API / Reactive Redis              │
└─────────────────────────────────────────┘
```

### 3.3 Reactive Streams 개념

#### Mono vs Flux

```java
// Mono<T>: 0~1개의 결과
Mono<String> mono = Mono.just("Hello");

// Flux<T>: 0~N개의 결과
Flux<String> flux = Flux.just("A", "B", "C");
```

#### 파이프라인 구성

```java
Mono<String> result = webClient.get()
    .uri("/api")
    .retrieve()
    .bodyToMono(String.class)
    .map(String::toUpperCase)      // 변환
    .filter(s -> s.length() > 5)   // 필터
    .doOnNext(s -> log.info(s))    // 사이드 이펙트
    .onErrorResume(e -> Mono.empty()); // 에러 처리
```

---

## 4. 단계별 전환 과정

### Phase 1: 의존성 추가 및 설정

#### 4.1.1 build.gradle 수정

```gradle
dependencies {
    // WebFlux 추가
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
    
    // Reactive Redis 추가
    implementation 'org.springframework.boot:spring-boot-starter-data-redis-reactive'
    
    // 기존 Web 제거 (선택적 - 완전 전환 시)
    // implementation 'org.springframework.boot:spring-boot-starter-web'
}
```

> **⚠️ 주의:** `spring-boot-starter-web`과 `spring-boot-starter-webflux`를 동시에 사용 가능하지만, 완전 전환 시 web 제거 권장

#### 4.1.2 application.yaml 설정

```yaml
spring:
  # Netty 서버 설정 (WebFlux 사용 시)
  webflux:
    base-path: /api
  
  # Reactive Redis 설정
  redis:
    host: ${REDIS_HOST:localhost}
    port: ${REDIS_PORT:6379}
    # Connection Pool 설정
    lettuce:
      pool:
        max-active: 10
        max-idle: 10
        min-idle: 2
```

---

### Phase 2: WebClient 설정 및 NaverNewsClient 전환

#### 4.2.1 WebClient 설정

**파일**: `src/main/java/com/jang/newsbara/global/config/WebClientConfig.java`

```java
package com.jang.newsbara.global.config;

import io.netty.channel.ChannelOption;
import io.netty.handler.timeout.ReadTimeoutHandler;
import io.netty.handler.timeout.WriteTimeoutHandler;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.reactive.ReactorClientHttpConnector;
import org.springframework.web.reactive.function.client.ExchangeFilterFunction;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;
import reactor.netty.http.client.HttpClient;
import reactor.netty.resources.ConnectionProvider;

import java.time.Duration;
import java.util.concurrent.TimeUnit;

@Slf4j
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient webClient() {
        // Connection Pool 설정
        ConnectionProvider provider = ConnectionProvider.builder("custom")
                .maxConnections(100)              // 최대 연결 수
                .maxIdleTime(Duration.ofSeconds(20))  // Idle 타임아웃
                .maxLifeTime(Duration.ofSeconds(60))  // 최대 수명
                .pendingAcquireTimeout(Duration.ofSeconds(60))  // 연결 획득 대기 시간
                .evictInBackground(Duration.ofSeconds(120))      // 백그라운드 정리 주기
                .build();

        // HttpClient 설정
        HttpClient httpClient = HttpClient.create(provider)
                .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000) // 연결 타임아웃: 5초
                .doOnConnected(conn -> conn
                        .addHandlerLast(new ReadTimeoutHandler(10, TimeUnit.SECONDS))   // 읽기 타임아웃: 10초
                        .addHandlerLast(new WriteTimeoutHandler(10, TimeUnit.SECONDS))  // 쓰기 타임아웃: 10초
                )
                .responseTimeout(Duration.ofSeconds(10));  // 전체 응답 타임아웃: 10초

        return WebClient.builder()
                .clientConnector(new ReactorClientHttpConnector(httpClient))
                .filter(logRequest())   // 요청 로깅
                .filter(logResponse())  // 응답 로깅
                .build();
    }

    // 요청 로깅 필터
    private ExchangeFilterFunction logRequest() {
        return ExchangeFilterFunction.ofRequestProcessor(request -> {
            log.debug("WebClient Request: {} {}", request.method(), request.url());
            return Mono.just(request);
        });
    }

    // 응답 로깅 필터
    private ExchangeFilterFunction logResponse() {
        return ExchangeFilterFunction.ofResponseProcessor(response -> {
            log.debug("WebClient Response: {}", response.statusCode());
            return Mono.just(response);
        });
    }
}
```

#### 4.2.2 NaverNewsClient 비동기 전환

**Before (동기):**

```java
@Component
public class NaverNewsClient {
    private final RestTemplate restTemplate;
    
    public NaverSearchResponse searchNews(String query, ...) {
        ResponseEntity<NaverSearchResponse> response = 
            restTemplate.exchange(uri, HttpMethod.GET, entity, NaverSearchResponse.class);
        return response.getBody();
    }
}
```

**After (비동기):**

```java
package com.jang.newsbara.external.naver;

import com.jang.newsbara.external.exception.ExternalErrorCode;
import com.jang.newsbara.external.naver.dto.NaverSearchResponse;
import com.jang.newsbara.global.config.properties.NaverApiProperties;
import com.jang.newsbara.global.exception.BusinessException;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.client.WebClient;
import org.springframework.web.reactive.function.client.WebClientResponseException;
import reactor.core.publisher.Mono;

@Slf4j
@Component
@RequiredArgsConstructor
public class NaverNewsClient {

    private final WebClient webClient;
    private final NaverApiProperties naverApiProperties;

    /**
     * 네이버 뉴스 검색 (비동기)
     * 
     * @return Mono<NaverSearchResponse> - 비동기 결과
     */
    public Mono<NaverSearchResponse> searchNews(String query, String sort, int start, int display) {
        log.info("네이버 뉴스 검색 API 호출 시작 - query: {}, sort: {}, start: {}, display: {}", 
                query, sort, start, display);

        return webClient.get()
                .uri(uriBuilder -> uriBuilder
                        .scheme("https")
                        .host("openapi.naver.com")
                        .path("/v1/search/news.json")
                        .queryParam("query", query)
                        .queryParam("sort", sort)
                        .queryParam("start", start)
                        .queryParam("display", display)
                        .build())
                .header("X-Naver-Client-Id", naverApiProperties.getClientId())
                .header("X-Naver-Client-Secret", naverApiProperties.getClientSecret())
                .retrieve()
                .bodyToMono(NaverSearchResponse.class)
                // null 응답 체크
                .switchIfEmpty(Mono.error(new BusinessException(ExternalErrorCode.API_NULL_RESPONSE)))
                // 에러 처리
                .onErrorMap(WebClientResponseException.class, e -> {
                    log.error("네이버 API HTTP 에러 - status: {}, body: {}", 
                            e.getStatusCode(), e.getResponseBodyAsString());
                    return new BusinessException(ExternalErrorCode.NAVER_API_ERROR);
                })
                .onErrorMap(throwable -> !(throwable instanceof BusinessException), e -> {
                    log.error("네이버 API 호출 실패 - error: {}", e.getMessage(), e);
                    return new BusinessException(ExternalErrorCode.NAVER_API_ERROR);
                })
                // 성공 로깅
                .doOnNext(response -> 
                    log.info("네이버 API 응답 성공 - query: {}, total: {}, items: {}",
                            query, response.getTotal(), 
                            response.getItems() != null ? response.getItems().size() : 0)
                );
    }
}
```

**주요 변경점:**
1. `RestTemplate` → `WebClient`
2. 반환 타입: `NaverSearchResponse` → `Mono<NaverSearchResponse>`
3. 에러 처리: try-catch → `onErrorMap()`
4. 로깅: 동기 → `doOnNext()`, `doOnError()`

---

### Phase 3: Reactive Redis 설정 및 RedisLockUtil 전환

#### 4.3.1 Reactive Redis 설정

**파일**: `src/main/java/com/jang/newsbara/global/config/ReactiveRedisConfig.java`

```java
package com.jang.newsbara.global.config;

import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.ReactiveRedisConnectionFactory;
import org.springframework.data.redis.core.ReactiveRedisTemplate;
import org.springframework.data.redis.core.ReactiveStringRedisTemplate;
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
@RequiredArgsConstructor
public class ReactiveRedisConfig {

    private final ObjectMapper objectMapper;

    @Bean
    public ReactiveStringRedisTemplate reactiveStringRedisTemplate(
            ReactiveRedisConnectionFactory connectionFactory) {
        return new ReactiveStringRedisTemplate(connectionFactory);
    }

    @Bean
    public ReactiveRedisTemplate<String, Object> reactiveRedisTemplate(
            ReactiveRedisConnectionFactory connectionFactory) {
        
        Jackson2JsonRedisSerializer<Object> serializer = 
                new Jackson2JsonRedisSerializer<>(objectMapper, Object.class);

        RedisSerializationContext<String, Object> context = 
                RedisSerializationContext.<String, Object>newSerializationContext(
                        StringRedisSerializer.UTF_8)
                .value(serializer)
                .hashValue(serializer)
                .build();

        return new ReactiveRedisTemplate<>(connectionFactory, context);
    }
}
```

#### 4.3.2 RedisLockUtil 비동기 전환

**Before (동기):**

```java
public Optional<String> acquireLock(String lockKey, Duration ttl) {
    String lockValue = UUID.randomUUID().toString();
    Boolean acquired = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, lockValue, ttl);
    return acquired ? Optional.of(lockValue) : Optional.empty();
}
```

**After (비동기):**

```java
package com.jang.newsbara.global.util;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.ReactiveStringRedisTemplate;
import org.springframework.data.redis.core.script.RedisScript;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Mono;

import java.time.Duration;
import java.util.Collections;
import java.util.UUID;

@Slf4j
@Component
@RequiredArgsConstructor
public class ReactiveRedisLockUtil {

    private final ReactiveStringRedisTemplate reactiveRedisTemplate;

    private static final String RELEASE_LOCK_SCRIPT =
            "if redis.call('GET', KEYS[1]) == ARGV[1] then " +
            "    return redis.call('DEL', KEYS[1]) " +
            "else " +
            "    return 0 " +
            "end";

    /**
     * 분산 락 획득 (비동기)
     * 
     * @return Mono<String> - 락 획득 성공 시 lockValue, 실패 시 Mono.empty()
     */
    public Mono<String> acquireLock(String lockKey, Duration ttl) {
        String lockValue = UUID.randomUUID().toString();
        
        return reactiveRedisTemplate.opsForValue()
                .setIfAbsent(lockKey, lockValue, ttl)
                .flatMap(acquired -> {
                    if (Boolean.TRUE.equals(acquired)) {
                        log.debug("락 획득 성공 - key: {}, value: {}", lockKey, lockValue);
                        return Mono.just(lockValue);
                    } else {
                        log.debug("락 획득 실패 - key: {}", lockKey);
                        return Mono.empty();
                    }
                });
    }

    /**
     * 분산 락 해제 (비동기)
     * 
     * @return Mono<Boolean> - 락 해제 성공 여부
     */
    public Mono<Boolean> releaseLock(String lockKey, String lockValue) {
        RedisScript<Long> script = RedisScript.of(RELEASE_LOCK_SCRIPT, Long.class);
        
        return reactiveRedisTemplate.execute(script, 
                        Collections.singletonList(lockKey), 
                        Collections.singletonList(lockValue))
                .next()
                .map(result -> {
                    boolean released = Long.valueOf(1).equals(result);
                    if (released) {
                        log.debug("락 해제 성공 - key: {}", lockKey);
                    } else {
                        log.warn("락 해제 실패 - key: {}, 이미 만료되었거나 다른 스레드가 소유", lockKey);
                    }
                    return released;
                })
                .onErrorResume(e -> {
                    log.error("락 해제 중 에러 발생 - key: {}, error: {}", lockKey, e.getMessage(), e);
                    return Mono.just(false);
                });
    }

    /**
     * 재시도와 함께 락 획득 (비동기)
     * 
     * @param maxRetries 최대 재시도 횟수
     * @param retryDelay 재시도 간격
     * @return Mono<String> - 락 값 또는 에러
     */
    public Mono<String> acquireLockWithRetry(String lockKey, Duration ttl, 
                                              int maxRetries, Duration retryDelay) {
        return acquireLock(lockKey, ttl)
                .repeatWhenEmpty(maxRetries, companion -> 
                    companion.delayElements(retryDelay))  // Thread.sleep() 대신 delayElements()
                .switchIfEmpty(Mono.error(
                    new RuntimeException("락 획득 실패 - 최대 재시도 초과: " + lockKey)));
    }
}
```

**핵심 변경:**
- `Thread.sleep()` → `delayElements()` (논블로킹 딜레이)
- `Optional<String>` → `Mono<String>`
- 재시도 로직: 명시적 루프 → `repeatWhenEmpty()`

---

### Phase 4: KeywordNewsService 비동기 전환

**Before (동기):**

```java
public PageResponse<KeywordNewsResponse> searchKeywordNews(...) {
    SearchResult result = getFromCacheOrFetch(...);
    return createPageResponse(...);
}
```

**After (비동기):**

```java
package com.jang.newsbara.keyword.service;

import reactor.core.publisher.Mono;
import reactor.util.retry.Retry;
import java.time.Duration;

@Slf4j
@Service
@RequiredArgsConstructor
public class KeywordNewsService {

    private final ReactiveStringRedisTemplate reactiveRedisTemplate;
    private final ReactiveRedisLockUtil reactiveRedisLockUtil;
    private final NaverNewsClient naverNewsClient;
    private final ObjectMapper objectMapper;
    private final CacheProperties cacheProperties;

    /**
     * 키워드 뉴스 검색 (비동기)
     */
    public Mono<PageResponse<KeywordNewsResponse>> searchKeywordNews(
            Long userId, Long keywordId, String sort, Pageable pageable) {
        
        return findUserKeyword(userId, keywordId)
                .flatMap(userKeyword -> {
                    String keyword = userKeyword.getKeyword();
                    int start = calculateStart(pageable);
                    int display = calculateDisplay(pageable);

                    log.info("키워드 뉴스 검색 - userId: {}, keyword: '{}', sort: {}, start: {}, display: {}", 
                            userId, keyword, sort, start, display);

                    String cacheKey = CacheKeyUtil.generateNewsListCacheKey(keyword, start);
                    
                    return getFromCacheOrFetch(keyword, sort, start, display, cacheKey)
                            .map(result -> createPageResponse(result.items(), pageable, result.total()));
                });
    }

    private Mono<UserKeyword> findUserKeyword(Long userId, Long keywordId) {
        // JPA는 여전히 블로킹이므로 별도 스레드에서 실행
        return Mono.fromCallable(() -> 
                userKeywordRepository.findByIdAndUserId(keywordId, userId)
                        .orElseThrow(() -> new BusinessException(KeywordErrorCode.KEYWORD_NOT_FOUND))
        ).subscribeOn(Schedulers.boundedElastic());  // 블로킹 작업용 스레드 풀
    }

    private Mono<SearchResult> getFromCacheOrFetch(String keyword, String sort, 
                                                     int start, int display, String cacheKey) {
        return getFromCache(cacheKey)
                .switchIfEmpty(
                    fetchWithLock(keyword, sort, start, display, cacheKey)
                );
    }

    private Mono<SearchResult> getFromCache(String cacheKey) {
        return reactiveRedisTemplate.opsForValue()
                .get(cacheKey)
                .flatMap(jsonValue -> {
                    try {
                        SearchResult result = objectMapper.readValue(
                                jsonValue, new TypeReference<SearchResult>() {});
                        log.info("캐시 히트 - key: {}", cacheKey);
                        return Mono.just(result);
                    } catch (Exception e) {
                        log.warn("캐시 역직렬화 실패 - key: {}", cacheKey);
                        return Mono.empty();
                    }
                });
    }

    private Mono<SearchResult> fetchWithLock(String keyword, String sort, 
                                              int start, int display, String cacheKey) {
        String lockKey = CacheKeyUtil.generateNewsListLockKey(keyword, start);
        Duration lockTtl = cacheProperties.getLock().getNewsTtl();

        // 락 획득 재시도 (논블로킹)
        return reactiveRedisLockUtil.acquireLockWithRetry(lockKey, lockTtl, 3, Duration.ofMillis(100))
                .flatMap(lockValue -> 
                    fetchAndCache(keyword, sort, start, display, cacheKey, lockKey, lockValue)
                        .doFinally(signalType -> 
                            // 락 해제 (성공/실패/취소 무관하게 실행)
                            reactiveRedisLockUtil.releaseLock(lockKey, lockValue)
                                    .subscribe()  // Fire-and-forget
                        )
                )
                // 락 획득 실패 시 API 직접 호출
                .onErrorResume(e -> {
                    log.warn("락 획득 실패 - API 직접 호출: {}", e.getMessage());
                    return fetchFromApi(keyword, sort, start, display);
                });
    }

    private Mono<SearchResult> fetchAndCache(String keyword, String sort, int start, int display,
                                              String cacheKey, String lockKey, String lockValue) {
        log.info("락 획득 성공 - key: {}", lockKey);

        // 더블 체크: 다른 스레드가 이미 캐시했을 수 있음
        return getFromCache(cacheKey)
                .switchIfEmpty(
                    fetchFromApi(keyword, sort, start, display)
                            .flatMap(result -> 
                                saveToCache(cacheKey, result)
                                        .thenReturn(result)
                            )
                );
    }

    private Mono<SearchResult> fetchFromApi(String keyword, String sort, int start, int display) {
        return naverNewsClient.searchNews(keyword, sort, start, display)
                .map(response -> {
                    List<KeywordNewsResponse> items = parseResponse(response);
                    return new SearchResult(items, response.getTotal());
                });
    }

    private Mono<Boolean> saveToCache(String cacheKey, SearchResult result) {
        try {
            String jsonValue = objectMapper.writeValueAsString(result);
            Duration cacheTtl = cacheProperties.getKeywordNews().getTtl();
            
            return reactiveRedisTemplate.opsForValue()
                    .set(cacheKey, jsonValue, cacheTtl)
                    .doOnNext(saved -> {
                        if (Boolean.TRUE.equals(saved)) {
                            log.info("캐시 저장 완료 - key: {}, items: {}", cacheKey, result.items().size());
                        }
                    })
                    .onErrorResume(e -> {
                        log.warn("캐시 저장 실패 - key: {}, error: {}", cacheKey, e.getMessage());
                        return Mono.just(false);
                    });
        } catch (Exception e) {
            log.warn("캐시 직렬화 실패 - key: {}", cacheKey);
            return Mono.just(false);
        }
    }

    private List<KeywordNewsResponse> parseResponse(NaverSearchResponse response) {
        return response.getItems().stream()
                .map(KeywordNewsResponse::from)
                .toList();
    }

    private PageResponse<KeywordNewsResponse> createPageResponse(
            List<KeywordNewsResponse> items, Pageable pageable, long total) {
        Page<KeywordNewsResponse> page = new PageImpl<>(items, pageable, total);
        return PageResponse.of(page);
    }

    private record SearchResult(List<KeywordNewsResponse> items, long total) {}
}
```

**핵심 패턴:**

1. **체이닝**: `flatMap()`, `map()`, `switchIfEmpty()`
2. **에러 처리**: `onErrorResume()`
3. **사이드 이펙트**: `doOnNext()`, `doFinally()`
4. **블로킹 작업 격리**: `subscribeOn(Schedulers.boundedElastic())`

---

### Phase 5: Controller 비동기 전환

**Before (동기):**

```java
@GetMapping("/{keywordId}/news")
public ResponseEntity<ApiResponse<PageResponse<KeywordNewsResponse>>> searchKeywordNews(...) {
    PageResponse<KeywordNewsResponse> news = keywordNewsService.searchKeywordNews(...);
    return ApiResponse.success(news);
}
```

**After (비동기):**

```java
@GetMapping("/{keywordId}/news")
public Mono<ResponseEntity<ApiResponse<PageResponse<KeywordNewsResponse>>>> searchKeywordNews(
        @PathVariable Long keywordId,
        @RequestParam(defaultValue = "sim") String sort,
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "15") int size,
        @AuthenticationPrincipal CustomUserDetails userDetails) {

    log.info("키워드 뉴스 검색 요청 - userId: {}, keywordId: {}, sort: {}, page: {}, size: {}", 
            userDetails.getUserId(), keywordId, sort, page, size);

    Pageable pageable = PageRequest.of(page, size);
    
    return keywordNewsService.searchKeywordNews(userDetails.getUserId(), keywordId, sort, pageable)
            .map(ApiResponse::success)
            .map(ResponseEntity::ok);
}
```

**또는 Spring WebFlux의 자동 언래핑 사용:**

```java
@GetMapping("/{keywordId}/news")
public Mono<ApiResponse<PageResponse<KeywordNewsResponse>>> searchKeywordNews(...) {
    Pageable pageable = PageRequest.of(page, size);
    return keywordNewsService.searchKeywordNews(userDetails.getUserId(), keywordId, sort, pageable)
            .map(ApiResponse::success);
}
```

---

## 5. 코드 예시

### 5.1 Reactive 패턴 예시

#### 순차 실행 (flatMap)

```java
// Bad: 블로킹
String user = userService.getUser(id);        // 1초 대기
String profile = profileService.get(user.id);  // 1초 대기
// 총 2초

// Good: 비동기
Mono<String> result = userService.getUser(id)     // 비동기 시작
        .flatMap(user -> profileService.get(user.id))  // 첫 번째 완료 후 두 번째 시작
        .map(Profile::getName);
// 총 2초 (동일하지만 스레드 블로킹 없음)
```

#### 병렬 실행 (zip)

```java
// 순차 실행: 3초
Mono<A> a = serviceA.call();  // 1초
Mono<B> b = serviceB.call();  // 1초  
Mono<C> c = serviceC.call();  // 1초

// 병렬 실행: 1초
Mono<Tuple3<A, B, C>> result = Mono.zip(
    serviceA.call(),
    serviceB.call(),
    serviceC.call()
);
```

#### 조건부 실행 (switchIfEmpty)

```java
getFromCache(key)
    .switchIfEmpty(fetchFromApi())  // 캐시 없으면 API 호출
    .switchIfEmpty(Mono.just(defaultValue));  // API도 실패하면 기본값
```

### 5.2 에러 처리 패턴

```java
webClient.get()
    .retrieve()
    .bodyToMono(String.class)
    // 특정 예외만 처리
    .onErrorResume(TimeoutException.class, e -> 
        Mono.just("Timeout occurred"))
    // 모든 예외 처리
    .onErrorResume(e -> {
        log.error("Error: {}", e.getMessage());
        return Mono.empty();
    })
    // 에러를 다른 타입으로 변환
    .onErrorMap(IOException.class, e -> 
        new BusinessException(ErrorCode.IO_ERROR))
    // 재시도
    .retryWhen(Retry.backoff(3, Duration.ofSeconds(1)))
    // 기본값 반환
    .defaultIfEmpty("default");
```

### 5.3 블로킹 코드 통합

```java
// JPA 같은 블로킹 코드
Mono<User> user = Mono.fromCallable(() -> 
        userRepository.findById(id)
                .orElseThrow(() -> new NotFoundException())
).subscribeOn(Schedulers.boundedElastic());  // 별도 스레드 풀
```

---

## 6. 주의사항 및 트레이드오프

### 6.1 주의사항

#### ⚠️ 절대 블로킹하지 마세요

```java
// ❌ 절대 금지!
Mono<String> result = someReactiveMethod();
String value = result.block();  // 💥 스레드 블로킹!

// ✅ 올바른 방법
Mono<String> result = someReactiveMethod()
        .flatMap(value -> doSomething(value));
```

#### ⚠️ 구독 필수

```java
// ❌ 실행 안 됨!
Mono<String> result = webClient.get()...;  // 아무 일도 안 일어남

// ✅ 구독해야 실행됨
Mono<String> result = webClient.get()...;
result.subscribe();  // 이제 실행됨

// 또는
return result;  // Spring이 자동으로 구독
```

#### ⚠️ ThreadLocal 사용 불가

```java
// ❌ ThreadLocal은 Reactive에서 작동 안 함
ThreadLocal<User> currentUser = new ThreadLocal<>();

// ✅ Context 사용
Mono.deferContextual(ctx -> {
    User user = ctx.get("user");
    return doSomething(user);
}).contextWrite(Context.of("user", currentUser));
```

### 6.2 트레이드오프

| 항목 | 동기 (Before) | 비동기 (After) |
|------|--------------|---------------|
| **코드 복잡도** | 낮음 ⭐ | 높음 ⭐⭐⭐ |
| **디버깅** | 쉬움 ⭐ | 어려움 ⭐⭐⭐ |
| **학습 곡선** | 낮음 | 높음 |
| **스레드 사용** | 많음 | 적음 ⭐⭐⭐ |
| **메모리 사용** | 많음 | 적음 ⭐⭐⭐ |
| **처리량** | 낮음 | 높음 ⭐⭐⭐ |
| **응답 속도** | 느림 (부하 시) | 빠름 ⭐⭐⭐ |

---

## 7. 테스트 전략

### 7.1 단위 테스트

```java
@Test
void testSearchNews_Reactive() {
    // Given
    NaverSearchResponse mockResponse = createMockResponse();
    when(naverNewsClient.searchNews(...))
            .thenReturn(Mono.just(mockResponse));

    // When
    Mono<PageResponse<KeywordNewsResponse>> result = 
            keywordNewsService.searchKeywordNews(...);

    // Then
    StepVerifier.create(result)
            .assertNext(page -> {
                assertThat(page.getContent()).hasSize(10);
                assertThat(page.getTotalElements()).isEqualTo(100);
            })
            .verifyComplete();
}
```

### 7.2 통합 테스트

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class KeywordControllerIntegrationTest {

    @Autowired
    private WebTestClient webTestClient;

    @Test
    void testSearchKeywordNews() {
        webTestClient.get()
                .uri("/api/keywords/1/news?sort=sim&page=0&size=15")
                .header("Authorization", "Bearer " + token)
                .exchange()
                .expectStatus().isOk()
                .expectBody()
                .jsonPath("$.data.content").isArray()
                .jsonPath("$.data.totalElements").isNumber();
    }
}
```

### 7.3 성능 테스트

```bash
# Apache Bench
ab -n 1000 -c 100 http://localhost:8080/api/keywords/1/news

# 측정 항목:
# - Requests per second (RPS)
# - Time per request
# - Memory usage
# - Thread count
```

---

## 8. 성능 측정

### 8.1 측정 지표

```yaml
# application.yaml
management:
  metrics:
    export:
      prometheus:
        enabled: true
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
```

**주요 메트릭:**
- `http.server.requests` - 요청 처리 시간
- `jvm.threads.live` - 활성 스레드 수
- `jvm.memory.used` - 메모리 사용량
- `redis.commands.count` - Redis 명령 횟수

### 8.2 부하 테스트 시나리오

```
시나리오 1: 동시 요청 100개
- Before: ~5초 (스레드 풀 고갈)
- After: ~1초 (논블로킹)

시나리오 2: 동시 요청 1000개
- Before: ~30초 또는 타임아웃
- After: ~3초

메모리 사용:
- Before: ~200MB (스레드 100개 * 1MB)
- After: ~50MB (스레드 10개)
```

---

## 9. 마이그레이션 체크리스트

### Phase 1: 준비
- [ ] WebFlux 의존성 추가
- [ ] Reactive Redis 의존성 추가
- [ ] WebClient 설정
- [ ] ReactiveRedisConfig 설정

### Phase 2: 외부 API 클라이언트
- [ ] NaverNewsClient 비동기 전환
- [ ] 에러 처리 검증
- [ ] 단위 테스트 작성

### Phase 3: 분산 락
- [ ] ReactiveRedisLockUtil 작성
- [ ] Lua 스크립트 검증
- [ ] 재시도 로직 테스트

### Phase 4: 서비스 레이어
- [ ] KeywordNewsService 비동기 전환
- [ ] 캐시 로직 검증
- [ ] JPA 블로킹 코드 격리

### Phase 5: 컨트롤러
- [ ] Controller 반환 타입 변경
- [ ] 통합 테스트 작성

### Phase 6: 검증
- [ ] 단위 테스트 통과
- [ ] 통합 테스트 통과
- [ ] 성능 테스트 (부하)
- [ ] 메모리 프로파일링

---

## 10. 참고 자료

### 공식 문서
- [Spring WebFlux 공식 문서](https://docs.spring.io/spring-framework/reference/web/webflux.html)
- [Project Reactor 문서](https://projectreactor.io/docs/core/release/reference/)
- [Reactive Redis 문서](https://docs.spring.io/spring-data/redis/reference/redis/reactive.html)

### 추천 학습 자료
- "Spring in Action" 6판 - WebFlux 챕터
- Baeldung Spring WebFlux 튜토리얼
- Reactor 3 Reference Guide

---

## 부록: 자주 하는 실수

### 1. flatMap vs map 혼동

```java
// ❌ 틀림
Mono<Mono<String>> wrong = mono.map(value -> Mono.just(value.toUpperCase()));

// ✅ 맞음
Mono<String> correct = mono.flatMap(value -> Mono.just(value.toUpperCase()));
```

### 2. block() 남용

```java
// ❌ 틀림
public String getUser(Long id) {
    return userService.getUser(id).block();  // 블로킹!
}

// ✅ 맞음
public Mono<String> getUser(Long id) {
    return userService.getUser(id);
}
```

### 3. 구독 누락

```java
// ❌ 틀림 - 실행 안 됨
Mono<Void> saveOperation = repository.save(entity);

// ✅ 맞음
Mono<Void> saveOperation = repository.save(entity);
saveOperation.subscribe();  // 또는 return하여 Spring이 구독하게 함
```

---

**이 문서를 참고하여 단계별로 진행하세요. 각 Phase마다 테스트하고 검증한 후 다음 단계로 진행하는 것을 권장합니다.** 🚀

## 11. Reactive 심화 개념

### 11.1 Backpressure (배압)

**문제 상황:**
생산자(Publisher)가 소비자(Subscriber)보다 빠르게 데이터를 생성하는 경우

```java
// 생산자가 초당 1000개 생성
Flux.interval(Duration.ofMillis(1))
    .subscribe(data -> {
        Thread.sleep(100);  // 소비자는 100ms 걸림
        process(data);
    });
// → OutOfMemoryError 가능!
```

**해결 방법:**

```java
// 1. buffer() - 버퍼링
Flux.interval(Duration.ofMillis(1))
    .buffer(100)  // 100개씩 모아서 처리
    .subscribe(batch -> processBatch(batch));

// 2. sample() - 샘플링
Flux.interval(Duration.ofMillis(1))
    .sample(Duration.ofMillis(100))  // 100ms마다 최신 값만 가져옴
    .subscribe(data -> process(data));

// 3. onBackpressureBuffer() - 백프레셔 버퍼
Flux.range(1, 1000)
    .onBackpressureBuffer(100)  // 최대 100개 버퍼
    .subscribe(data -> process(data));

// 4. onBackpressureDrop() - 드롭
Flux.range(1, 1000)
    .onBackpressureDrop(dropped -> log.warn("Dropped: {}", dropped))
    .subscribe(data -> process(data));
```

**우리 시스템에 적용:**

```java
// 대량의 뉴스 항목 처리 시
public Flux<KeywordNewsResponse> searchAllNews(List<String> keywords) {
    return Flux.fromIterable(keywords)
            .flatMap(keyword -> naverNewsClient.searchNews(keyword, ...))
            .onBackpressureBuffer(50)  // 최대 50개까지 버퍼
            .flatMap(response -> Flux.fromIterable(response.getItems()))
            .map(KeywordNewsResponse::from);
}
```

---

### 11.2 Scheduler (스케줄러)

**종류별 특성:**

| Scheduler | 용도 | 스레드 풀 크기 | 사용 예 |
|-----------|------|---------------|---------|
| `Schedulers.immediate()` | 현재 스레드에서 즉시 실행 | - | 테스트 |
| `Schedulers.single()` | 단일 재사용 스레드 | 1 | 간단한 작업 |
| `Schedulers.parallel()` | 병렬 처리 | CPU 코어 수 | CPU 집약적 작업 |
| `Schedulers.boundedElastic()` | I/O 블로킹 작업 | 10 * CPU 코어 | JPA, 파일 I/O |
| `Schedulers.fromExecutor()` | 커스텀 Executor | 사용자 정의 | 특수 목적 |

**사용 예시:**

```java
// CPU 집약적 작업
Mono.fromCallable(() -> heavyComputation())
    .subscribeOn(Schedulers.parallel());

// I/O 블로킹 작업 (JPA)
Mono.fromCallable(() -> userRepository.findById(id))
    .subscribeOn(Schedulers.boundedElastic());

// 작업 스케줄러 변경
Mono.just("data")
    .subscribeOn(Schedulers.boundedElastic())  // 여기서 실행
    .map(String::toUpperCase)
    .publishOn(Schedulers.parallel())  // 여기부터는 parallel에서 실행
    .map(String::toLowerCase);
```

**주의사항:**

```java
// ❌ 잘못된 사용
Mono.just("data")
    .map(data -> {
        Thread.sleep(1000);  // 블로킹!
        return data.toUpperCase();
    });

// ✅ 올바른 사용
Mono.just("data")
    .flatMap(data -> Mono.delay(Duration.ofSeconds(1))
        .thenReturn(data.toUpperCase()));

// 또는 블로킹 작업은 격리
Mono.fromCallable(() -> {
        Thread.sleep(1000);
        return data.toUpperCase();
    })
    .subscribeOn(Schedulers.boundedElastic());
```

---

### 11.3 Hot vs Cold Publishers

#### Cold Publisher (기본)

```java
// 구독할 때마다 새로 시작
Mono<String> cold = Mono.fromCallable(() -> {
    System.out.println("API 호출!");
    return callApi();
});

cold.subscribe(d -> System.out.println("구독자 1: " + d));
// 출력: API 호출!
//      구독자 1: result

cold.subscribe(d -> System.out.println("구독자 2: " + d));
// 출력: API 호출!  ← 다시 실행!
//      구독자 2: result
```

#### Hot Publisher

```java
// 구독 여부와 관계없이 데이터 생성
Flux<Long> hot = Flux.interval(Duration.ofSeconds(1))
    .share();  // Hot으로 전환

hot.subscribe(d -> System.out.println("구독자 1: " + d));
Thread.sleep(2000);
hot.subscribe(d -> System.out.println("구독자 2: " + d));

// 출력:
// 구독자 1: 0
// 구독자 1: 1
// 구독자 1: 2
// 구독자 2: 2  ← 중간부터 시작
```

**실무 활용:**

```java
// SSE (Server-Sent Events)를 위한 Hot Publisher
Flux<NewsUpdate> newsUpdateStream = Flux.interval(Duration.ofSeconds(10))
    .flatMap(tick -> fetchLatestNews())
    .share();  // 모든 클라이언트가 같은 스트림 공유

@GetMapping(value = "/news/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<NewsUpdate> streamNews() {
    return newsUpdateStream;
}
```

---

### 11.4 Context (컨텍스트)

ThreadLocal 대체:

```java
// ❌ ThreadLocal은 Reactive에서 작동 안 함
ThreadLocal<User> currentUser = new ThreadLocal<>();

// ✅ Context 사용
Mono<String> result = Mono.deferContextual(ctx -> {
        User user = ctx.get("user");
        return processWithUser(user);
    })
    .contextWrite(Context.of("user", currentUser));

// 실무 예시: 인증 정보 전파
public Mono<PageResponse<KeywordNewsResponse>> searchKeywordNews(...) {
    return Mono.deferContextual(ctx -> {
            Long userId = ctx.get("userId");
            return doSearch(userId, ...);
        })
        .contextWrite(Context.of("userId", userDetails.getUserId()));
}
```

---

## 12. 트러블슈팅 가이드

### 12.1 자주 발생하는 에러

#### 에러 1: `IllegalStateException: block()/blockFirst()/blockLast() are blocking`

```
java.lang.IllegalStateException: block()/blockFirst()/blockLast() 
are blocking, which is not supported in thread reactor-http-nio-2
```

**원인:** Reactor 스레드에서 block() 호출

**해결:**

```java
// ❌ 틀림
@GetMapping("/news")
public String getNews() {
    return webClient.get()
        .retrieve()
        .bodyToMono(String.class)
        .block();  // 💥
}

// ✅ 맞음
@GetMapping("/news")
public Mono<String> getNews() {
    return webClient.get()
        .retrieve()
        .bodyToMono(String.class);
}
```

---

#### 에러 2: Subscription not happening

```java
// ❌ 실행 안 됨
public void saveData() {
    Mono<Void> save = repository.save(entity);
    // 구독 안 함!
}

// ✅ 맞음
public Mono<Void> saveData() {
    return repository.save(entity);  // Controller가 구독
}
```

---

#### 에러 3: Memory Leak (메모리 누수)

```java
// ❌ Flux가 무한히 생성됨
Flux.interval(Duration.ofSeconds(1))
    .subscribe(tick -> System.out.println(tick));
// 구독 해제 안 함!

// ✅ 올바른 사용
Disposable subscription = Flux.interval(Duration.ofSeconds(1))
    .subscribe(tick -> System.out.println(tick));

// 나중에 정리
subscription.dispose();

// 또는 take()로 제한
Flux.interval(Duration.ofSeconds(1))
    .take(10)  // 10개만
    .subscribe(tick -> System.out.println(tick));
```

---

### 12.2 디버깅 팁

#### 1. log() 연산자 사용

```java
Mono.just("data")
    .log()  // 모든 이벤트 로깅
    .map(String::toUpperCase)
    .log("AFTER_MAP")  // 특정 단계 로깅
    .subscribe();

// 출력:
// INFO: onSubscribe(MonoJust.MonoJustSubscription)
// INFO: request(unbounded)
// INFO: onNext(data)
// INFO AFTER_MAP: onNext(DATA)
// INFO: onComplete()
```

#### 2. checkpoint() 사용

```java
Mono.error(new RuntimeException("Error!"))
    .checkpoint("체크포인트 A")
    .map(String::toUpperCase)
    .checkpoint("체크포인트 B")
    .subscribe();

// 스택 트레이스에 체크포인트 정보 포함
```

#### 3. Hooks 사용

```java
// 전역 에러 핸들러
Hooks.onErrorDropped(e -> 
    log.error("에러 드롭됨: {}", e.getMessage()));

// 연산자 디버그 모드
Hooks.onOperatorDebug();
```

---

## 13. 성능 최적화 팁

### 13.1 불필요한 체이닝 줄이기

```java
// ❌ 비효율적
Mono.just(data)
    .map(d -> d)
    .flatMap(d -> Mono.just(d))
    .map(d -> d.toUpperCase());

// ✅ 효율적
Mono.just(data)
    .map(String::toUpperCase);
```

### 13.2 캐시 활용

```java
// API 결과 캐싱
Mono<String> cached = webClient.get()
    .retrieve()
    .bodyToMono(String.class)
    .cache(Duration.ofMinutes(5));  // 5분간 캐시

// 여러 번 호출해도 1번만 실행
cached.subscribe();
cached.subscribe();  // 캐시된 값 사용
```

### 13.3 병렬 처리

```java
// 순차 처리 (느림)
Flux.fromIterable(urls)
    .flatMap(url -> webClient.get().uri(url).retrieve().bodyToMono(String.class))
    .collectList();

// 병렬 처리 (빠름)
Flux.fromIterable(urls)
    .parallel(4)  // 4개 병렬
    .runOn(Schedulers.parallel())
    .flatMap(url -> webClient.get().uri(url).retrieve().bodyToMono(String.class))
    .sequential()
    .collectList();
```

---

## 14. 실전 패턴 모음

### 14.1 재시도 전략

```java
// 지수 백오프 재시도
webClient.get()
    .retrieve()
    .bodyToMono(String.class)
    .retryWhen(Retry.backoff(3, Duration.ofSeconds(1))
        .maxBackoff(Duration.ofSeconds(10))
        .filter(throwable -> throwable instanceof TimeoutException));

// 조건부 재시도
webClient.get()
    .retrieve()
    .bodyToMono(String.class)
    .retryWhen(Retry.max(3)
        .filter(e -> e instanceof WebClientResponseException.ServiceUnavailable));
```

### 14.2 타임아웃 패턴

```java
// 전체 타임아웃
webClient.get()
    .retrieve()
    .bodyToMono(String.class)
    .timeout(Duration.ofSeconds(5));

// 폴백과 함께
webClient.get()
    .retrieve()
    .bodyToMono(String.class)
    .timeout(Duration.ofSeconds(5))
    .onErrorReturn("기본값");
```

### 14.3 Circuit Breaker 패턴

```java
// Resilience4j와 함께
CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("naverApi");

Mono<String> result = Mono.fromCallable(() -> 
        circuitBreaker.executeSupplier(() -> 
            callNaverApi()
        )
    )
    .subscribeOn(Schedulers.boundedElastic());
```

### 14.4 캐시 어사이드 패턴

```java
public Mono<Data> getData(String key) {
    return reactiveRedisTemplate.opsForValue()
        .get(key)
        // 캐시 미스
        .switchIfEmpty(
            fetchFromDatabase(key)
                .flatMap(data -> 
                    // DB에서 가져온 데이터를 캐시에 저장
                    reactiveRedisTemplate.opsForValue()
                        .set(key, data, Duration.ofMinutes(10))
                        .thenReturn(data)
                )
        );
}
```

---

## 15. 마이그레이션 로드맵 (실전)

### Week 1: 준비 및 학습
- [ ] WebFlux, Reactor 문서 학습
- [ ] 팀 교육 세션
- [ ] 의존성 추가
- [ ] WebClient, ReactiveRedis 설정

### Week 2: 외부 API 클라이언트 전환
- [ ] NaverNewsClient 전환
- [ ] 단위 테스트 작성
- [ ] 통합 테스트

### Week 3: 락 및 캐시 레이어
- [ ] ReactiveRedisLockUtil 구현
- [ ] 캐시 로직 전환
- [ ] 락 동작 검증

### Week 4: 서비스 레이어 전환
- [ ] KeywordNewsService 전환
- [ ] JPA 블로킹 코드 격리
- [ ] 재시도 로직 검증

### Week 5: 컨트롤러 및 통합
- [ ] Controller 전환
- [ ] E2E 테스트
- [ ] 성능 테스트

### Week 6: 운영 준비
- [ ] 모니터링 설정
- [ ] 알람 설정
- [ ] 롤백 계획 수립
- [ ] 스테이징 배포

### Week 7: 프로덕션 배포
- [ ] 카나리 배포
- [ ] 성능 모니터링
- [ ] 이슈 대응
- [ ] 점진적 트래픽 전환

---

## 16. FAQ

### Q1: WebFlux로 전환하면 무조건 빠른가요?
**A:** 아니요. I/O 대기가 많은 경우에만 효과적입니다. CPU 집약적 작업은 오히려 느릴 수 있습니다.

### Q2: JPA를 R2DBC로 전환해야 하나요?
**A:** 필수는 아닙니다. `subscribeOn(Schedulers.boundedElastic())`로 JPA를 별도 스레드에서 실행 가능합니다. 하지만 진정한 논블로킹을 원한다면 R2DBC 고려.

### Q3: 기존 코드를 한 번에 전환해야 하나요?
**A:** 아니요. Spring Boot는 Web과 WebFlux 공존을 지원합니다. 점진적 전환 가능.

### Q4: 디버깅이 너무 어려운데요?
**A:** `log()`, `checkpoint()`, Reactor DevTools 사용을 권장합니다. 초기에는 학습 곡선이 있지만 익숙해지면 괜찮습니다.

### Q5: 언제 비동기 전환을 하지 말아야 하나요?
**A:** 
- 트래픽이 적은 서비스 (동시 요청 < 50)
- 팀의 Reactive 경험이 전혀 없는 경우
- CRUD 중심의 단순한 서비스

---

## 17. 체크리스트 (최종)

### 전환 전 확인사항
- [ ] 현재 시스템의 병목 지점 파악
- [ ] 비동기 전환의 명확한 목표 설정
- [ ] 팀의 Reactive 학습 완료
- [ ] 충분한 테스트 환경 준비

### 전환 중 확인사항
- [ ] 각 Phase마다 테스트
- [ ] 성능 벤치마크 측정
- [ ] 메모리 프로파일링
- [ ] 에러 처리 검증

### 전환 후 확인사항
- [ ] 성능 목표 달성 확인
- [ ] 모니터링 대시보드 설정
- [ ] 운영 문서 업데이트
- [ ] 팀 회고 및 개선점 도출

---

