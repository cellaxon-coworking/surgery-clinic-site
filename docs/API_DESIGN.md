# API 설계 문서

## 1. API 개요

### 1.1 기본 정보
- **Base URL**: `https://api.example.com/api/v1`
- **데이터 형식**: JSON
- **인증 방식**: Bearer Token (JWT)
- **字符 인코딩**: UTF-8

### 1.2 공통 응답 형식

```typescript
// 성공 응답
interface SuccessResponse<T> {
  success: true;
  data: T;
}

// 에러 응답
interface ErrorResponse {
  success: false;
  error: {
    code: string;
    message: string;
    details?: any;
  };
}

// 페이지네이션
interface PaginatedResponse<T> {
  success: true;
  data: {
    items: T[];
    pagination: {
      page: number;
      limit: number;
      total: number;
      totalPages: number;
    };
  };
}
```

### 1.3 HTTP 상태 코드

| 코드 | 설명 |
|-----|------|
| 200 | 성공 |
| 201 | 생성 성공 |
| 400 | 요청 오류 |
| 401 | 인증 실패 |
| 403 | 권한 없음 |
| 404 | 리소스 없음 |
| 422 | 검증 실패 |
| 429 | 요청 초과 |
| 500 | 서버 오류 |

---

## 2. 인증 API

### 2.1 간편가입 (이메일)

#### 이메일 인증 코드 요청
```
POST /auth/email/send-code
```

**Request:**
```json
{
  "email": "user@example.com"
}
```

**Response:** 200 OK
```json
{
  "success": true,
  "data": {
    "expiresIn": 300  // 5분
  }
}
```

#### 이메일 인증 확인
```
POST /auth/email/verify
```

**Request:**
```json
{
  "email": "user@example.com",
  "code": "123456"
}
```

**Response:** 200 OK
```json
{
  "success": true,
  "data": {
    "verifyToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

#### 회원가입
```
POST /auth/register
```

**Request:**
```json
{
  "verifyToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "name": "홍길동",
  "nickname": "길동이",
  "password": "SecurePassword123!"
}
```

**Response:** 201 Created
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "email": "user@example.com",
      "name": "홍길동",
      "nickname": "길동이"
    },
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

### 2.2 소셜 로그인

#### Google 로그인 시작
```
GET /auth/google
```

**Redirect:** Google OAuth 페이지

#### Google 콜백
```
GET /auth/google/callback?code=...
```

**Response:** 200 OK (리다이렉트 포함)

#### Kakao 로그인
```
GET /auth/kakao
GET /auth/kakao/callback
```

#### Naver 로그인
```
GET /auth/naver
GET /auth/naver/callback
```

### 2.3 로그인/로그아웃

#### 로그인
```
POST /auth/login
```

**Request:**
```json
{
  "email": "user@example.com",
  "password": "SecurePassword123!"
}
```

**Response:** 200 OK
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "email": "user@example.com",
      "name": "홍길동",
      "nickname": "길동이"
    },
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

#### 로그아웃
```
POST /auth/logout
```

**Headers:** `Authorization: Bearer {token}`

**Response:** 200 OK

---

## 3. 상담 게시판 API

### 3.1 카테고리 조회

```
GET /consultations/categories
```

**Response:** 200 OK
```json
{
  "success": true,
  "data": [
    {
      "id": "eye",
      "name": "눈 성형",
      "description": "쌍꺼풀, 눈매교정 등"
    },
    {
      "id": "nose",
      "name": "코 성형",
      "description": "코끝, 코등, 비중격 교정 등"
    },
    {
      "id": "contour",
      "name": "윤곽 성형",
      "description": "앞턱, 옆턱, 턱수술 등"
    },
    {
      "id": "breast",
      "name": "가슴 성형",
      "description": "확대, 축소, 재건 등"
    },
    {
      "id": "body",
      "name": "체형 교정",
      "description": "지방흡입, 브라치오 등"
    },
    {
      "id": "anti-aging",
      "name": "안티에이징",
      "description": "보톡스, 필러, 리프팅 등"
    },
    {
      "id": "skin",
      "name": "피부 시술",
      "description": "레이저, 색소침착 등"
    },
    {
      "id": "hair",
      "name": "모발 이식",
      "description": "탈모 치료, 모발 이식 등"
    }
  ]
}
```

### 3.2 공개 게시물 목록

```
GET /consultations/public
```

**Query Parameters:**
| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| category | string | 아니요 | 카테고리 ID |
| page | number | 아니요 | 페이지 번호 (기본값: 1) |
| limit | number | 아니요 | 페이지당 개수 (기본값: 10) |
| search | string | 아니요 | 검색어 |

**Response:** 200 OK
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "post-id-1",
        "category": {
          "id": "eye",
          "name": "눈 성형"
        },
        "title": "쌍꺼풀 수술 비용이 어떻게 되나요?",
        "content": "30대 남자인데...",
        "author": {
          "nickname": "궁금이",
          "isAnonymous": true
        },
        "replyCount": 1,
        "createdAt": "2024-03-15T10:30:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 10,
      "total": 45,
      "totalPages": 5
    }
  }
}
```

### 3.3 공개 게시물 상세

```
GET /consultations/public/{id}
```

**Response:** 200 OK
```json
{
  "success": true,
  "data": {
    "id": "post-id-1",
    "category": {
      "id": "eye",
      "name": "눈 성형"
    },
    "title": "쌍꺼풀 수술 비용이 어떻게 되나요?",
    "content": "30대 남자인데...\n\n상세 내용입니다.",
    "author": {
      "nickname": "궁금이",
      "isAnonymous": true
    },
    "attachments": [
      {
        "id": "attach-1",
        "url": "https://cdn.example.com/images/xxx.jpg",
        "thumbnailUrl": "https://cdn.example.com/thumbnails/xxx.jpg"
      }
    ],
    "replies": [
      {
        "id": "reply-1",
        "content": "안녕하세요, 병원입니다...\n\n답변 내용입니다.",
        "author": {
          "name": "김의사",
          "title": "원장",
          "isDoctor": true
        },
        "createdAt": "2024-03-15T14:20:00Z"
      }
    ],
    "createdAt": "2024-03-15T10:30:00Z",
    "updatedAt": "2024-03-15T14:20:00Z"
  }
}
```

### 3.4 내 게시물 목록

```
GET /consultations/my
```

**Headers:** `Authorization: Bearer {token}`

**Query Parameters:** page, limit

**Response:** 200 OK
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "post-id-2",
        "category": {
          "id": "nose",
          "name": "코 성형"
        },
        "title": "코 성형 상담 문의드립니다",
        "isPublic": false,
        "isAnswered": true,
        "createdAt": "2024-03-14T09:15:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 10,
      "total": 3,
      "totalPages": 1
    }
  }
}
```

### 3.5 게시물 상세 (내 게시물)

```
GET /consultations/my/{id}
```

**Headers:** `Authorization: Bearer {token}`

**Response:** 공개 상세와 동일 (비밀 게시물 포함)

### 3.6 게시물 작성

```
POST /consultations
```

**Headers:** `Authorization: Bearer {token}`

**Request:**
```json
{
  "category": "nose",
  "title": "코 성형 상담 문의드립니다",
  "content": "내용입니다...\n\n줄바꿈도 포함.",
  "isPublic": false,
  "attachments": [
    "uploaded-file-id-1",
    "uploaded-file-id-2"
  ]
}
```

**Response:** 201 Created
```json
{
  "success": true,
  "data": {
    "id": "new-post-id",
    "category": "nose",
    "title": "코 성형 상담 문의드립니다",
    "isPublic": false,
    "createdAt": "2024-03-19T12:00:00Z"
  }
}
```

### 3.7 게시물 수정

```
PATCH /consultations/{id}
```

**Headers:** `Authorization: Bearer {token}`

**Request:** 게시물 작성과 동일

**Response:** 200 OK

### 3.8 게시물 삭제

```
DELETE /consultations/{id}
```

**Headers:** `Authorization: Bearer {token}`

**Response:** 200 OK

---

## 4. 파일 업로드 API

### 4.1 이미지 업로드 (프리사인)

```
POST /files/upload/presign
```

**Headers:** `Authorization: Bearer {token}`

**Request:**
```json
{
  "filename": "image.jpg",
  "contentType": "image/jpeg",
  "fileSize": 1024000  // bytes
}
```

**Response:** 200 OK
```json
{
  "success": true,
  "data": {
    "uploadUrl": "https://storage.example.com/upload/xxx",
    "fileId": "file-id-1",
    "expiresAt": "2024-03-19T12:30:00Z",
    "method": "PUT",
    "headers": {
      "Content-Type": "image/jpeg"
    }
  }
}
```

### 4.2 업로드 확인

```
POST /files/upload/confirm
```

**Headers:** `Authorization: Bearer {token}`

**Request:**
```json
{
  "fileId": "file-id-1"
}
```

**Response:** 200 OK
```json
{
  "success": true,
  "data": {
    "id": "file-id-1",
    "url": "https://cdn.example.com/images/xxx.jpg",
    "thumbnailUrl": "https://cdn.example.com/thumbnails/xxx.jpg",
    "size": 1024000,
    "width": 1920,
    "height": 1080
  }
}
```

---

## 5. 사용자 API

### 5.1 프로필 조회

```
GET /users/me
```

**Headers:** `Authorization: Bearer {token}`

**Response:** 200 OK
```json
{
  "success": true,
  "data": {
    "id": "user-id-1",
    "email": "user@example.com",
    "name": "홍길동",
    "nickname": "길동이",
    "provider": "email",
    "createdAt": "2024-01-01T00:00:00Z"
  }
}
```

### 5.2 프로필 수정

```
PATCH /users/me
```

**Headers:** `Authorization: Bearer {token}`

**Request:**
```json
{
  "nickname": "새로운닉네임"
}
```

**Response:** 200 OK

### 5.3 비밀번호 변경

```
POST /users/me/password
```

**Headers:** `Authorization: Bearer {token}`

**Request:**
```json
{
  "currentPassword": "OldPassword123!",
  "newPassword": "NewPassword456!"
}
```

**Response:** 200 OK

### 5.4 회원 탈퇴

```
DELETE /users/me
```

**Headers:** `Authorization: Bearer {token}`

**Request:**
```json
{
  "password": "Password123!",
  "reason": "더 이상 이용하지 않아서"
}
```

**Response:** 200 OK

---

## 6. 관리자 API

### 6.1 관리자 인증

#### 관리자 로그인
```
POST /admin/auth/login
```

**Request:**
```json
{
  "email": "admin@example.com",
  "password": "AdminPassword123!",
  "adminCode": "ADMIN_SECRET_CODE"
}
```

**Response:** 200 OK
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "admin-id-1",
      "email": "admin@example.com",
      "role": "admin"
    },
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

### 6.2 상담 관리

#### 전체 게시물 목록
```
GET /admin/consultations
```

**Headers:** `Authorization: Bearer {admin_token}`

**Query Parameters:**
| 파라미터 | 타입 | 설명 |
|---------|------|------|
| status | string | pending, answered, all |
| isPublic | boolean | 공개 여부 |
| category | string | 카테고리 |
| page | number | 페이지 번호 |
| limit | number | 페이지당 개수 |

**Response:** 200 OK
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "post-id-1",
        "category": "eye",
        "title": "쌍꺼풀 수술 비용 문의",
        "isPublic": true,
        "isAnswered": false,
        "author": {
          "nickname": "궁금이",
          "email": "user@example.com"
        },
        "createdAt": "2024-03-15T10:30:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 100,
      "totalPages": 5
    }
  }
}
```

#### 게시물 상세 (관리자용)
```
GET /admin/consultations/{id}
```

**Headers:** `Authorization: Bearer {admin_token}`

**Response:** 200 OK (모든 정보 포함)

### 6.3 답변 작성

```
POST /admin/consultations/{id}/replies
```

**Headers:** `Authorization: Bearer {admin_token}`

**Request:**
```json
{
  "content": "안녕하세요, 병원입니다.\n\n문의하신 내용에 대해 답변드립니다..."
}
```

**Response:** 201 Created

### 6.4 답변 수정/삭제

```
PATCH /admin/replies/{id}
DELETE /admin/replies/{id}
```

### 6.5 게시물 공개 상태 변경

```
PATCH /admin/consultations/{id}/visibility
```

**Headers:** `Authorization: Bearer {admin_token}`

**Request:**
```json
{
  "isPublic": true
}
```

**Response:** 200 OK

### 6.6 회원 관리

#### 회원 목록
```
GET /admin/users
```

**Query Parameters:** page, limit, search

**Response:** 200 OK
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "user-id-1",
        "email": "user@example.com",
        "name": "홍길동",
        "nickname": "길동이",
        "consultationCount": 5,
        "createdAt": "2024-01-01T00:00:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 150,
      "totalPages": 8
    }
  }
}
```

---

## 7. 콘텐츠 API (블로그/영상)

### 7.1 블로그 포스트

#### 포스트 목록
```
GET /posts
```

**Query Parameters:**
| 파라미터 | 타입 | 설명 |
|---------|------|------|
| category | string | 카테고리 |
| page | number | 페이지 번호 |
| limit | number | 페이지당 개수 |

**Response:** 200 OK
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "slug": "double-eyelid-surgery-guide",
        "title": "쌍꺼풀 수술 완벽 가이드",
        "excerpt": "쌍꺼풀 수술을 고민하고 계신다면...",
        "thumbnailUrl": "https://cdn.example.com/thumbnails/xxx.jpg",
        "publishedAt": "2024-03-10T00:00:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 10,
      "total": 25,
      "totalPages": 3
    }
  }
}
```

#### 포스트 상세
```
GET /posts/{slug}
```

**Response:** 200 OK
```json
{
  "success": true,
  "data": {
    "slug": "double-eyelid-surgery-guide",
    "title": "쌍꺼풀 수술 완벽 가이드",
    "content": "<p>쌍꺼풀 수술을 고민하고 계신다면...</p>",
    "thumbnailUrl": "https://cdn.example.com/thumbnails/xxx.jpg",
    "category": "eye",
    "publishedAt": "2024-03-10T00:00:00Z",
    "updatedAt": "2024-03-10T00:00:00Z"
  }
}
```

### 7.2 영상

#### 영상 목록
```
GET /videos
```

**Query Parameters:** category, page, limit

**Response:** 200 OK
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "video-1",
        "youtubeId": "dQw4w9WgXcQ",
        "title": "쌍꺼풀 수술 과정 설명",
        "description": "쌍꺼풀 수술의 전체 과정을...",
        "thumbnailUrl": "https://img.youtube.com/vi/dQw4w9WgXcQ/maxresdefault.jpg",
        "category": "eye",
        "publishedAt": "2024-03-01T00:00:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 12,
      "total": 30,
      "totalPages": 3
    }
  }
}
```

#### 영상 상세
```
GET /videos/{id}
```

**Response:** 200 OK (목록과 동일한 구조)

---

## 8. 통계 API

### 8.1 대시보드 데이터

```
GET /admin/stats/dashboard
```

**Headers:** `Authorization: Bearer {admin_token}`

**Response:** 200 OK
```json
{
  "success": true,
  "data": {
    "consultations": {
      "total": 500,
      "pending": 45,
      "answered": 455,
      "thisMonth": 32
    },
    "users": {
      "total": 320,
      "thisMonth": 15
    },
    "visits": {
      "today": 1234,
      "thisMonth": 45678
    }
  }
}
```

---

## 9. 에러 코드

| 코드 | 메시지 | 설명 |
|-----|--------|------|
| AUTH_001 | 이메일 또는 비밀번호가 올바르지 않습니다 | 로그인 실패 |
| AUTH_002 | 인증 코드가 올바르지 않습니다 | 인증 코드 오류 |
| AUTH_003 | 인증 코드가 만료되었습니다 | 인증 코드 만료 |
| AUTH_004 | 이미 가입된 이메일입니다 | 중복 가입 |
| AUTH_005 | 로그인이 필요합니다 | 인증 필요 |
| USER_001 | 존재하지 않는 사용자입니다 | 사용자 없음 |
| USER_002 | 비밀번호가 올바르지 않습니다 | 비밀번호 오류 |
| POST_001 | 존재하지 않는 게시물입니다 | 게시물 없음 |
| POST_002 | 권한이 없습니다 | 권한 없음 |
| POST_003 | 삭제된 게시물입니다 | 삭제된 게시물 |
| FILE_001 | 지원하지 않는 파일 형식입니다 | 파일 형식 오류 |
| FILE_002 | 파일 크기가 초과되었습니다 | 파일 크기 초과 |
| RATE_001 | 요청이 너무 많습니다 | Rate Limit |
