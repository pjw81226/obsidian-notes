1. 유저는 로그인 후에 메인화면에서 자신의 기사 정리 기록을 확인할 수 있다. (한달 단위)
2. 주요 뉴스를 확인할 수 있다. 
3. 사용자가 등록한 키워드에 대한 뉴스를 확인할 수 있다.
4. 아티클을 확인할 수 있다. (구분 탭 존재)
5. 사용자는 기사에 대한 글을 작성할 수 있다.
6. 사용자는 기사를 스크랩 할 수 있다.


디비에 저장하는 모든 시간은 UTC 기준으로
화면에 보여줄 때는 사용자의 타임존을 기준으로 변환해서 보여줘야한다.

article time
- 한국기준으로 1월 8일 04시에 올렸으면 
	user_daily_article_summary_count 는 1월 8일에 증가시키고 
	아티클의 생성시간은 1월 7일 19시 인것


MySQL

base entity는 createdAt, updatedAt

- **user_daily_article_summary_count** 
	- 캘린더의 정보라고 생각하면됨
	- 유저가 특정 날짜에 몇개의 글을 정리했는지 카운트 (카운트는 유저의 timezone으로)

- **user**
	- 여기에 provider, email, name, profile image 들어감
	- `last_active_ymd` : 가장 마지막에 글 작성한 날짜 (KST 사용자 timezone)
		- 작성완료 처리시
			- 같은 날(ymd == last_active_ymd)면 streak 변화 없음
			- 다음 날(ymd == last_active_ymd + 1)면 current_streak++
			- 그 이상 띄면 current_streak=1
			- best_streak = max(best_streak, current_streak)
	- `current_streak` : 연속 작성 일자
		- 주기적으로 체크해야할듯.. 유저의 타임존마다 스케줄러가 돌면서 초기화 시켜야함
	- `best_streak` : 최고 기록

- daily_news

- article_sources

- articles

- user_scrap

- user_summary_news

- user_summary_article

- quoted_item 인용된 글 저장 (url_hash 로 유니크 한번 저장된 글은 계속 저장시켜놓음)

- user_subscription_keyword (유저가 저장한 키워드)


# Redis
### 뉴스 리스트
- `nl:{sort}:{qk}` (qk = 키워드를 **정규화한 뒤 해시로 만든 값**)
	- 예: `nl:sim:9f2a...` / `nl:date:9f2a...`
	- 값: 기사 배열(JSON) 또는 msgpack
	- TTL:
	    - sim: 30~60분
	    - date: 5~15분
	- 저장 개수: 키워드당 top 20~50개 정도

### 썸네일 캐시 (핫) 키
- `th:{urlHash}`
	- - 예: `th:ab12cd...`
	- 값(권장): 작은 JSON
	    - OK: `{ "s":"OK", "u":"https://...", "ts":170... }`
	    - FAIL: `{ "s":"FAIL", "ra":170... }` (retryAfter epoch)
	- TTL:
	    - OK: 1~7일(메모리 아끼면 1~3일)
	    - FAIL: 1~6시간

### 락(스탬피드 방지) 키
- 리스트 갱신 락 : `lk:nl:{sort}:{qk}`
	- TTL: 10~30초
	- 용도: 캐시 미스 시 **1명만 네이버 API 호출**하게
- 썸네일 생성 락 : `lk:th:{urlHash}`
	- TTL: 30~120초
	- 용도: 여러 클라이언트가 동시에 `/thumbs`를 때릴 때 **OG 파싱 중복 방지**

### 도메인 → 언론사 매핑 캐시(선택)
- `pub:host:{host}`
	- 예: `pub:host:sedaily.com -> "서울경제"`
	- TTL: 길게(30~180일) 또는 영구
	- 값: `{name, logoUrl, updatedAt}`

### “유저 키워드 목록” 캐시(선택)
- `u:{userId}:kw`
	- 값: 유저 키워드 배열(JSON)
	- TTL: 1~6시간(또는 이벤트 기반 무효화)
