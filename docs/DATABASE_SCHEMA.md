# 데이터베이스 스키마 설계

## 1. 개요

### 1.1 DBMS 선정
- **주 DB**: PostgreSQL 15+ (서울 리전)
- **이유**: ACID 보장, JSON 지원, 풍부한 기능

### 1.2 네이밍 규칙
- 테이블명: 소문자, 스네이크 케이스 (`user_profiles`)
- 컬럼명: 소문자, 스네이크 케이스 (`created_at`)
- Primary Key: `id` (UUID)
- Timestamp: `created_at`, `updated_at`

### 1.3 공통 컬럼

```sql
-- 모든 테이블에 포함
created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
deleted_at TIMESTAMP WITH TIME ZONE  -- Soft Delete
```

---

## 2. 사용자 (Users)

### 2.1 users

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255),              -- NULL for social login
    name VARCHAR(100) NOT NULL,
    nickname VARCHAR(50),
    phone VARCHAR(20),
    birth_date DATE,                         -- 생년월일 (의료 상담용)

    -- 소셜 로그인 정보
    provider VARCHAR(20) NOT NULL DEFAULT 'email',  -- email, google, kakao, naver
    provider_id VARCHAR(255),                -- 소셜 ID
    provider_data JSONB,                     -- 소셜 프로필 추가 정보

    -- 계정 상태
    status VARCHAR(20) NOT NULL DEFAULT 'active',  -- active, suspended, withdrawn
    email_verified BOOLEAN NOT NULL DEFAULT FALSE,
    phone_verified BOOLEAN NOT NULL DEFAULT FALSE,

    -- 마케팅 동의
    marketing_agreed BOOLEAN NOT NULL DEFAULT FALSE,
    privacy_agreed_at TIMESTAMP WITH TIME ZONE,

    -- 타임스탬프
    last_login_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP WITH TIME ZONE
);

-- 인덱스
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_provider ON users(provider, provider_id);
CREATE INDEX idx_users_status ON users(status);
CREATE INDEX idx_users_created_at ON users(created_at);

-- 트리거: updated_at 자동 업데이트
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_users_updated_at BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

### 2.2 refresh_tokens

```sql
CREATE TABLE refresh_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token VARCHAR(500) UNIQUE NOT NULL,
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    revoked_at TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_refresh_tokens_user ON refresh_tokens(user_id);
CREATE INDEX idx_refresh_tokens_token ON refresh_tokens(token);
CREATE INDEX idx_refresh_tokens_expires ON refresh_tokens(expires_at);
```

---

## 3. 상담 게시판 (Consultations)

### 3.1 consultation_categories

```sql
CREATE TABLE consultation_categories (
    id VARCHAR(50) PRIMARY KEY,              -- eye, nose, breast 등
    name VARCHAR(100) NOT NULL,
    name_en VARCHAR(100),
    description TEXT,
    display_order INT NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- 초기 데이터
INSERT INTO consultation_categories (id, name, name_en, description, display_order) VALUES
    ('eye', '눈 성형', 'Eye', '쌍꺼풀, 눈매교정, 매몰법, 절개법 등', 1),
    ('nose', '코 성형', 'Nose', '코끝, 코등, 비중격 교정, 코 만들기 등', 2),
    ('contour', '윤곽 성형', 'Facial Contour', '앞턱, 옆턱, 턱수술, 무턱 등', 3),
    ('breast', '가슴 성형', 'Breast', '가슴 확대, 축소, 재건, 유두 축소 등', 4),
    ('body', '체형 교정', 'Body Contour', '지방흡입, 브라치오, 복부성형 등', 5),
    ('anti-aging', '안티에이징', 'Anti-Aging', '보톡스, 필러, 리프팅, 실리프팅 등', 6),
    ('skin', '피부 시술', 'Skin', '레이저, 색소침착, 주름개선, 모공축소 등', 7),
    ('hair', '모발 이식', 'Hair Transplant', '탈모 치료, 모발 이식, 헤어라인 등', 8),
    ('etc', '기타 상담', 'Other', '기타 시술 문의', 99);
```

### 3.2 consultations

```sql
CREATE TABLE consultations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    category_id VARCHAR(50) NOT NULL REFERENCES consultation_categories(id),

    -- 게시물 정보
    title VARCHAR(200) NOT NULL,
    content TEXT NOT NULL,

    -- 공개 설정
    is_public BOOLEAN NOT NULL DEFAULT FALSE,
    is_anonymous BOOLEAN NOT NULL DEFAULT TRUE,    -- 작성자 익명 여부

    -- 진행 상태
    status VARCHAR(20) NOT NULL DEFAULT 'pending', -- pending, answered, completed, closed
    is_answered BOOLEAN NOT NULL DEFAULT FALSE,
    answered_at TIMESTAMP WITH TIME ZONE,

    -- 통계
    view_count INT NOT NULL DEFAULT 0,

    -- 관리자 메모
    admin_notes TEXT,

    -- 타임스탬프
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP WITH TIME ZONE
);

-- 인덱스
CREATE INDEX idx_consultations_user ON consultations(user_id);
CREATE INDEX idx_consultations_category ON consultations(category_id);
CREATE INDEX idx_consultations_public ON consultations(is_public, is_answered, created_at DESC);
CREATE INDEX idx_consultations_status ON consultations(status);
CREATE INDEX idx_consultations_created_at ON consultations(created_at DESC);
```

### 3.3 consultation_replies

```sql
CREATE TABLE consultation_replies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    consultation_id UUID NOT NULL REFERENCES consultations(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    -- 답변 정보
    content TEXT NOT NULL,
    is_internal BOOLEAN NOT NULL DEFAULT FALSE,  -- 내부용 답변

    -- 타임스탬프
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP WITH TIME ZONE
);

-- 인덱스
CREATE INDEX idx_consultation_replies_consultation ON consultation_replies(consultation_id);
CREATE INDEX idx_consultation_replies_user ON consultation_replies(user_id);
```

### 3.4 consultation_attachments

```sql
CREATE TABLE consultation_attachments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    consultation_id UUID NOT NULL REFERENCES consultations(id) ON DELETE CASCADE,

    -- 파일 정보
    file_name VARCHAR(255) NOT NULL,
    file_url VARCHAR(500) NOT NULL,
    file_size INT NOT NULL,
    file_type VARCHAR(100) NOT NULL,
    mime_type VARCHAR(100),

    -- 이미지 정보
    width INT,
    height INT,
    thumbnail_url VARCHAR(500),

    -- 업로드 정보
    storage_provider VARCHAR(50) NOT NULL DEFAULT 'cloudflare',  -- cloudflare, s3, etc
    storage_path VARCHAR(500),

    -- 타임스탬프
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- 인덱스
CREATE INDEX idx_consultation_attachments_consultation ON consultation_attachments(consultation_id);
```

---

## 4. 콘텐츠 (Content)

### 4.1 posts (블로그)

```sql
CREATE TABLE posts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug VARCHAR(200) UNIQUE NOT NULL,

    -- 포스트 정보
    title VARCHAR(200) NOT NULL,
    content TEXT NOT NULL,                    -- Markdown 또는 HTML
    excerpt TEXT,

    -- 카테고리
    category VARCHAR(50),

    -- 미디어
    thumbnail_url VARCHAR(500),
    og_image_url VARCHAR(500),

    -- SEO
    meta_title VARCHAR(200),
    meta_description TEXT,
    meta_keywords VARCHAR(500),

    -- 게시 상태
    status VARCHAR(20) NOT NULL DEFAULT 'draft',  -- draft, published, archived
    published_at TIMESTAMP WITH TIME ZONE,

    -- 작성자
    author_id UUID NOT NULL REFERENCES users(id),

    -- 통계
    view_count INT NOT NULL DEFAULT 0,
    like_count INT NOT NULL DEFAULT 0,

    -- 타임스탬프
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP WITH TIME ZONE
);

-- 인덱스
CREATE INDEX idx_posts_slug ON posts(slug);
CREATE INDEX idx_posts_status ON posts(status, published_at DESC);
CREATE INDEX idx_posts_category ON posts(category);
CREATE UNIQUE INDEX idx_posts_unique_slug ON posts(slug) WHERE deleted_at IS NULL;
```

### 4.2 post_tags

```sql
CREATE TABLE post_tags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    post_id UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    tag_name VARCHAR(50) NOT NULL,

    UNIQUE(post_id, tag_name)
);

CREATE INDEX idx_post_tags_post ON post_tags(post_id);
CREATE INDEX idx_post_tags_name ON post_tags(tag_name);
```

### 4.3 videos (유튜브 영상)

```sql
CREATE TABLE videos (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- 유튜브 정보
    youtube_id VARCHAR(20) UNIQUE NOT NULL,    -- YouTube Video ID

    -- 영상 정보
    title VARCHAR(200) NOT NULL,
    description TEXT,

    -- 썸네일
    thumbnail_url VARCHAR(500),
    thumbnail_default_url VARCHAR(500),
    thumbnail_medium_url VARCHAR(500),
    thumbnail_high_url VARCHAR(500),

    -- 카테고리
    category VARCHAR(50) REFERENCES consultation_categories(id),

    -- 게시 상태
    status VARCHAR(20) NOT NULL DEFAULT 'draft',  -- draft, published, archived
    published_at TIMESTAMP WITH TIME ZONE,

    -- 정렬
    display_order INT NOT NULL DEFAULT 0,

    -- 통계
    view_count INT NOT NULL DEFAULT 0,        -- 사이트 내 조회수

    -- 타임스탬프
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP WITH TIME ZONE
);

-- 인덱스
CREATE INDEX idx_videos_youtube_id ON videos(youtube_id);
CREATE INDEX idx_videos_status ON videos(status, display_order);
CREATE INDEX idx_videos_category ON videos(category);
```

---

## 5. 파일 관리 (Files)

### 5.1 file_uploads

```sql
CREATE TABLE file_uploads (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- 파일 정보
    file_name VARCHAR(255) NOT NULL,
    file_url VARCHAR(500) NOT NULL,
    file_size INT NOT NULL,
    file_type VARCHAR(100) NOT NULL,
    mime_type VARCHAR(100),

    -- 이미지 정보
    width INT,
    height INT,
    thumbnail_url VARCHAR(500),

    -- 업로드 정보
    uploaded_by UUID REFERENCES users(id),
    upload_purpose VARCHAR(50),                -- consultation, post, profile, etc

    -- 저장소 정보
    storage_provider VARCHAR(50) NOT NULL DEFAULT 'cloudflare',
    storage_path VARCHAR(500),
    storage_bucket VARCHAR(100),

    -- 상태
    status VARCHAR(20) NOT NULL DEFAULT 'uploaded',  -- uploaded, confirmed, deleted
    expires_at TIMESTAMP WITH TIME ZONE,

    -- 타임스탬프
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    confirmed_at TIMESTAMP WITH TIME ZONE
);

-- 인덱스
CREATE INDEX idx_file_uploads_user ON file_uploads(uploaded_by);
CREATE INDEX idx_file_uploads_status ON file_uploads(status);
CREATE INDEX idx_file_uploads_expires ON file_uploads(expires_at) WHERE status = 'uploaded';
```

---

## 6. 관리자 (Admin)

### 6.1 admin_users

```sql
CREATE TABLE admin_users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(100) NOT NULL,

    -- 권한
    role VARCHAR(20) NOT NULL DEFAULT 'admin',  -- super_admin, admin, staff
    permissions JSONB,                           -- {"consultations": true, "users": false}

    -- 계정 상태
    is_active BOOLEAN NOT NULL DEFAULT TRUE,

    -- 로그인 정보
    last_login_at TIMESTAMP WITH TIME ZONE,
    last_login_ip VARCHAR(50),

    -- 타임스탬프
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- 인덱스
CREATE INDEX idx_admin_users_email ON admin_users(email);
CREATE INDEX idx_admin_users_role ON admin_users(role);
```

### 6.2 admin_logs

```sql
CREATE TABLE admin_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    admin_id UUID REFERENCES admin_users(id),

    -- 작업 정보
    action VARCHAR(100) NOT NULL,              -- consultation.create, user.delete 등
    target_type VARCHAR(50),                   -- consultation, user, post 등
    target_id UUID,

    -- 상세 정보
    details JSONB,

    -- IP 정보
    ip_address VARCHAR(50),
    user_agent TEXT,

    -- 타임스탬프
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- 인덱스
CREATE INDEX idx_admin_logs_admin ON admin_logs(admin_id);
CREATE INDEX idx_admin_logs_target ON admin_logs(target_type, target_id);
CREATE INDEX idx_admin_logs_created_at ON admin_logs(created_at DESC);
```

---

## 7. 방문자 통계 (Analytics)

### 7.1 page_views

```sql
CREATE TABLE page_views (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- 페이지 정보
    path VARCHAR(500) NOT NULL,
    referrer VARCHAR(500),

    -- 방문자 정보
    session_id VARCHAR(100),
    ip_address VARCHAR(50),
    user_agent TEXT,

    -- UTM 파라미터
    utm_source VARCHAR(100),
    utm_medium VARCHAR(100),
    utm_campaign VARCHAR(100),

    -- 타임스탬프
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- 파티션 (월 단위)
-- CREATE TABLE page_views_2024_03 PARTITION OF page_views
--     FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');

-- 인덱스
CREATE INDEX idx_page_views_path ON page_views(path);
CREATE INDEX idx_page_views_created_at ON page_views(created_at DESC);
CREATE INDEX idx_page_views_session ON page_views(session_id);
```

### 7.2 daily_stats (집계 테이블)

```sql
CREATE TABLE daily_stats (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stat_date DATE NOT NULL,

    -- 방문자
    page_views INT NOT NULL DEFAULT 0,
    unique_visitors INT NOT NULL DEFAULT 0,

    -- 상담
    new_consultations INT NOT NULL DEFAULT 0,
    answered_consultations INT NOT NULL DEFAULT 0,

    -- 회원
    new_users INT NOT NULL DEFAULT 0,

    -- 콘텐츠
    blog_views INT NOT NULL DEFAULT 0,
    video_views INT NOT NULL DEFAULT 0,

    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(stat_date)
);

CREATE INDEX idx_daily_stats_date ON daily_stats(stat_date DESC);
```

---

## 8. 설정 (Settings)

### 8.1 site_settings

```sql
CREATE TABLE site_settings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    key VARCHAR(100) UNIQUE NOT NULL,
    value TEXT NOT NULL,
    value_type VARCHAR(20) NOT NULL DEFAULT 'string',  -- string, json, boolean, number
    description TEXT,

    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- 초기 설정 데이터
INSERT INTO site_settings (key, value, value_type, description) VALUES
    ('site_name', 'OO성형외과', 'string', '사이트 이름'),
    ('site_description', '신뢰와 전문성을 갖춘 성형외과', 'string', '사이트 설명'),
    ('contact_phone', '02-1234-5678', 'string', '연락처'),
    ('contact_email', 'contact@example.com', 'string', '이메일'),
    ('business_hours', '{"weekday": "09:00-18:00", "saturday": "09:00-14:00", "sunday": "휴진"}', 'json', '진료 시간'),
    ('kakaoTalk_id', 'kakao_id', 'string', '카카오톡 상담 ID'),
    ('instagram_url', 'https://instagram.com/...', 'string', '인스타그램 URL'),
    ('youtube_channel_id', 'UC...', 'string', '유튜브 채널 ID'),
    ('privacy_policy_url', '/privacy', 'string', '개인정보처리방침 경로'),
    ('terms_of_service_url', '/terms', 'string', '이용약관 경로');
```

---

## 9. 마이그레이션 (Migration)

### 9.1 초기 마이그레이션 순서

```bash
# 1. 공통 테이블
users
refresh_tokens
admin_users
admin_logs

# 2. 상담 관련
consultation_categories
consultations
consultation_replies
consultation_attachments

# 3. 콘텐츠 관련
posts
post_tags
videos

# 4. 파일 관련
file_uploads

# 5. 통계 관련
page_views
daily_stats

# 6. 설정
site_settings
```

### 9.2 마이그레이션 명령어 (예시)

```bash
# 새 마이그레이션 생성
pnpm migrate:create create_users_table

# 마이그레이션 실행
pnpm migrate:up

# 마이그레이션 롤백
pnpm migrate:down
```

---

## 10. 백업 및 복구

### 10.1 백업 전략

| 타입 | 주기 | 보관 기간 | 위치 |
|-----|------|----------|------|
| 전체 백업 | 매일 새벽 2시 | 30일 | 국내 S3 호환 |
| 증분 백업 | 매시간 | 7일 | 국내 S3 호환 |
| 로그 백업 | 실시간 | 90일 | CloudFlare Log |

### 10.2 백업 스크립트

```bash
#!/bin/bash
# backup.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups"
DB_NAME="surgery_clinic"

# pg_dump로 백업
pg_dump -h $DB_HOST -U $DB_USER -d $DB_NAME \
    --format=custom \
    --file=$BACKUP_DIR/backup_$DATE.dump

# S3에 업로드
aws s3 cp $BACKUP_DIR/backup_$DATE.dump \
    s3://backup-bucket/db-backups/

# 30일 이상 된 백업 삭제
find $BACKUP_DIR -name "backup_*.dump" -mtime +30 -delete
```
