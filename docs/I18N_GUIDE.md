# 다국어 지원 가이드

## 1. 개요

성형외과 웹사이트는 한국어, 영어, 중국어, 일본어를 지원합니다. 특히 중국, 일본에서의 의료 관광객을 고려하여 다국어 지원이 필수적입니다.

---

## 2. 지원 언어

| 언어 | 코드 | locale | 타겟 시장 |
|-----|------|--------|----------|
| 한국어 | ko | ko-KR | 국내 내원 환자 |
| 영어 | en | en-US | 외국인 거주자, 일반 |
| 중국어 | zh | zh-CN | 중국 의료 관광객 |
| 일본어 | ja | ja-JP | 일본 의료 관광객 |

---

## 3. 기술 구현

### 3.1 라이브러리

- **svelte-i18n**: SvelteKit에서 다국어 지원
- **ICU MessageFormat**: 복수형, 날짜/시간 형식화

```bash
pnpm add svelte-i18n
```

### 3.2 구조

```
src/lib/i18n/
├── index.ts              # i18n 설정
├── locales/
│   ├── ko.json          # 한국어
│   ├── en.json          # 영어
│   ├── zh.json          # 중국어 (간체)
│   └── ja.json          # 일본어
└── types.ts             # 타입 정의
```

### 3.3 기본 설정

```typescript
// src/lib/i18n/index.ts
import { init, register } from 'svelte-i18n';

import ko from './locales/ko.json';
import en from './locales/en.json';
import zh from './locales/zh.json';
import ja from './locales/ja.json';

register('ko', () => Promise.resolve(ko));
register('en', () => Promise.resolve(en));
register('zh', () => Promise.resolve(zh));
register('ja', () => Promise.resolve(ja));

init({
  fallbackLocale: 'ko',
  initialLocale: 'ko'
});
```

---

## 4. URL 구조

### 4.1 경로별 언어 지원

| 페이지 | 한국어 | 영어 | 중국어 | 일본어 |
|-------|--------|------|--------|--------|
| 메인 | `/` | `/en` | `/zh` | `/ja` |
| 블로그 목록 | `/blog` | `/en/blog` | `/zh/blog` | `/ja/blog` |
| 블로그 상세 | `/blog/2024/slug` | `/en/blog/2024/slug` | `/zh/blog/2024/slug` | `/ja/blog/2024/slug` |
| 진료과목 | `/procedures` | `/en/procedures` | `/zh/procedures` | `/ja/procedures` |

### 4.2 언어 감지 우선순위

1. URL 경로 (`/en/...`)
2. 쿠키 저장된 언어 설정
3. Accept-Language 헤더
4. 기본값 (한국어)

---

## 5. 번역 리소스 구조

### 5.1 JSON 구조

```json
// ko.json
{
  "common": {
    "loading": "로딩 중...",
    "save": "저장",
    "cancel": "취소",
    "delete": "삭제",
    "edit": "편집"
  },
  "nav": {
    "home": "홈",
    "blog": "블로그",
    "procedures": "진료과목",
    "doctors": "의료진 소개",
    "consult": "상담 신청",
    "myPage": "마이페이지"
  },
  "blog": {
    "title": "블로그",
    "readMore": "더 읽기",
    "publishedAt": "발행일",
    "category": "카테고리"
  },
  "procedures": {
    "eye": {
      "title": "눈 성형",
      "description": "쌍꺼풀, 눈매교정 등"
    },
    "nose": {
      "title": "코 성형",
      "description": "코끝, 코등, 비중격 교정 등"
    }
  }
}
```

### 5.2 Svelte 컴포넌트 사용

```svelte
<script>
  import { _, locale } from 'svelte-i18n';
</script>

<h1>{$_('blog.title')}</h1>
<p>{$locale}</p>

<!-- 복수형 -->
<p>{$_('items.count', { count: items.length })}</p>
```

---

## 6. 콘텐츠 번역 관리

### 6.1 DB 스키마

```sql
-- 방법 1: 별도 테이블
posts (
  id,
  slug,
  created_at,
  updated_at
)

post_translations (
  id,
  post_id (FK),
  language,
  title,
  content,
  excerpt,
  meta_title,
  meta_description
)

-- 방법 2: JSONB 필드
posts (
  id,
  slug,
  translations JSONB
  -- {
  --   "ko": {"title": "...", "content": "..."},
  --   "en": {"title": "...", "content": "..."}
  -- }
)
```

### 6.2 관리자 UI

- 언어 탭으로 전환하여 번역 입력
- 번역 누락 표시
- 번역 상태 (완료/진행중/미번역)

---

## 7. SEO 최적화

### 7.1 hreflang 태그

```svelte
<!-- src/routes/+layout.svelte -->
<svelte:head>
  <link rel="alternate" hreflang="ko" href="https://example.com/blog/2024/slug" />
  <link rel="alternate" hreflang="en" href="https://example.com/en/blog/2024/slug" />
  <link rel="alternate" hreflang="zh" href="https://example.com/zh/blog/2024/slug" />
  <link rel="alternate" hreflang="ja" href="https://example.com/ja/blog/2024/slug" />
  <link rel="alternate" hreflang="x-default" href="https://example.com/blog/2024/slug" />
</svelte:head>
```

### 7.2 메타 태그 번역

```typescript
// 각 언어별 메타 태그
export const load = ({ url }) => {
  const locale = getLocaleFromUrl(url);

  return {
    meta: {
      title: translations[locale].meta.title,
      description: translations[locale].meta.description,
      ogTitle: translations[locale].meta.ogTitle,
      ogDescription: translations[locale].meta.ogDescription
    }
  };
};
```

---

## 8. 언어 전환 UI

### 8.1 언어 선택기

```svelte
<script>
  import { locale } from 'svelte-i18n';
  import { goto } from '$app/navigation';

  const languages = [
    { code: 'ko', name: '한국어', flag: '🇰🇷' },
    { code: 'en', name: 'English', flag: '🇺🇸' },
    { code: 'zh', name: '中文', flag: '🇨🇳' },
    { code: 'ja', name: '日本語', flag: '🇯🇵' }
  ];

  function changeLanguage(lang: string) {
    $locale = lang;
    document.cookie = `locale=${lang};path=/;max-age=31536000`;

    // URL 경로 변경
    const currentPath = window.location.pathname;
    const newPath = currentPath.replace(/^\/[a-z]{2}/, `/${lang === 'ko' ? '' : lang}`);
    goto(newPath);
  }
</script>

<select bind:value={$locale} on:change={(e) => changeLanguage(e.target.value)}>
  {#each languages as lang}
    <option value={lang.code}>{lang.flag} {lang.name}</option>
  {/each}
</select>
```

---

## 9. 날짜/시간 형식

### 9.1 ICU MessageFormat

```json
// ko.json
{
  "datetime": {
    "short": "{date, date, short}",
    "long": "{date, date, full}",
    "time": "{date, time, short}"
  }
}
```

```svelte
<script>
  import { _, formatDate } from 'svelte-i18n';

  const now = new Date();
</script>

<p>{$_('datetime.short', { values: { date: now } })}</p>
<!-- 한국어: 2024. 3. 19. -->
<!-- 영어: 3/19/24 -->
<!-- 중국어: 2024/3/19 -->
<!-- 일본어: 2024/03/19 -->
```

---

## 10. 번역 가이드라인

### 10.1 의료 용어 번역

| 한국어 | 영어 | 중국어 | 일본어 |
|-------|------|--------|--------|
| 쌍꺼풀 수술 | Double Eyelid Surgery | 双眼皮手术 | 二重整形 |
| 코 성형 | Rhinoplasty | 隆鼻术 | 鼻整形 |
| 지방 흡입 | Liposuction | 吸脂术 | 脂肪吸引 |
| 보톡스 | Botox | 肉毒杆菌 | ボトックス |
| 필러 | Filler | 玻尿酸填充 | ヒアルロン酸 |
| 리프팅 | Lifting | 拉皮手术 | リフトアップ |

### 10.2 번역 주의사항

1. **의료 용어 정확성**: 전문 의료 용어 사용
2. **문화적 차이 고려**: 각국 문화에 맞는 표현
3. **법적 표현 준수**: 의료광고 심의기준 반영
4. **브랜드명 통일**: 병원명/의료진명 표기 통일

---

## 11. 번역 외주 관리

### 11.1 번역 프로세스

1. 원본 작성 (한국어)
2. 번역 전송 (업체 또는 내부)
3. 번역 검수
4. CMS에 등록
5. 정적 빌드 트리거

### 11.2 번역 업체 선정 시 고려사항

- 의료 전문 번역 가능 여부
- 4개 언어 모두 지원 여부
- 검수/수정 프로세스
- 가격과 리드타임
