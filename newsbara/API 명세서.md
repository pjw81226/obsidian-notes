# Newsbara API 명세서

뉴스와 아티클을 읽고 정리하는 개인 뉴스 서비스 API

## 목차

- [개요](#개요)
- [공통 사항](#공통-사항)
  - [Base URL](#base-url)
  - [인증](#인증)
  - [공통 응답 구조](#공통-응답-구조)
  - [에러 코드](#에러-코드)
- [API 엔드포인트](#api-엔드포인트)
  - [인증 (Auth)](#인증-auth)
  - [사용자 (User)](#사용자-user)
  - [뉴스 (News)](#뉴스-news)
  - [아티클 (Article)](#아티클-article)
  - [스크랩 (Scrap)](#스크랩-scrap)
  - [정리글 (Summary)](#정리글-summary)
  - [키워드 (Keyword)](#키워드-keyword)
  - [공유 (Share)](#공유-share)
  - [태그 (Tag)](#태그-tag)

---

## 개요

Newsbara는 사용자가 뉴스와 아티클을 읽고, 스크랩하고, 정리할 수 있는 서비스입니다.

### 주요 기능
- 뉴스/아티클 수집 및 제공
- 스크랩 기능 (뉴스, 아티클)
- 정리글 작성 (공개/비공개)
- 키워드 구독
- 스트릭 시스템 (연속 작성 추적)
- 리더보드
- 캐릭터 진화 시스템

---

## 공통 사항

### Base URL

```
localhost:8080
```

### 비로그인 가능 페이지
- GET /api/news/daily (주요 뉴스 목록)
- GET /api/articles (아티클 목록)

- GET /api/summaries (isPublic=true만)
- GET /api/summaries/{id} (공개 정리글만)
- GET /api/share/{shareToken} (공유 링크)

- GET /api/tags/popular
- GET /api/users/leaderboard (상위 10명만)

- 홈화면의 카피바라 키우는 섹션, 현재 순위, 몇일 연속 인지, 내가 몇개 게시글 작성했는지는 전부 "로그인하고 이용해보세요" 등의 UI로 묶어두자.

### 인증

#### 인증 방식

OAuth 2.0 + JWT 기반 인증
- 지원 OAuth 제공자: Google, Kakao, Naver, GitHub
- 모든 인증된 요청에 Authorization 헤더에 Bearer 토큰 포함

```
Authorization: Bearer {access_token}
```

#### 토큰 종류

1. **Access Token**
   - 유효기간: 1시간
   - API 요청 시 사용
   - JWT 형식

2. **Refresh Token**
   - 유효기간: 7일
   - Access Token 갱신 시 사용
   - DB에 저장되어 관리
   - Refresh Token Rotation (RTR) 적용

#### 인증 플로우

```
1. 클라이언트 → OAuth 로그인 시작
   - 웹: GET /oauth2/authorize/{provider}?state=web
   - 앱: GET /oauth2/authorize/{provider}?state=android|ios

2. OAuth 제공자 → 사용자 인증 → 콜백

3. 백엔드 → JWT 생성 → Redis 임시 저장 (5분 TTL)
   → 플랫폼별 리다이렉트 (sessionId 포함)

4. 클라이언트 → POST /api/auth/exchange (sessionId 전달)
   → AccessToken + RefreshToken 수신

5. 클라이언트 → 토큰 저장
   - 웹: LocalStorage 또는 Memory
   - 앱: Secure Storage (Keychain/KeyStore)

6. 신규 사용자 → 온보딩 → POST /api/auth/onboarding
```

#### 보안 고려사항

- HTTPS 필수
- 토큰은 URL 파라미터로 전달하지 않음 (세션 기반 교환 방식)
- Refresh Token Rotation으로 재사용 방지
- Redis 세션은 일회용 (5분 TTL)

### 공통 응답 구조

#### 성공 응답

```json
{
  "success": true,
  "data": {
    // 실제 데이터
  },
  "message": "요청 성공",  // optional
  "timestamp": "2026-01-12T18:38:52+09:00"
}
```

#### 에러 응답

```json
{
  "success": false,
  "error": {
    "code": "USER-2000",
    "message": "사용자를 찾을 수 없습니다",
    "details": {  // optional
      // 추가 에러 정보
    }
  },
  "timestamp": "2026-01-12T18:38:52+09:00"
}
```

#### 페이징 응답

```json
{
  "success": true,
  "data": {
    "items": [
      // 데이터 목록
    ],
    "pagination": {
      "page": 0,
      "size": 20,
      "totalElements": 100,
      "totalPages": 5,
      "hasNext": true,
      "hasPrevious": false
    }
  },
  "timestamp": "2026-01-12T18:38:52+09:00"
}
```

### 에러 코드

#### 공통 에러 코드 (1000~1999)

| 코드 | HTTP 상태 | 설명 |
|------|-----------|------|
| COMMON-1000 | 400 | 잘못된 입력값입니다 |
| COMMON-1001 | 401 | 인증이 필요합니다 |
| COMMON-1002 | 403 | 접근 권한이 없습니다 |
| COMMON-1003 | 404 | 리소스를 찾을 수 없습니다 |
| COMMON-1004 | 500 | 서버 내부 오류가 발생했습니다 |
| COMMON-1005 | 405 | 허용되지 않은 HTTP 메서드입니다 |

#### 도메인별 에러 코드

추가 에러 코드는 각 도메인 섹션에서 정의됩니다.

---

## API 엔드포인트

### 1. 인증 (Auth)

#### 1.1. OAuth 로그인 시작

```
GET /oauth2/authorize/{provider}
```

**설명**: OAuth 제공자를 통한 로그인을 시작합니다.

**인증**: 불필요

**경로 파라미터**

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| provider | string | O | OAuth 제공자 (google, kakao, naver, github) |

**쿼리 파라미터**

| 파라미터 | 타입 | 필수 | 기본값 | 설명 |
|---------|------|------|--------|------|
| state | string | X | web | 플랫폼 구분 (web, android, ios) |

**응답**: OAuth 제공자 로그인 페이지로 리다이렉트

#### 1.2. OAuth 콜백

```
GET /oauth2/callback/{provider}
```

**설명**: OAuth 인증 후 콜백을 처리합니다. (내부 API, 클라이언트에서 직접 호출하지 않음)

**인증**: 불필요

**경로 파라미터**

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| provider | string | O | OAuth 제공자 |

**쿼리 파라미터**

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| code | string | O | OAuth 인증 코드 |
| state | string | X | 플랫폼 구분 |

**응답**: 플랫폼별 리다이렉트
- 웹: `https://newsbara.com/onboarding?session={sessionId}` (신규 사용자)
- 웹: `https://newsbara.com/home?session={sessionId}` (기존 사용자)
- 앱: `newsbara://auth/callback?session={sessionId}`

#### 1.3. 세션 교환 (토큰 발급)

```
POST /api/auth/exchange
```

**설명**: OAuth 로그인 후 임시 세션 ID를 JWT 토큰으로 교환합니다.

**인증**: 불필요

**요청 본문**

```json
{
  "sessionId": "550e8400-e29b-41d4-a716-446655440000"
}
```

**응답**

```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "tokenType": "Bearer",
    "expiresIn": 3600,
    "isNewUser": true,
    "userInfo": {
      "email": "user@example.com",
      "name": "홍길동",
      "profileImageUrl": "https://example.com/profile.jpg"
    }
  },
  "timestamp": "2026-01-12T19:00:00+09:00"
}
```

**에러**

| HTTP 상태 | 에러 코드 | 설명 |
|-----------|-----------|------|
| 401 | COMMON-1001 | 세션이 만료되었거나 유효하지 않음 |

#### 1.4. 토큰 갱신

```
POST /api/auth/refresh
```

**설명**: Refresh Token을 사용하여 새로운 Access Token을 발급받습니다.

**인증**: 불필요 (Refresh Token 필요)

**요청 본문**

```json
{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**응답**

```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "tokenType": "Bearer",
    "expiresIn": 3600
  },
  "timestamp": "2026-01-12T19:00:00+09:00"
}
```

**참고**: Refresh Token Rotation이 적용되어 매번 새로운 Refresh Token이 발급됩니다.

**에러**

| HTTP 상태 | 에러 코드 | 설명 |
|-----------|-----------|------|
| 401 | COMMON-1001 | Refresh Token이 유효하지 않거나 만료됨 |

#### 1.5. 온보딩 완료

```
POST /api/auth/onboarding
```

**설명**: 신규 사용자의 온보딩 정보(닉네임)를 등록합니다.

**인증**: 필요

**요청 본문**

```json
{
  "nickname": "카피바라"
}
```

**응답**

```json
{
  "success": true,
  "data": {
    "id": 1,
    "email": "user@example.com",
    "nickname": "카피바라",
    "profileImageUrl": "https://example.com/profile.jpg",
    "timezone": "Asia/Seoul",
    "role": "USER",
    "status": "ACTIVE",
    "createdAt": "2026-01-12T19:00:00+09:00",
    "updatedAt": "2026-01-12T19:00:00+09:00"
  },
  "message": "온보딩이 완료되었습니다",
  "timestamp": "2026-01-12T19:00:00+09:00"
}
```

**유효성 검증**

- nickname: 2~20자, 한글/영문/숫자 허용

**에러**

| HTTP 상태 | 에러 코드 | 설명 |
|-----------|-----------|------|
| 400 | COMMON-1000 | 닉네임 형식이 올바르지 않음 |
| 409 | USER-2001 | 이미 사용 중인 닉네임 |

#### 1.6. 로그아웃

```
POST /api/auth/logout
```

**설명**: 로그아웃하고 Refresh Token을 무효화합니다.

**인증**: 필요

**요청 본문**

```json
{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**응답**

```json
{
  "success": true,
  "message": "로그아웃되었습니다",
  "timestamp": "2026-01-12T19:00:00+09:00"
}
```

---

### 2. 사용자 (User)

#### 2.1. 내 정보 조회

```
GET /api/users/me
```

**설명**: 현재 로그인한 사용자의 정보를 조회합니다.

**인증**: 필요

**응답**

```json
{
  "success": true,
  "data": {
    "id": 1,
    "email": "user@example.com",
    "nickname": "카피바라",
    "profileImageUrl": "https://example.com/profile.jpg",
    "timezone": "Asia/Seoul",
    "currentStreak": 7,
    "bestStreak": 30,
    "totalArticles": 42,
    "characterLevel": 3,
    "role": "USER",
    "status": "ACTIVE",
    "createdAt": "2026-01-01T00:00:00+09:00",
    "updatedAt": "2026-01-12T18:38:52+09:00"
  },
  "timestamp": "2026-01-12T18:38:52+09:00"
}
```

#### 2.2. 프로필 수정

```
PATCH /api/users/me
```

**설명**: 사용자 프로필 정보를 수정합니다.

**인증**: 필요

**요청 본문**

```json
{
  "nickname": "새로운닉네임",
  "profileImageUrl": "https://example.com/new-profile.jpg",
  "timezone": "America/New_York"
}
```

**응답**

```json
{
  "success": true,
  "data": {
    "id": 1,
    "email": "user@example.com",
    "nickname": "새로운닉네임",
    "profileImageUrl": "https://example.com/new-profile.jpg",
    "timezone": "America/New_York",
    "updatedAt": "2026-01-12T18:40:00+09:00"
  },
  "message": "프로필이 수정되었습니다",
  "timestamp": "2026-01-12T18:40:00+09:00"
}
```

#### 2.3. 리더보드 조회

```
GET /api/users/leaderboard
```

**설명**: 글 작성 개수 기준 사용자 랭킹을 조회합니다.

**인증**: 필요

**쿼리 파라미터**

| 파라미터 | 타입 | 필수 | 기본값 | 설명 |
|---------|------|------|--------|------|
| page | integer | X | 0 | 페이지 번호 |
| size | integer | X | 20 | 페이지 크기 |

**응답**

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "rank": 1,
        "userId": 10,
        "nickname": "작성왕",
        "totalArticles": 500,
        "percentile": 99.9
      },
      {
        "rank": 2,
        "userId": 5,
        "nickname": "열심러",
        "totalArticles": 300,
        "percentile": 98.5
      }
    ],
    "pagination": {
      "page": 0,
      "size": 20,
      "totalElements": 1000,
      "totalPages": 50,
      "hasNext": true,
      "hasPrevious": false
    }
  },
  "timestamp": "2026-01-12T18:38:52+09:00"
}
```

#### 2.4. 캐릭터 정보 조회

```
GET /api/users/me/character
```

**설명**: 현재 사용자의 캐릭터 정보를 조회합니다.

**인증**: 필요

**응답**

```json
{
  "success": true,
  "data": {
    "level": 3,
    "name": "청소년 카피바라",
    "imageUrl": "https://example.com/capybara-level3.png",
    "currentArticles": 42,
    "nextLevelArticles": 50,
    "progress": 84.0
  },
  "timestamp": "2026-01-12T18:38:52+09:00"
}
```

---

### 3. 뉴스 (News)

#### 3.1. 주요 뉴스 목록 조회

```
GET /api/news/daily
```

**설명**: Python 크롤러가 수집한 주요 뉴스 목록을 조회합니다.

**인증**: 필요

**쿼리 파라미터**

| 파라미터 | 타입 | 필수 | 기본값 | 설명 |
|---------|------|------|--------|------|
| category | string | X | ALL | 뉴스 카테고리 (ALL, POLITICS, ECONOMY, SOCIETY, CULTURE, WORLD, TECH) |
| page | integer | X | 0 | 페이지 번호 |
| size | integer | X | 20 | 페이지 크기 |

**응답**

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": 1,
        "title": "뉴스 제목",
        "description": "뉴스 요약 내용...",
        "content": "뉴스 전체 내용...",
        "url": "https://news.example.com/article/1",
        "urlHash": "abc123",
        "thumbnailUrl": "https://example.com/thumbnail.jpg",
        "category": "TECH",
        "publishedAt": "2026-01-12T14:00:00+09:00",
        "createdAt": "2026-01-12T15:00:00+09:00"
      }
    ],
    "pagination": {
      "page": 0,
      "size": 20,
      "totalElements": 100,
      "totalPages": 5,
      "hasNext": true,
      "hasPrevious": false
    }
  },
  "timestamp": "2026-01-12T18:38:52+09:00"
}
```

#### 3.2. 키워드 기반 뉴스 검색

```
GET /api/news/search
```

**설명**: 사용자가 구독한 키워드 기반으로 네이버 API에서 뉴스를 검색합니다. (Redis 캐싱)

**인증**: 필요

**쿼리 파라미터**

| 파라미터 | 타입 | 필수 | 기본값 | 설명 |
|---------|------|------|--------|------|
| keyword | string | O | - | 검색 키워드 |
| sort | string | X | date | 정렬 방식 (sim: 정확도, date: 최신순) |
| page | integer | X | 0 | 페이지 번호 |
| size | integer | X | 20 | 페이지 크기 |

**응답**

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "title": "검색된 뉴스 제목",
        "description": "뉴스 요약",
        "url": "https://news.naver.com/article/...",
        "urlHash": "def456",
        "publishedAt": "2026-01-12T16:00:00+09:00"
      }
    ],
    "pagination": {
      "page": 0,
      "size": 20,
      "totalElements": 50,
      "totalPages": 3,
      "hasNext": true,
      "hasPrevious": false
    }
  },
  "timestamp": "2026-01-12T18:38:52+09:00"
}
```

---

### 4. 아티클 (Article)

#### 4.1, 아티클 목록 조회

```
GET /api/articles
```

**설명**: Python 크롤러가 수집한 아티클 목록을 조회합니다.

**인증**: 필요

**쿼리 파라미터**

| 파라미터 | 타입 | 필수 | 기본값 | 설명 |
|---------|------|------|--------|------|
| sourceType | string | X | ALL | 출처 타입 (ALL, MEDIUM, BRUNCH, TISTORY, VELOG) |
| page | integer | X | 0 | 페이지 번호 |
| size | integer | X | 20 | 페이지 크기 |

**응답**

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": 1,
        "title": "아티클 제목",
        "description": "아티클 요약",
        "content": "아티클 전체 내용",
        "url": "https://medium.com/@user/article",
        "urlHash": "ghi789",
        "thumbnailUrl": "https://example.com/article-thumb.jpg",
        "author": "작성자",
        "source": {
          "id": 1,
          "name": "Medium",
          "type": "MEDIUM",
          "baseUrl": "https://medium.com"
        },
        "publishedAt": "2026-01-11T10:00:00+09:00",
        "createdAt": "2026-01-11T12:00:00+09:00"
      }
    ],
    "pagination": {
      "page": 0,
      "size": 20,
      "totalElements": 200,
      "totalPages": 10,
      "hasNext": true,
      "hasPrevious": false
    }
  },
  "timestamp": "2026-01-12T18:38:52+09:00"
}
```

#### 4.2. 아티클 상세 조회

```
GET /api/articles/{articleId}
```

**설명**: 특정 아티클의 상세 정보를 조회합니다.

**인증**: 필요

**경로 파라미터**

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| articleId | integer | O | 아티클 ID |

**응답**

```json
{
  "success": true,
  "data": {
    "id": 1,
    "title": "아티클 제목",
    "description": "아티클 요약",
    "content": "아티클 전체 내용",
    "url": "https://medium.com/@user/article",
    "urlHash": "ghi789",
    "thumbnailUrl": "https://example.com/article-thumb.jpg",
    "author": "작성자",
    "source": {
      "id": 1,
      "name": "Medium",
      "type": "MEDIUM",
      "baseUrl": "https://medium.com"
    },
    "publishedAt": "2026-01-11T10:00:00+09:00",
    "createdAt": "2026-01-11T12:00:00+09:00"
  },
  "timestamp": "2026-01-12T18:38:52+09:00"
}
```

---

### 5. 스크랩 (Scrap)

#### 5.1. 스크랩 목록 조회

```
GET /api/scraps
```

**설명**: 사용자가 스크랩한 뉴스/아티클 목록을 조회합니다.

**인증**: 필요

**쿼리 파라미터**

| 파라미터 | 타입 | 필수 | 기본값 | 설명 |
|---------|------|------|--------|------|
| contentType | string | X | ALL | 컨텐츠 타입 (ALL, NEWS, ARTICLE) |
| page | integer | X | 0 | 페이지 번호 |
| size | integer | X | 20 | 페이지 크기 |

**응답**

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": 1,
        "contentType": "NEWS",
        "contentId": 5,
        "title": "스크랩한 뉴스 제목",
        "url": "https://news.example.com/article/5",
        "thumbnailUrl": "https://example.com/thumb.jpg",
        "memo": "나중에 읽기",
        "createdAt": "2026-01-12T10:00:00+09:00"
      },
      {
        "id": 2,
        "contentType": "ARTICLE",
        "contentId": 10,
        "title": "스크랩한 아티클 제목",
        "url": "https://medium.com/@user/article",
        "thumbnailUrl": "https://example.com/article.jpg",
        "memo": "중요한 내용",
        "createdAt": "2026-01-11T15:30:00+09:00"
      }
    ],
    "pagination": {
      "page": 0,
      "size": 20,
      "totalElements": 50,
      "totalPages": 3,
      "hasNext": true,
      "hasPrevious": false
    }
  },
  "timestamp": "2026-01-12T18:38:52+09:00"
}
```

#### 5.2. 스크랩 생성

```
POST /api/scraps
```

**설명**: 뉴스 또는 아티클을 스크랩합니다.

**인증**: 필요

**요청 본문**

```json
{
  "contentType": "NEWS",
  "contentId": 5,
  "memo": "나중에 읽기"
}
```

**응답**

```json
{
  "success": true,
  "data": {
    "id": 1,
    "contentType": "NEWS",
    "contentId": 5,
    "memo": "나중에 읽기",
    "createdAt": "2026-01-12T18:40:00+09:00"
  },
  "message": "스크랩되었습니다",
  "timestamp": "2026-01-12T18:40:00+09:00"
}
```

#### 5.3. 스크랩 삭제

```
DELETE /api/scraps/{scrapId}
```

**설명**: 스크랩을 삭제합니다.

**인증**: 필요

**경로 파라미터**

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| scrapId | integer | O | 스크랩 ID |

**응답**

```json
{
  "success": true,
  "message": "스크랩이 삭제되었습니다",
  "timestamp": "2026-01-12T18:40:00+09:00"
}
```

#### 5.4. 스크랩 메모 수정

```
PATCH /api/scraps/{scrapId}
```

**설명**: 스크랩의 메모를 수정합니다.

**인증**: 필요

**경로 파라미터**

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| scrapId | integer | O | 스크랩 ID |

**요청 본문**

```json
{
  "memo": "수정된 메모"
}
```

**응답**

```json
{
  "success": true,
  "data": {
    "id": 1,
    "memo": "수정된 메모",
    "updatedAt": "2026-01-12T18:40:00+09:00"
  },
  "message": "메모가 수정되었습니다",
  "timestamp": "2026-01-12T18:40:00+09:00"
}
```

---

### 6. 정리글 (Summary)

#### 6.1. 정리글 목록 조회

```
GET /api/summaries
```

**설명**: 사용자가 작성한 정리글 목록을 조회합니다.

**인증**: 필요

**쿼리 파라미터**

| 파라미터 | 타입 | 필수 | 기본값 | 설명 |
|---------|------|------|--------|------|
| contentType | string | X | ALL | 컨텐츠 타입 (ALL, NEWS, ARTICLE) |
| isPublic | boolean | X | - | 공개 여부 (true, false) |
| page | integer | X | 0 | 페이지 번호 |
| size | integer | X | 20 | 페이지 크기 |

**응답**

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": 1,
        "contentType": "NEWS",
        "contentId": 5,
        "title": "뉴스 정리글 제목",
        "summary": "정리한 내용...",
        "isPublic": true,
        "tags": ["경제", "IT"],
        "originalTitle": "원본 뉴스 제목",
        "originalUrl": "https://news.example.com/article/5",
        "createdAt": "2026-01-12T10:00:00+09:00",
        "updatedAt": "2026-01-12T11:00:00+09:00"
      }
    ],
    "pagination": {
      "page": 0,
      "size": 20,
      "totalElements": 42,
      "totalPages": 3,
      "hasNext": true,
      "hasPrevious": false
    }
  },
  "timestamp": "2026-01-12T18:38:52+09:00"
}
```

#### 6.2. 정리글 상세 조회

```
GET /api/summaries/{summaryId}
```

**설명**: 특정 정리글의 상세 정보를 조회합니다.

**인증**: 필요 (공개 정리글의 경우 선택)

**경로 파라미터**

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| summaryId | integer | O | 정리글 ID |

**응답**

```json
{
  "success": true,
  "data": {
    "id": 1,
    "contentType": "NEWS",
    "contentId": 5,
    "title": "뉴스 정리글 제목",
    "summary": "정리한 내용...",
    "isPublic": true,
    "tags": ["경제", "IT"],
    "originalTitle": "원본 뉴스 제목",
    "originalUrl": "https://news.example.com/article/5",
    "author": {
      "id": 1,
      "nickname": "카피바라"
    },
    "createdAt": "2026-01-12T10:00:00+09:00",
    "updatedAt": "2026-01-12T11:00:00+09:00"
  },
  "timestamp": "2026-01-12T18:38:52+09:00"
}
```

#### 6.3. 정리글 작성

```
POST /api/summaries
```

**설명**: 뉴스 또는 아티클 기반으로 정리글을 작성합니다.

**인증**: 필요

**요청 본문**

```json
{
  "contentType": "NEWS",
  "contentId": 5,
  "title": "정리글 제목",
  "summary": "정리한 내용...",
  "isPublic": true,
  "tags": ["경제", "IT"]
}
```

**응답**

```json
{
  "success": true,
  "data": {
    "id": 1,
    "contentType": "NEWS",
    "contentId": 5,
    "title": "정리글 제목",
    "summary": "정리한 내용...",
    "isPublic": true,
    "tags": ["경제", "IT"],
    "createdAt": "2026-01-12T18:40:00+09:00"
  },
  "message": "정리글이 작성되었습니다",
  "timestamp": "2026-01-12T18:40:00+09:00"
}
```

#### 6.4. 정리글 수정

```
PUT /api/summaries/{summaryId}
```

**설명**: 정리글을 수정합니다.

**인증**: 필요

**경로 파라미터**

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| summaryId | integer | O | 정리글 ID |

**요청 본문**

```json
{
  "title": "수정된 제목",
  "summary": "수정된 내용...",
  "isPublic": false,
  "tags": ["경제", "IT", "뉴스"]
}
```

**응답**

```json
{
  "success": true,
  "data": {
    "id": 1,
    "title": "수정된 제목",
    "summary": "수정된 내용...",
    "isPublic": false,
    "tags": ["경제", "IT", "뉴스"],
    "updatedAt": "2026-01-12T18:45:00+09:00"
  },
  "message": "정리글이 수정되었습니다",
  "timestamp": "2026-01-12T18:45:00+09:00"
}
```

#### 6.5. 정리글 삭제

```
DELETE /api/summaries/{summaryId}
```

**설명**: 정리글을 삭제합니다.

**인증**: 필요

**경로 파라미터**

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| summaryId | integer | O | 정리글 ID |

**응답**

```json
{
  "success": true,
  "message": "정리글이 삭제되었습니다",
  "timestamp": "2026-01-12T18:45:00+09:00"
}
```

#### 6.6. 일일 정리글 작성 통계

```
GET /api/summaries/daily-stats
```

**설명**: 사용자의 일일 정리글 작성 통계를 조회합니다. (캘린더용)

**인증**: 필요

**쿼리 파라미터**

| 파라미터 | 타입 | 필수 | 기본값 | 설명 |
|---------|------|------|--------|------|
| year | integer | X | 현재년도 | 조회할 년도 |
| month | integer | X | 현재월 | 조회할 월 (1-12) |

**응답**

```json
{
  "success": true,
  "data": {
    "year": 2026,
    "month": 1,
    "dailyStats": [
      {
        "date": "2026-01-12",
        "count": 3
      },
      {
        "date": "2026-01-11",
        "count": 1
      }
    ],
    "totalCount": 4
  },
  "timestamp": "2026-01-12T18:38:52+09:00"
}
```

---

### 7. 키워드 (Keyword)

#### 7.1. 구독 키워드 목록 조회

```
GET /api/keywords
```

**설명**: 사용자가 구독 중인 키워드 목록을 조회합니다.

**인증**: 필요

**응답**

```json
{
  "success": true,
  "data": [
    {
      "id": 1,
      "keyword": "Spring Boot",
      "createdAt": "2026-01-01T10:00:00+09:00"
    },
    {
      "id": 2,
      "keyword": "AI",
      "createdAt": "2026-01-05T14:30:00+09:00"
    }
  ],
  "timestamp": "2026-01-12T18:38:52+09:00"
}
```

#### 7.2. 키워드 구독

```
POST /api/keywords
```

**설명**: 새로운 키워드를 구독합니다.

**인증**: 필요

**요청 본문**

```json
{
  "keyword": "React"
}
```

**응답**

```json
{
  "success": true,
  "data": {
    "id": 3,
    "keyword": "React",
    "createdAt": "2026-01-12T18:40:00+09:00"
  },
  "message": "키워드가 구독되었습니다",
  "timestamp": "2026-01-12T18:40:00+09:00"
}
```

#### 7.3. 키워드 구독 취소

```
DELETE /api/keywords/{keywordId}
```

**설명**: 키워드 구독을 취소합니다.

**인증**: 필요

**경로 파라미터**

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| keywordId | integer | O | 키워드 ID |

**응답**

```json
{
  "success": true,
  "message": "키워드 구독이 취소되었습니다",
  "timestamp": "2026-01-12T18:40:00+09:00"
}
```

---

### 8. 공유 (Share)

#### 8.1. 공유 링크 생성

```
POST /api/share/summaries/{summaryId}
```

**설명**: 정리글의 공유 링크를 생성합니다.

**인증**: 필요

**경로 파라미터**

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| summaryId | integer | O | 정리글 ID |

**요청 본문**

```json
{
  "expiresInDays": 7  // optional, 기본값: 30일
}
```

**응답**

```json
{
  "success": true,
  "data": {
    "shareToken": "abc123xyz",
    "shareUrl": "https://newsbara.com/share/abc123xyz",
    "expiresAt": "2026-01-19T18:40:00+09:00",
    "createdAt": "2026-01-12T18:40:00+09:00"
  },
  "message": "공유 링크가 생성되었습니다",
  "timestamp": "2026-01-12T18:40:00+09:00"
}
```

#### 8.2. 공유 링크로 정리글 조회

```
GET /api/share/{shareToken}
```

**설명**: 공유 링크를 통해 정리글을 조회합니다.

**인증**: 불필요

**경로 파라미터**

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| shareToken | string | O | 공유 토큰 |

**응답**

```json
{
  "success": true,
  "data": {
    "id": 1,
    "title": "공유된 정리글 제목",
    "summary": "정리한 내용...",
    "tags": ["경제", "IT"],
    "originalTitle": "원본 뉴스 제목",
    "originalUrl": "https://news.example.com/article/5",
    "author": {
      "nickname": "카피바라"
    },
    "createdAt": "2026-01-12T10:00:00+09:00"
  },
  "timestamp": "2026-01-12T18:38:52+09:00"
}
```

---

### 9. 태그 (Tag)

#### 9.1. 인기 태그 조회

```
GET /api/tags/popular
```

**설명**: 인기 태그 목록을 조회합니다.

**인증**: 필요

**쿼리 파라미터**

| 파라미터 | 타입 | 필수 | 기본값 | 설명 |
|---------|------|------|--------|------|
| limit | integer | X | 10 | 조회할 태그 개수 |

**응답**

```json
{
  "success": true,
  "data": [
    {
      "tag": "경제",
      "count": 150
    },
    {
      "tag": "IT",
      "count": 120
    },
    {
      "tag": "AI",
      "count": 80
    }
  ],
  "timestamp": "2026-01-12T18:38:52+09:00"
}
```

#### 9.2. 내 태그 목록 조회

```
GET /api/tags/me
```

**설명**: 사용자가 사용한 태그 목록을 조회합니다.

**인증**: 필요

**응답**

```json
{
  "success": true,
  "data": [
    {
      "tag": "경제",
      "count": 15
    },
    {
      "tag": "IT",
      "count": 12
    }
  ],
  "timestamp": "2026-01-12T18:38:52+09:00"
}
```

---

## 부록

### 데이터 타입

#### NewsCategory (Enum)
- `ALL`: 전체
- `POLITICS`: 정치
- `ECONOMY`: 경제
- `SOCIETY`: 사회
- `CULTURE`: 문화
- `WORLD`: 세계
- `TECH`: 기술/IT

#### SourceType (Enum)
- `MEDIUM`: Medium
- `BRUNCH`: Brunch
- `TISTORY`: Tistory
- `VELOG`: Velog

#### ContentType (Enum)
- `NEWS`: 뉴스
- `ARTICLE`: 아티클

#### UserRole (Enum)
- `USER`: 일반 사용자
- `ADMIN`: 관리자

#### UserStatus (Enum)
- `ACTIVE`: 활성
- `INACTIVE`: 비활성
- `DELETED`: 삭제됨

### Pagination

모든 목록 조회 API는 페이징을 지원합니다.

**기본값**
- `page`: 0 (첫 페이지)
- `size`: 20 (페이지 크기)

**응답 형식**
```json
{
  "pagination": {
    "page": 0,
    "size": 20,
    "totalElements": 100,
    "totalPages": 5,
    "hasNext": true,
    "hasPrevious": false
  }
}
```

### Timezone

- 서버는 UTC 기준으로 시간을 저장
- 사용자 활동일 계산은 사용자의 timezone 설정 기준
- 응답의 모든 시간은 ISO 8601 형식 (예: `2026-01-12T18:38:52+09:00`)
