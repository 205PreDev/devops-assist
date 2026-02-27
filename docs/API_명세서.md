# devops-assist API 명세서

> 버전: 0.1 | 작성일: 2026-02-27 | 상태: 초안

---

## 1. 공통 규칙

### Base URL
```
개발: http://localhost:8080/api
배포: http://{host}:8080/api
```

### 인증
모든 API (인증 제외)는 헤더에 Access Token 필요:
```
Authorization: Bearer {accessToken}
```

### Content-Type
```
요청: Content-Type: application/json
응답: Content-Type: application/json
SSE: Content-Type: text/event-stream
```

### 날짜 형식
```
ISO 8601: 2026-02-27T10:30:00Z
```

### 공통 에러 응답
```json
{
  "timestamp": "2026-02-27T10:30:00Z",
  "status": 401,
  "error": "UNAUTHORIZED",
  "code": "AUTH_TOKEN_EXPIRED",
  "message": "액세스 토큰이 만료되었습니다.",
  "path": "/api/documents"
}
```

### HTTP 상태코드 정책

| 코드 | 의미 | 사용 상황 |
|------|------|-----------|
| 200 | OK | 조회, 수정 성공 |
| 201 | Created | 생성 성공 |
| 204 | No Content | 삭제 성공, 로그아웃 |
| 400 | Bad Request | 입력값 유효성 오류 |
| 401 | Unauthorized | 토큰 없음/만료/유효하지 않음 |
| 403 | Forbidden | 타인 리소스 접근 시도 |
| 404 | Not Found | 리소스 없음 |
| 409 | Conflict | 중복 (이메일, 태그명 등) |
| 500 | Internal Server Error | 서버 오류 |
| 503 | Service Unavailable | Gemini API 응답 불가 |

### 도메인별 에러 코드

| 코드 | 상태 | 설명 |
|------|------|------|
| `AUTH_EMAIL_DUPLICATE` | 409 | 이미 사용 중인 이메일 |
| `AUTH_INVALID_CREDENTIALS` | 401 | 이메일 또는 비밀번호 불일치 |
| `AUTH_TOKEN_EXPIRED` | 401 | Access Token 만료 |
| `AUTH_TOKEN_INVALID` | 401 | Access Token 유효하지 않음 |
| `AUTH_REFRESH_TOKEN_INVALID` | 401 | Refresh Token 유효하지 않음 또는 만료 |
| `DOC_NOT_FOUND` | 404 | 문서를 찾을 수 없음 |
| `DOC_ACCESS_DENIED` | 403 | 타인의 문서 접근 |
| `FOLD_NOT_FOUND` | 404 | 폴더를 찾을 수 없음 |
| `FOLD_ACCESS_DENIED` | 403 | 타인의 폴더 접근 |
| `FOLD_MAX_DEPTH` | 400 | 폴더 최대 깊이(5단계) 초과 |
| `TAG_NOT_FOUND` | 404 | 태그를 찾을 수 없음 |
| `TAG_DUPLICATE` | 409 | 동일 워크스페이스에 같은 태그명 존재 |
| `AI_SERVICE_UNAVAILABLE` | 503 | Gemini API 응답 실패 |
| `VALIDATION_ERROR` | 400 | 입력값 유효성 검증 실패 |

---

## 2. 인증 API

### POST /api/auth/signup — 회원가입

**요청**
```json
{
  "email": "user@example.com",
  "password": "password123!",
  "username": "홍길동"
}
```

**유효성 규칙**

| 필드 | 규칙 |
|------|------|
| email | 이메일 형식, 최대 255자 |
| password | 8~72자, 영문+숫자 포함 |
| username | 2~50자 |

**응답 201**
```json
{
  "id": 1,
  "email": "user@example.com",
  "username": "홍길동",
  "createdAt": "2026-02-27T10:30:00Z"
}
```

**에러**

| 코드 | 상황 |
|------|------|
| `AUTH_EMAIL_DUPLICATE` | 이미 가입된 이메일 |
| `VALIDATION_ERROR` | 입력값 형식 오류 |

---

### POST /api/auth/login — 로그인

**요청**
```json
{
  "email": "user@example.com",
  "password": "password123!"
}
```

**응답 200**
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiJ9...",
  "tokenType": "Bearer",
  "expiresIn": 900
}
```
> Refresh Token은 `HttpOnly; Secure; SameSite=Strict` 쿠키로 별도 전송
> 쿠키명: `refresh_token`, 유효기간: 7일

**에러**

| 코드 | 상황 |
|------|------|
| `AUTH_INVALID_CREDENTIALS` | 이메일 또는 비밀번호 불일치 |

---

### POST /api/auth/refresh — Access Token 재발급

**요청**
본문 없음. `refresh_token` 쿠키 자동 포함.

**응답 200**
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiJ9...",
  "tokenType": "Bearer",
  "expiresIn": 900
}
```
> Refresh Token Rotation 적용: 기존 Refresh Token 무효화 + 새 Refresh Token 쿠키 발급

**에러**

| 코드 | 상황 |
|------|------|
| `AUTH_REFRESH_TOKEN_INVALID` | Refresh Token 없음, 만료, 또는 이미 사용됨 |

---

### POST /api/auth/logout — 로그아웃

**요청**
본문 없음. `refresh_token` 쿠키 자동 포함.

**응답 204**
본문 없음. Refresh Token DB 삭제 + 쿠키 만료 처리.

---

## 3. 문서 API

### GET /api/documents — 문서 목록

**쿼리 파라미터**

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| folderId | Long | 선택 | 특정 폴더의 문서만 조회 (없으면 전체) |
| tagId | Long | 선택 | 특정 태그의 문서만 조회 |
| favorited | Boolean | 선택 | true이면 즐겨찾기만 조회 |
| sort | String | 선택 | `updatedAt`(기본) \| `createdAt` \| `title` |

**응답 200**
```json
[
  {
    "id": 1,
    "title": "Spring Security 정리",
    "folderId": 2,
    "folderName": "Spring",
    "tags": ["spring", "security", "jwt"],
    "isFavorited": false,
    "createdAt": "2026-02-27T10:00:00Z",
    "updatedAt": "2026-02-27T11:30:00Z"
  }
]
```

---

### POST /api/documents — 문서 생성

**요청**
```json
{
  "title": "새 문서",
  "folderId": 2
}
```
> folderId 생략 시 루트(미분류)에 생성

**응답 201**
```json
{
  "id": 5,
  "title": "새 문서",
  "content": null,
  "folderId": 2,
  "tags": [],
  "isFavorited": false,
  "createdAt": "2026-02-27T12:00:00Z",
  "updatedAt": "2026-02-27T12:00:00Z"
}
```

---

### GET /api/documents/{id} — 문서 상세 조회

**응답 200**
```json
{
  "id": 1,
  "title": "Spring Security 정리",
  "content": {
    "type": "doc",
    "content": [
      {
        "type": "heading",
        "attrs": { "level": 1 },
        "content": [{ "type": "text", "text": "Spring Security" }]
      },
      {
        "type": "paragraph",
        "content": [{ "type": "text", "text": "Spring Security는..." }]
      }
    ]
  },
  "folderId": 2,
  "folderName": "Spring",
  "tags": ["spring", "security"],
  "isFavorited": false,
  "createdAt": "2026-02-27T10:00:00Z",
  "updatedAt": "2026-02-27T11:30:00Z"
}
```
> `content`는 Tiptap JSON 구조 그대로 반환

**에러**

| 코드 | 상황 |
|------|------|
| `DOC_NOT_FOUND` | 문서 없음 |
| `DOC_ACCESS_DENIED` | 타인의 문서 |

---

### PUT /api/documents/{id} — 문서 수정 (자동저장 겸용)

**요청**
```json
{
  "title": "Spring Security 정리 v2",
  "content": { "type": "doc", "content": [...] },
  "tags": ["spring", "security", "jwt"],
  "isFavorited": false
}
```
> 부분 수정 가능 — 보내지 않은 필드는 기존 값 유지
> `tags` 배열 기준으로 태그 자동 생성/연결/해제 처리

**응답 200**
```json
{
  "id": 1,
  "title": "Spring Security 정리 v2",
  "tags": ["spring", "security", "jwt"],
  "updatedAt": "2026-02-27T11:35:00Z"
}
```

**에러**

| 코드 | 상황 |
|------|------|
| `DOC_NOT_FOUND` | 문서 없음 |
| `DOC_ACCESS_DENIED` | 타인의 문서 |

---

### DELETE /api/documents/{id} — 문서 삭제

**응답 204** — 본문 없음

**에러**

| 코드 | 상황 |
|------|------|
| `DOC_NOT_FOUND` | 문서 없음 |
| `DOC_ACCESS_DENIED` | 타인의 문서 |

---

### PATCH /api/documents/{id}/move — 문서 이동

**요청**
```json
{
  "folderId": 3
}
```
> folderId를 `null`로 보내면 루트(미분류)로 이동

**응답 200**
```json
{
  "id": 1,
  "folderId": 3,
  "updatedAt": "2026-02-27T12:00:00Z"
}
```

---

## 4. 폴더 API

### GET /api/folders — 폴더 트리 조회

**응답 200**
재귀 트리 구조로 반환:
```json
[
  {
    "id": 1,
    "name": "개발",
    "orderIndex": 0,
    "children": [
      {
        "id": 2,
        "name": "Spring",
        "orderIndex": 0,
        "children": []
      },
      {
        "id": 3,
        "name": "React",
        "orderIndex": 1,
        "children": []
      }
    ]
  }
]
```

---

### POST /api/folders — 폴더 생성

**요청**
```json
{
  "name": "새 폴더",
  "parentId": 1
}
```
> parentId 생략 시 최상위 폴더 생성

**응답 201**
```json
{
  "id": 4,
  "name": "새 폴더",
  "parentId": 1,
  "orderIndex": 2,
  "children": []
}
```

**에러**

| 코드 | 상황 |
|------|------|
| `FOLD_NOT_FOUND` | parentId 폴더 없음 |
| `FOLD_MAX_DEPTH` | 5단계 깊이 초과 |

---

### PUT /api/folders/{id} — 폴더 이름 변경

**요청**
```json
{
  "name": "백엔드"
}
```

**응답 200**
```json
{
  "id": 1,
  "name": "백엔드"
}
```

---

### DELETE /api/folders/{id} — 폴더 삭제

**동작**: 폴더 삭제 시 하위 문서는 루트(미분류)로 이동. 하위 폴더도 함께 삭제.

**응답 204** — 본문 없음

**에러**

| 코드 | 상황 |
|------|------|
| `FOLD_NOT_FOUND` | 폴더 없음 |
| `FOLD_ACCESS_DENIED` | 타인의 폴더 |

---

## 5. 태그 API

### GET /api/tags — 태그 목록

**응답 200**
```json
[
  {
    "id": 1,
    "name": "spring",
    "documentCount": 5
  },
  {
    "id": 2,
    "name": "jwt",
    "documentCount": 2
  }
]
```

---

### PUT /api/tags/{id} — 태그 이름 변경

**요청**
```json
{
  "name": "spring-boot"
}
```

**응답 200**
```json
{
  "id": 1,
  "name": "spring-boot"
}
```

**에러**

| 코드 | 상황 |
|------|------|
| `TAG_NOT_FOUND` | 태그 없음 |
| `TAG_DUPLICATE` | 이미 같은 이름의 태그 존재 |

---

### DELETE /api/tags/{id} — 태그 삭제

**동작**: 태그 삭제 시 연결된 모든 문서에서 해당 태그 제거.

**응답 204** — 본문 없음

---

## 6. AI API

> AI는 보조 역할로 두 가지 방식으로 동작한다:
> - **인라인 제안**: 입력창 포커스 시 즉각 호출, Tab으로 수락
> - **온디맨드**: 버튼 클릭으로 요약/제안/추천 실행

### POST /api/ai/inline/suggest — 인라인 제안 (태그·제목·폴더명)

**요청**
```json
{
  "type": "tag",
  "documentTitle": "Spring Security 정리",
  "documentContent": "Spring Security는 인증과 인가를 담당하는...",
  "existingTags": ["spring"]
}
```

**type 값**

| type | 설명 |
|------|------|
| `tag` | 태그 후보 3~5개 제안 |
| `title` | 문서 제목 1개 제안 |
| `folder` | 폴더명 1개 제안 |

**응답 200**
```json
{
  "type": "tag",
  "suggestions": ["security", "jwt", "authentication"]
}
```
> `title`, `folder` 타입은 `suggestions` 배열에 1개만 반환

**특성**
- 응답 목표: 500ms 이내 (인라인 UX 체감 속도)
- 캐싱: 동일 문서 내용에 대한 중복 호출은 1분간 캐시 응답

---

### POST /api/ai/related/{docId} — 관련 문서 추천

**요청** — 본문 없음

**응답 200**
```json
{
  "sourceDocId": 1,
  "recommendations": [
    {
      "docId": 3,
      "title": "JWT 인증 구현",
      "reason": "Spring Security와 JWT 토큰 관련 내용을 공통으로 다룸",
      "score": 0.87,
      "commonTags": ["spring", "jwt"]
    },
    {
      "docId": 7,
      "title": "OAuth2 소셜 로그인",
      "reason": "인증/인가 흐름이 유사함",
      "score": 0.72,
      "commonTags": ["spring", "security"]
    }
  ]
}
```

**에러**

| 코드 | 상황 |
|------|------|
| `DOC_NOT_FOUND` | 문서 없음 |
| `AI_SERVICE_UNAVAILABLE` | Gemini API 응답 실패 |

---

### GET /api/ai/summarize/{docId} — 문서 요약 (SSE 스트리밍)

**응답** `Content-Type: text/event-stream`
```
data: {"chunk": "이 문서는 Spring Security의 "}

data: {"chunk": "기본 개념과 JWT 인증 흐름을 "}

data: {"chunk": "설명합니다. 핵심 내용은..."}

data: [DONE]
```

**에러** (스트리밍 중 오류 시)
```
data: {"error": "AI_SERVICE_UNAVAILABLE", "message": "AI 서비스에 일시적인 오류가 발생했습니다."}

data: [DONE]
```

---

### GET /api/ai/suggest/{docId} — 보완 제안 (SSE 스트리밍)

**응답** `Content-Type: text/event-stream`
요약과 동일한 SSE 형식. 내용은 "이런 내용을 추가하면 좋겠어요:" 형태.

---

### POST /api/ai/search — 자연어 검색 *(Phase A 후반)*

**요청**
```json
{
  "query": "JWT 토큰 만료 처리 방법"
}
```

**응답 200**
```json
{
  "query": "JWT 토큰 만료 처리 방법",
  "results": [
    {
      "docId": 1,
      "title": "Spring Security 정리",
      "relevantText": "...액세스 토큰 만료 시 리프레시 토큰으로 재발급하는 방법은...",
      "score": 0.91
    }
  ]
}
```

---

### POST /api/ai/tags/{docId} — 태그 자동 제안 *(Phase A 후반)*

**요청** — 본문 없음

**응답 200**
```json
{
  "docId": 1,
  "suggestions": ["spring-security", "jwt", "authentication", "token", "filter"]
}
```
