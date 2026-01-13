# User Domain

- **users**
	- 유저의 프로필 정보
	- timezone
	- 총 작성 게시글 수
	- 마지막 활동 날짜, 현재 연속 작성 일수, 최고 연속 작성 일수
- **userSubscriptionKeyword**
	- 유저가 구독한 키워드

# Summary Domain

- **userDailySummaryCount**
	- 유저ID, 일자, count
- **SummaryArticle**
	- 유저ID
	- ArticleId (Articles 테이블 의존, 어차피 Articles 테이블은 영속성)
	- ~~제목~~ (이건 빼도될듯)
	- is_public (공개)
- **SummaryNews**
	- PersistedNewsId (영속성 뉴스 테이블 의존, 뉴스 정리시 해당 뉴스를 영속성 테이블로 이동)

# Tag
Summary에 붙이는 유저의 커스텀 태그
- **Tag**
	- 이름
	- 사용카운트

- **SummaryTag**
	- summary article, summary news 아이디
	- 태그 아이디

# ShareLink
- **ShareLink**
	- shareToken : 고유의 uuid 기본 베이스 url에 이거붙여서 사용자 게시글 보여줄듯
- **ViewCount**
	- viewCount : 보여지는 카운트

# Scrap
- **User Scrap**
	- 아티클Id, persisted news ID
	- 메모 (이건 필요없을지도)

# Notifcation
- **userNotifaciton**
	- 유저 알림

# News
- **Daily News**
- **Persisted News**

# CharacterLevel
- CharacterLevel
	- 레벨
	- 필요 아티클
	- 설명
	- 이미지url