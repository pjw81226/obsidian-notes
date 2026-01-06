## 2026-01-06 작업/커밋 상세 정리 (binance-rise-alert)

### 작업 목표 요약
- **보안**: Discord Webhook URL 하드코딩 제거(레포 유출 방지)
- **안정성**: Producer WebSocket 끊김 복구 / Kafka 전송 내구성 강화
- **운영성**: 토픽명/파티션/오프셋 시작 위치를 환경변수로 분리(토픽 v2 컷오버 가능)
- **성능/정합성**: Flink 과도 로그 제거 / VolumeSpike 카운트 정합성 개선
- **알림 안정성**: Discord 429/5xx 재시도 + 전송 직렬화

---

## 커밋 1) `consumer: discord webhook env`

### 왜 했나
- Discord Webhook URL이 코드에 들어가 있으면 레포에 푸시되는 순간 **시크릿 유출**이 됩니다.
- 실행 시점에만 주입되도록 바꿔서 유출면적을 줄였습니다.

### 변경 내용
- `consumer`에서 Webhook URL을 **환경변수 `DISCORD_WEBHOOK_URL`로만** 받도록 변경
  - 값이 없으면 바로 실패(fail-fast)하도록 처리
- `docker-compose.yml`에서 `consumer` 컨테이너에 `DISCORD_WEBHOOK_URL`을 주입
- `README.md`에서 “Config.java를 직접 수정” 안내 제거 → “환경변수로 설정” 안내로 변경

### 운영 메모
- 과거 커밋 히스토리에 URL이 남아있을 수 있으니, **Webhook은 반드시 폐기/재발급**하세요.

---

## 커밋 2) `producer: ws reconnect`

### 왜 했나
- Binance WebSocket은 네트워크/서버 사유로 언제든 끊길 수 있는데,
  끊김 시 즉시 종료하면 데이터 수집이 멈춥니다.

### 변경 내용
- WebSocket 연결이 끊기거나 에러가 나면 **재연결 루프**로 자동 복구
  - 지수 백오프(기본 1s → 최대 60s)
  - 연결 성공 시 백오프 리셋
- 종료 시그널(프로세스 종료)에서
  - WebSocket 정상 close 시도
  - Kafka producer flush 시도
- 메시지 파편 수신 처리에서 마지막 조각일 때만 JSON 파싱/전송

### 새 환경변수(선택)
- `WS_RECONNECT_BASE_MS` (기본 `1000`)
- `WS_RECONNECT_MAX_MS` (기본 `60000`)

---

## 커밋 3) `producer: kafka reliability`

### 왜 했나
- Kafka 전송은 순간 장애/재시도로 인해 중복/유실 가능성이 생길 수 있어
  idempotence 기반으로 내구성을 올렸습니다.

### 변경 내용(Producer 설정 강화)
- `enable.idempotence=true`
- `acks=all`
- `retries=Integer.MAX_VALUE`
- `max.in.flight.requests.per.connection=5` (idempotence + ordering 전제)
- `delivery.timeout.ms`, `request.timeout.ms` 설정
- 처리량/네트워크 최적화: `linger.ms`, `batch.size`, `compression.type`

### 새 환경변수(선택)
- `KAFKA_COMPRESSION` (기본 `lz4`)

---

## 커밋 4) `infra: topic config`

### 왜 했나
- “심볼을 key로 파티션에 태우는 구조”에서 **파티션 수를 늘리면**
  key→partition 매핑이 바뀌어 심볼 순서 보장이 깨질 수 있습니다.
- 그래서 파티션 확장은 “기존 토픽 증설” 대신,
  **새 토픽(v2) 생성 → Producer/Flink 컷오버**가 가능한 구조로 바꿨습니다.

### 변경 내용(토픽명을 코드에서 제거하고 환경변수화)
- Producer가 RAW 토픽명을 환경변수로 받도록 변경:
  - `RAW_TOPIC_KLINE`, `RAW_TOPIC_AGGTRADE`, `RAW_TOPIC_BOOKTICKER`
- Flink도 RAW/Signal 토픽명을 환경변수로 받도록 변경:
  - `RAW_TOPIC_*`, `SIGNAL_TOPIC_VOLUME`, `SIGNAL_TOPIC_MOMENTUM`
- Flink의 Kafka source 시작 위치를 환경변수로 선택 가능:
  - `RAW_TOPIC_OFFSETS=earliest|latest` (기본 earliest)
- Producer의 토픽 자동 생성(`KafkaTopics.ensureDefaultTopics`)을 운영자가 제어 가능:
  - `ENSURE_TOPICS=true|false`
  - 파티션/복제계수도 환경변수로 설정 가능:
    - `RAW_TOPIC_PARTITIONS`, `SIGNAL_TOPIC_PARTITIONS`, `TOPIC_REPLICATION_FACTOR`
- `docker-compose.yml`에 Flink(jobmanager/taskmanager)와 consumer로 관련 env 주입
- `run_producer.sh`가 위 환경변수들을 받아 넘기도록 확장
- `README.md`에 v2 토픽 예시 추가

### 컷오버(토픽 마이그레이션) 권장 절차(요약)
1. v2 토픽을 **새 이름**으로 생성(파티션 수 크게)
2. Producer 중지(또는 v1 쓰기 중단)
3. Flink도 v1 소비를 중단/정리
4. Producer를 v2 토픽 환경변수로 재시작
5. Flink를 v2 토픽 환경변수로 재시작

---

## 커밋 5) `flink: reduce logs`

### 왜 했나
- Flink에서 이벤트마다 `System.out.println`을 찍으면
  트래픽 증가 시 **병목/비용/로그 폭증**이 됩니다.

### 변경 내용
- Kafka에서 받은 raw JSON을 매번 출력하던 로그 제거
- 파싱은 `map(DataParser::parse)`로 단순화(로직 변경 없음)

---

## 커밋 6) `flink: fix volume spike`

### 왜 했나
- 기존 `VolumeSpike`는 aggTrade를 단순 카운트했다가 kline final에서 리셋하는 구조라,
  **aggTrade가 어떤 kline 구간에 속하는지**가 애매해져 정합성이 흔들릴 수 있었습니다.

### 변경 내용(“최소 수정”)
- 심볼별 상태로 다음을 관리:
  - `currentKlineEnd` (현재 기준 kline endTime)
  - `tradeCountCurrent` (현재 캔들에 속한다고 보는 aggTrade 카운트)
  - `tradeCountNext` (다음 캔들로 넘길 카운트)
- aggTrade는 `ev.ts <= currentKlineEnd`면 current, 아니면 next로 카운트
- kline final 도착 시점에 endTime이 전진하면 `next -> current`로 shift
- 늦게 도착한 kline(endTime이 과거)은 무시(꼬임 방지)

### 기대 효과
- aggTrade 카운트가 “캔들 경계”와 조금 더 맞게 움직여
  volume spike 신호의 노이즈가 줄어드는 방향

---

## 커밋 7) `consumer: discord retry`

### 왜 했나
- Discord는 레이트리밋(429)이 자주 나고,
  일시 장애(5xx)도 있을 수 있는데 현재는 실패 시 그냥 로그만 남겨 알림이 끊길 수 있었습니다.

### 변경 내용
- Discord 전송을 즉시 HTTP로 보내지 않고,
  **내부 큐(최대 1000) + 단일 워커 스레드**로 직렬 처리
  - 메시지 전송 순서가 안정적으로 유지됨
  - 큐가 가득 차면 드랍(로그 출력)
- HTTP 응답별 처리
  - 2xx: 성공
  - 429: `Retry-After` 또는 `X-RateLimit-Reset-After` 헤더 기반 대기 후 재시도
  - 5xx: 지수 백오프 재시도(최대 5회)
  - 그 외: non-retryable로 보고 실패 로그 후 종료

### 주의/추가 개선 후보
- 현재 Kafka consumer는 기본 설정(자동 커밋)일 가능성이 높아,
  “Discord 전송 실패”가 나도 Kafka 오프셋은 진행될 수 있습니다.
  알림 at-least-once가 필요하면 **수동 커밋/재처리 전략**을 별도 설계하는 게 좋습니다.

---

## 이번 작업에서 추가된/중요 환경변수 목록

### 필수
- `DISCORD_WEBHOOK_URL`

### Producer (선택)
- `WS_RECONNECT_BASE_MS`, `WS_RECONNECT_MAX_MS`
- `KAFKA_COMPRESSION`
- `RAW_TOPIC_KLINE`, `RAW_TOPIC_AGGTRADE`, `RAW_TOPIC_BOOKTICKER`
- `ENSURE_TOPICS`, `RAW_TOPIC_PARTITIONS`, `SIGNAL_TOPIC_PARTITIONS`, `TOPIC_REPLICATION_FACTOR`

### Flink (선택)
- `RAW_TOPIC_KLINE`, `RAW_TOPIC_AGGTRADE`, `RAW_TOPIC_BOOKTICKER`
- `SIGNAL_TOPIC_VOLUME`, `SIGNAL_TOPIC_MOMENTUM`
- `RAW_TOPIC_OFFSETS=earliest|latest`

---

## 체크리스트(운영자가 지금 해야 할 일)
- [ ] Discord Webhook **폐기/재발급**
- [ ] 로컬/서버 런타임에 `DISCORD_WEBHOOK_URL` 환경변수 설정
- [ ] (필요 시) v2 토픽 전략에 맞춰 토픽명/파티션 env 세팅 후 컷오버 계획 수립