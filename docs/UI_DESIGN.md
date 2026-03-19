# UI/UX 디자인 가이드

## 1. 디자인 원칙

### 1.1 핵심 가치

- **신뢰감**: 의료 서비스 특성상 전문성과 신뢰감 전달
- **접근성**: 다양한 연령층과 국적 사용자 고려
- **명확성**: 복잡한 의료 정보를 알기 쉽게 전달
- **프라이버시**: 개인 정보 보호 강조

### 1.2 타겟 사용자

| 그룹 | 특징 | 고려사항 |
|-----|------|----------|
| 20대 후반~30대 | 시술 관심 높음, 모바일 중심 | 트렌디한 디자인, 모바일 최적화 |
| 40대~50대 | 재정 여유 있음, PC/모바일 혼용 | 가독성, 명확한 안내 |
| 중국/일본인 | 의료 관광객 | 다국어 지원, 간편한 상담 |

---

## 2. 디자인 시스템

### 2.1 색상

```css
/* 주 색상 - 신뢰감 있는 블루 계열 */
--primary-50: #eff6ff;
--primary-100: #dbeafe;
--primary-500: #3b82f6;
--primary-600: #2563eb;
--primary-700: #1d4ed8;

/* 보조 색상 - 부드러운 핑크/그레이 */
--secondary-50: #fdf2f8;
--secondary-100: #fce7f3;
--secondary-500: #ec4899;
--secondary-600: #db2777;

/* 중립 색상 */
--gray-50: #f9fafb;
--gray-100: #f3f4f6;
--gray-200: #e5e7eb;
--gray-500: #6b7280;
--gray-700: #374151;
--gray-900: #111827;

/* 상태 색상 */
--success: #10b981;
--warning: #f59e0b;
--error: #ef4444;
--info: #3b82f6;
```

### 2.2 타이포그래피

```css
/* 본문 */
--font-sans: 'Pretendard', -apple-system, system-ui, sans-serif;
--font-size-sm: 0.875rem;
--font-size-base: 1rem;
--font-size-lg: 1.125rem;
--font-size-xl: 1.25rem;

/* 제목 */
--font-size-2xl: 1.5rem;
--font-size-3xl: 1.875rem;
--font-size-4xl: 2.25rem;

/* 줄간격 */
--leading-tight: 1.25;
--leading-normal: 1.5;
--leading-relaxed: 1.75;
```

### 2.3 간격

```css
--spacing-1: 0.25rem;
--spacing-2: 0.5rem;
--spacing-4: 1rem;
--spacing-6: 1.5rem;
--spacing-8: 2rem;
--spacing-12: 3rem;
--spacing-16: 4rem;
```

### 2.4 둥근 모서리

```css
--radius-sm: 0.25rem;
--radius-md: 0.375rem;
--radius-lg: 0.5rem;
--radius-xl: 0.75rem;
--radius-2xl: 1rem;
--radius-full: 9999px;
```

---

## 3. 페이지 레이아웃

### 3.1 헤더

```
┌─────────────────────────────────────────────────────────────┐
│  [로고]  진료과목  의료진  블로그  미디어        [언어] [상담] │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 메인 페이지

```
┌─────────────────────────────────────────────────────────────┐
│                        헤더                                   │
├─────────────────────────────────────────────────────────────┤
│  [Hero Section]                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  흉터 없는 눈 성형                                    │   │
│  │  자연스러운 결과를 약속합니다                          │   │
│  │  [상담 신청]                                          │   │
│  └─────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│  [주요 시술]                                                 │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐                    │
│  │ 눈   │ │  코  │ │ 윤곽 │ │ 가슴 │                    │
│  └──────┘ └──────┘ └──────┘ └──────┘                    │
├─────────────────────────────────────────────────────────────┤
│  [의료진 소개]                                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  [의사 사진]  김의사 원장                              │   │
│  │              20년 경력의 눈/코 전문의                  │   │
│  └─────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│  [최신 블로그]                                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  쌍꺼풀 수술 가이드                                   │   │
│  │  쌍꺼풀 수술을 고민하고 계신다면...                    │   │
│  └─────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│                        푸터                                   │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 블로그 목록

```
┌─────────────────────────────────────────────────────────────┐
│  블로그 > 2024년                                             │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐   │
│  │ [썸네일]  쌍꺼풀 수술 완벽 가이드                       │   │
│  │          2024.03.15  |  조회 1,234                     │   │
│  │          쌍꺼풀 수술을 고민하고 계신다면...              │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ [썸네일]  코 성형 전 후 비교                           │   │
│  │          2024.03.10  |  조회 987                       │   │
│  │          코 성형 결과를 미리 확인해보세요...             │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  [< 1 2 3 4 5 >]                                            │
└─────────────────────────────────────────────────────────────┘
```

### 3.4 상담 게시판

```
┌─────────────────────────────────────────────────────────────┐
│  상담 게시판                                                 │
├─────────────────────────────────────────────────────────────┤
│  [카테고리 ▼]  [검색]                                       │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐   │
│  │  [공개] 쌍꺼풀 수술 비용 문의드려요                    │   │
│  │         눈성형 | 궁금이 | 2024.03.15                 │   │
│  │         답변 완료                                      │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  [공개] 재수술 가능한가요?                            │   │
│  │         눈성형 | 익명 | 2024.03.14                   │   │
│  │         답변 대기                                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  [+] 새 상담 작성                                            │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. 컴포넌트 디자인

### 4.1 버튼

```svelte
<!-- Primary Button -->
<button class="btn btn-primary">
  상담 신청
</button>

<!-- Secondary Button -->
<button class="btn btn-secondary">
  더 알아보기
</button>

<!-- Outline Button -->
<button class="btn btn-outline">
  목록으로
</button>

<!-- Icon Button -->
<button class="btn btn-icon">
  <SearchIcon />
</button>
```

### 4.2 카드

```svelte
<div class="card">
  <img src="thumbnail.jpg" alt="썸네일" class="card-image" />
  <div class="card-content">
    <span class="card-category">눈 성형</span>
    <h3 class="card-title">쌍꺼풀 수술 가이드</h3>
    <p class="card-excerpt">쌍꺼풀 수술을 고민하고 계신다면...</p>
    <div class="card-meta">
      <span>2024.03.15</span>
      <span>조회 1,234</span>
    </div>
  </div>
</div>
```

### 4.3 폼

```svelte
<form class="form">
  <div class="form-group">
    <label for="email">이메일</label>
    <input
      id="email"
      type="email"
      placeholder="example@email.com"
      class="form-input"
    />
    <span class="form-error">올바른 이메일을 입력해주세요</span>
  </div>

  <div class="form-group">
    <label for="category">진료과목</label>
    <select id="category" class="form-select">
      <option value="">선택해주세요</option>
      <option value="eye">눈 성형</option>
      <option value="nose">코 성형</option>
    </select>
  </div>

  <div class="form-group">
    <label for="content">내용</label>
    <textarea
      id="content"
      rows="5"
      class="form-textarea"
      placeholder="상담 내용을 입력해주세요"
    ></textarea>
  </div>

  <div class="form-actions">
    <button type="submit" class="btn btn-primary">제출하기</button>
  </div>
</form>
```

### 4.4 탭

```svelte
<div class="tabs">
  <button class="tab tab-active">공개 상담</button>
  <button class="tab">내 상담</button>
</div>

<div class="tab-content">
  <!-- 탭 내용 -->
</div>
```

---

## 5. 반응형 디자인

### 5.1 브레이크포인트

```css
/* Mobile First */
/* 기본: 모바일 스타일 */

@media (min-width: 640px) {
  /* sm: 640px */
}

@media (min-width: 768px) {
  /* md: 768px - 태블릿 */
}

@media (min-width: 1024px) {
  /* lg: 1024px - 데스크톱 */
}

@media (min-width: 1280px) {
  /* xl: 1280px - 대형 데스크톱 */
}
```

### 5.2 모바일 네비게이션

```svelte
<!-- 모바일 햄버거 메뉴 -->
<button class="mobile-menu-btn" on:click={toggleMenu}>
  <MenuIcon />
</button>

<div class="mobile-menu" class:open={menuOpen}>
  <nav class="mobile-nav">
    <a href="/procedures">진료과목</a>
    <a href="/doctors">의료진 소개</a>
    <a href="/blog">블로그</a>
    <a href="/media">미디어</a>
    <hr />
    <a href="/consult">상담 신청</a>
    <a href="/my">마이페이지</a>
  </nav>
</div>
```

---

## 6. 접근성

### 6.1 WAI-ARIA

```svelte
<!-- 모달 -->
<div
  role="dialog"
  aria-modal="true"
  aria-labelledby="modal-title"
  aria-describedby="modal-description"
>
  <h2 id="modal-title">상담 신청</h2>
  <p id="modal-description">상담 내용을 입력해주세요</p>
  <!-- ... -->
</div>

<!-- 로딩 상태 -->
<div role="status" aria-live="polite">
  <span class="sr-only">로딩 중...</span>
  <LoadingSpinner />
</div>
```

### 6.2 키보드 네비게이션

```css
/* 포커스 스타일 */
:focus-visible {
  outline: 2px solid var(--primary-500);
  outline-offset: 2px;
}

/* 스킵 링크 */
.skip-link {
  position: absolute;
  top: -40px;
  left: 0;
  padding: 8px;
  background: var(--primary-500);
  color: white;
  z-index: 100;
}

.skip-link:focus {
  top: 0;
}
```

---

## 7. 애니메이션

### 7.1 원칙

- 자연스럽고 부드러운 전환
- 사용자 작업에 대한 즉각적인 피드백
- 과도한 애니메이션 자제 (의료 사이트 특성)

### 7.2 예시

```css
/* 페이드인 */
@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

/* 슬라이드업 */
@keyframes slideUp {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* 버튼 호버 */
.btn-primary:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(59, 130, 246, 0.4);
}

.btn-primary:active {
  transform: translateY(0);
}
```

---

## 8. 다크 모드 (선택)

사용자가 다크 모드를 선호할 수 있으므로 지원 고려:

```css
@media (prefers-color-scheme: dark) {
  :root {
    --gray-50: #1f2937;
    --gray-100: #374151;
    --gray-900: #f9fafb;
  }
}
```
