# 성형외과 웹사이트

성형외과의원을 위한 종합 웹사이트 프로젝트입니다. 병원 홍보, 의학 정보 게시, 고객 상담 기능을 제공합니다.

## 프로젝트 개요

- **목적**: 성형외과의원 홈페이지 및 상담 시스템
- **기술 스택**: SvelteKit (프론트), Ranvier (백엔드), Cloudflare (배포)
- **특징**: 정적 사이트 생성, 다국어 지원, CRM 기능

## 문서

| 문서 | 설명 |
|-----|------|
| [PROJECT_PLAN.md](./docs/PROJECT_PLAN.md) | 프로젝트 개발 계획 |
| [API_DESIGN.md](./docs/API_DESIGN.md) | API 설계 |
| [DATABASE_SCHEMA.md](./docs/DATABASE_SCHEMA.md) | 데이터베이스 스키마 |
| [UI_DESIGN.md](./docs/UI_DESIGN.md) | UI/UX 가이드 |
| [I18N_GUIDE.md](./docs/I18N_GUIDE.md) | 다국어 지원 가이드 |
| [AUTH_2FA_GUIDE.md](./docs/AUTH_2FA_GUIDE.md) | 2단계 인증 가이드 |
| [MEDICAL_COMPLIANCE.md](./docs/MEDICAL_COMPLIANCE.md) | 의료 법규 준수 가이드 |
| [CRM_PLAN.md](./docs/CRM_PLAN.md) | 병원 CRM 기획 |

## 주요 기능

- **홍보 콘텐츠**: 블로그, 유튜브 영상 관리
- **상담 게시판**: 공개/비밀 상담, 간편가입
- **다국어**: 한국어, 영어, 중국어, 일본어
- **정적 빌드**: DB 편집 후 자동 정적 파일 생성
- **2FA**: TOTP/이메일 OTP 지원
- **CRM**: 고객 관리, 예약 관리, 통계

## 개발 환경

```bash
# 패키지 설치
pnpm install

# 프론트엔드 개발
cd packages/frontend
pnpm dev

# 백엔드 개발
cd packages/backend
pnpm dev

# 정적 빌드 Worker
cd packages/static-worker
pnpm dev
```

## 라이선스

내부용 프로젝트입니다.
