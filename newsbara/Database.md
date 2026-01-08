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

Redis
