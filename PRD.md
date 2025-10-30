# MOOKA PRD (Product Requirements Document)

> **플랫폼 이름**: MOOKA (무카)
> **버전**: MVP 1.0
> **목표**: CivitAI의 오픈 공유 대신 **크리에이터 90% 수익**으로 프리미엄 콘텐츠 시장 선점

---

## 1. 제품 개요

| 항목 | 내용 |
|------|------|
| **비전** | 크리에이터가 수익의 **90%**를 가져가며 OnlyFans처럼 팬덤형 AI 콘텐츠 시장 구축 |
| **핵심 가설** | 수익 대부분 → 크리에이터 공격적 마케팅 + 고품질 콘텐츠 생산 → 소비자 유료 전환 |
| **타겟 사용자** | **크리에이터** (AI 모델/LoRA/프롬프트 제작자) + **소비자** (AI 생성 사용자) |
| **차별점** | CivitAI: 무료 중심 + 수익 일부<br>**MOOKA: 유료 중심 + 크리에이터 90% 수익** |

---

## 2. 수익 모델 (MVP)

| 항목 | 정책 |
|------|------|
| **구매 방식** | **One-time 구매** (크리에이터가 가격 설정) |
| **가격 범위** | **최소 $0.99 / 최대 $100** (MVP 상한선) |
| **수익 배분** | **크리에이터 90% / 플랫폼 10%** (Stripe 수수료 별도) |
| **무료 vs 유료** | 크리에이터가 콘텐츠별로 선택 |
| **티저 정책** | - 무료: **전체 열람 가능**<br>- 유료: **최대 3장 이미지 미리보기** (크리에이터 재량) |

> **Stripe Connect는 추후 연동 가능** → MVP에서는 **플랫폼이 일괄 정산 후 수동 이체** (Workers Cron Job)

---

## 3. 기능 요구사항 (MVP)

| 기능 | 핵심 로직 |
|------|----------|
| **콘텐츠 형식** | CivitAI + Promptbase **완전 복제**<br>→ .safetensors, LoRA, 프롬프트, 샘플 이미지, 태그, 버전 히스토리 |
| **업로드** | 드래그앤드롭 → R2 업로드 → D1에 메타 저장 → `is_paid`, `price`, `preview_images[]` 설정 |
| **검색/정렬** | - **최신순** (기본)<br>- **추천순**: `score = 구매수 × 가격` (D1 쿼리에서 계산) |
| **RSS 피드** | `/feed?sort=latest\|recommended&tag=xxx` → Workers에서 XML 생성 + CDN 캐시 |
| **페이먼트월** | 구매 전 → 402 + Stripe Checkout 세션 생성 |
| **결제 흐름** | 1. 소비자 결제 → 2. Webhook → 3. D1 `purchases` 저장 → 4. 다운로드 URL 발급 |
| **다운로드** | 구매 성공 → R2 Presigned URL (1회성, 5분 유효) |
| **수익 배분** | Webhook → D1 `payouts` 테이블 생성 → **주간 Cron Job으로 플랫폼 계좌에서 크리에이터 이체** |
| **관리자 대시보드** | 신고 처리 큐, 콘텐츠 삭제/숨김, 사용자 차단 |

---

## 4. 사용자 역할 & 인증

| 역할 | 핵심 행동 |
|------|----------|
| **크리에이터** | 업로드 → 가격/무료 설정 → 수익 내역 확인 |
| **소비자** | 검색 → 티저 보기 → 결제 → 다운로드 |
| **관리자** | 콘텐츠 moderation (신고 3회 → 자동 숨김) |

### 인증 시스템

- **Clerk** (Email + Social 옵션)
- JWT 검증 → Workers에서 `request.headers.get('authorization')`
- **18+ 인증**: 미국(Ondato), 한국(휴대폰) → **한국 문제 시 IP 차단**

---

## 5. 기술 아키텍처 (Cloudflare 중심)

```text
[React + shadcn + Tailwind + TS] ←→ [Cloudflare Pages]
        ↓ (API 호출)
[Cloudflare Workers] ←→ [D1 (메타, 구매, payout)]
        ↓              ↓
[R2 (파일 저장)]   [Stripe Webhook]
        ↓
[Cron Job → 수동 이체]
```

### 핵심 API 엔드포인트 (Workers)

| 메서드 | 경로 | 설명 |
|--------|------|------|
| POST | `/api/upload` | 파일 → R2, 메타 → D1 |
| GET | `/api/content/:id` | 페이월 체크 → 티저 or 402 |
| GET | `/api/search` | 정렬 + 필터 |
| POST | `/api/purchase` | Stripe 세션 생성 |
| POST | `/webhook/stripe` | 결제 성공 → 다운로드 권한 부여 |
| GET | `/feed` | RSS XML |
| GET | `/admin/reports` | 관리자 전용 신고 큐 |

---

## 6. 로케일 & 지역 제한

| 항목 | 정책 |
|------|------|
| **지원 언어** | 영어 + 한국어 (i18n with next-intl or react-i18next) |
| **한국 사용자** | 휴대폰 본인인증 실패 시 → IP 차단 (Workers Geo-block) |
| **NSFW 정책** | 업로드 시 AI 태그 → 80% 이상 → 관리자 리뷰 큐 |

---

## 7. MVP 배포 체크리스트

| 완료 여부 | 항목 |
|-----------|------|
| ☐ | Cloudflare Pages + Workers 배포 |
| ☐ | D1 스키마: users, contents, purchases, payouts, reports |
| ☐ | R2 버킷: mooka-content (Public off) |
| ☐ | Clerk + Stripe 테스트 계정 연동 |
| ☐ | 관리자 대시보드 (신고 처리 UI) |
| ☐ | 한국어/영어 로케일 적용 |
| ☐ | 한국 IP 차단 로직 (Workers cf.country) |

---

## 8. 데이터베이스 스키마 (D1)

### users
```sql
CREATE TABLE users (
  id TEXT PRIMARY KEY,
  clerk_id TEXT UNIQUE NOT NULL,
  email TEXT NOT NULL,
  role TEXT DEFAULT 'consumer', -- consumer | creator | admin
  created_at INTEGER DEFAULT (unixepoch())
);
```

### contents
```sql
CREATE TABLE contents (
  id TEXT PRIMARY KEY,
  creator_id TEXT NOT NULL,
  title TEXT NOT NULL,
  description TEXT,
  file_url TEXT NOT NULL, -- R2 URL
  is_paid INTEGER DEFAULT 1, -- 0: free, 1: paid
  price REAL DEFAULT 0.99,
  preview_images TEXT, -- JSON array
  tags TEXT, -- JSON array
  downloads INTEGER DEFAULT 0,
  created_at INTEGER DEFAULT (unixepoch()),
  FOREIGN KEY (creator_id) REFERENCES users(id)
);
```

### purchases
```sql
CREATE TABLE purchases (
  id TEXT PRIMARY KEY,
  content_id TEXT NOT NULL,
  buyer_id TEXT NOT NULL,
  amount REAL NOT NULL,
  stripe_session_id TEXT,
  purchased_at INTEGER DEFAULT (unixepoch()),
  FOREIGN KEY (content_id) REFERENCES contents(id),
  FOREIGN KEY (buyer_id) REFERENCES users(id)
);
```

### payouts
```sql
CREATE TABLE payouts (
  id TEXT PRIMARY KEY,
  creator_id TEXT NOT NULL,
  amount REAL NOT NULL,
  status TEXT DEFAULT 'pending', -- pending | completed | failed
  created_at INTEGER DEFAULT (unixepoch()),
  FOREIGN KEY (creator_id) REFERENCES users(id)
);
```

### reports
```sql
CREATE TABLE reports (
  id TEXT PRIMARY KEY,
  content_id TEXT NOT NULL,
  reporter_id TEXT NOT NULL,
  reason TEXT,
  status TEXT DEFAULT 'pending', -- pending | reviewed | resolved
  created_at INTEGER DEFAULT (unixepoch()),
  FOREIGN KEY (content_id) REFERENCES contents(id),
  FOREIGN KEY (reporter_id) REFERENCES users(id)
);
```

---

## 9. 향후 확장 계획 (Post-MVP)

- **구독 모델**: 크리에이터별 월간 구독 옵션
- **Stripe Connect**: 자동 수익 분배
- **AI 태깅**: 자동 NSFW 탐지 및 카테고리 분류
- **소셜 기능**: 팔로우, 좋아요, 댓글
- **프리미엄 티어**: 플랫폼 수수료 5%로 감소 (연간 구독)
- **모바일 앱**: React Native 또는 PWA
