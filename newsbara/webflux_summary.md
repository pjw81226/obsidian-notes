# RestTemplate → WebClient 전환 요약

> ⚠️ **비동기 전환 아님!** WebClient를 동기(`block()`)로 사용

## 변경 사항
- RestTemplate → WebClient + `block()` (동기)
- 분산 락 → RedisLockUtil (UUID + Lua 스크립트)
- Virtual Thread 활성화

## 왜 WebClient? (동기로 써도)
1. RestTemplate deprecated 예정
2. Connection Pool 자동 관리
3. 나중에 비동기 전환 쉬움 (block() 제거만)

## 왜 비동기 안 함?
1. JPA가 블로킹 → R2DBC 전환 필요 (큰 공수)
2. 복잡도 증가
3. Virtual Thread로 충분

## 삭제된 파일
- `RestTemplateConfig.java`
- `ReactiveRedisConfig.java`
- `thumbnail/` 디렉토리

## 현재 구조
```
Controller (동기) → Service (동기, JPA) → WebClient.block() → 외부 API
```
