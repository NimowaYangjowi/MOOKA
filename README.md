# MOOKA (무카)

> AI 콘텐츠 프리미엄 마켓플레이스 - 크리에이터가 수익의 **90%**를 가져갑니다

## 프로젝트 개요

MOOKA는 AI 생성 모델(.safetensors), LoRA, 프롬프트를 거래하는 프리미엄 마켓플레이스입니다.
CivitAI의 무료 공유 모델과 달리, **유료 콘텐츠 중심**으로 크리에이터의 수익을 극대화합니다.

### 핵심 차별점
- **크리에이터 90% 수익 배분** (vs CivitAI 일부 수익)
- **유료 콘텐츠 중심** 마켓플레이스
- **OnlyFans 스타일** 팬덤형 AI 콘텐츠 시장

---

## 주요 기능 (MVP 1.0)

### 크리에이터
- 콘텐츠 업로드 (.safetensors, LoRA, 프롬프트, 샘플 이미지)
- 가격 설정 ($0.99 ~ $100)
- 무료/유료 선택 (유료는 최대 3장 티저 이미지)
- 수익 내역 확인 (주간 정산)

### 소비자
- 검색 및 정렬 (최신순, 추천순)
- 티저 미리보기
- Stripe 결제 (One-time 구매)
- 구매 후 다운로드 (R2 Presigned URL, 5분 유효)

### 관리자
- 신고 처리 큐 (신고 3회 → 자동 숨김)
- 콘텐츠 삭제/숨김
- 사용자 차단

---

## 기술 스택

### Frontend
- **React 19** + **TypeScript 5.7+**
- **Tailwind CSS 4** + **shadcn/ui**
- **TanStack Query v5** (낙관적 업데이트, 캐싱)
- **Cloudflare Pages** 배포

### Backend
- **Cloudflare Workers** (API + Webhook)
- **D1** (SQLite 데이터베이스)
- **R2** (파일 스토리지)
- **Cron Jobs** (주간 수익 정산)

### 외부 서비스
- **Clerk** (인증)
- **Stripe** (결제)
- **i18n** (영어/한국어)

---

## 프로젝트 구조

```
MOOKA/
├── src/
│   ├── components/       # UI 컴포넌트
│   ├── lib/              # 비즈니스 로직
│   └── workers/          # Cloudflare Workers API
├── migrations/           # D1 데이터베이스 마이그레이션
├── docs/                 # 문서
├── task_history/         # 작업 내역
├── PRD.md                # Product Requirements Document
├── CLAUDE.md             # AI 개발 가이드
└── README.md             # 이 파일
```

---

## 개발 환경 설정

### 요구사항
- Node.js 20+
- Wrangler CLI
- Cloudflare 계정

### 설치 및 실행

```bash
# 의존성 설치
npm install

# Frontend 개발 서버
npm run dev

# Workers 로컬 실행
wrangler dev

# D1 마이그레이션 (로컬)
wrangler d1 migrations apply mooka-db --local
```

### 환경 변수 설정

`.env` 파일 생성:
```env
VITE_CLERK_PUBLISHABLE_KEY=pk_test_...
STRIPE_PUBLISHABLE_KEY=pk_test_...
```

> ⚠️ **주의**: `.env` 파일은 절대 Git에 커밋하지 마세요.

---

## 배포

```bash
# Frontend 빌드 및 배포
npm run build
wrangler pages deploy dist

# Workers 배포
wrangler deploy

# D1 마이그레이션 (프로덕션)
wrangler d1 migrations apply mooka-db
```

---

## 개발 규칙

### 코드 품질
- 파일 길이: 200줄 초과 시 모듈화, **600줄 절대 금지**
- TypeScript strict mode 준수
- 파일 최상단에 목적/역할 주석 필수

### 문서화
- DB 스키마 변경 시 `/docs/DATABASE_SCHEMA.md` 업데이트
- 새 폴더 생성 시 `{폴더명}/CLAUDE.md` 필수 작성
- 작업 완료 후 `task_history/` 내역 작성

### 보안
- `.env`, `*.key` 파일 절대 커밋 금지
- R2 Presigned URL: 1회성, 5분 유효
- 관리자 API: JWT role 검증 필수

---

## 문서

- [PRD.md](./PRD.md) - 제품 요구사항 문서
- [CLAUDE.md](./CLAUDE.md) - AI 개발 가이드 (파일 구조, 금지 사항 등)

---

## 라이선스

All rights reserved. (라이선스 정책은 추후 결정)

---

## 문의

프로젝트 관련 문의는 이슈 트래커를 이용해주세요.
