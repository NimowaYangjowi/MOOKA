# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## 프로젝트 개요

**MOOKA**: AI 콘텐츠(모델/LoRA/프롬프트) 프리미엄 마켓플레이스
- 크리에이터 90% 수익 배분
- Cloudflare 풀스택: Pages + Workers + D1 + R2
- PRD 상세: [PRD.md](./PRD.md)

---

## 기본 원칙

### 언어 및 작업 방식
- **한국어** 입출력
- 단계적 계획 및 접근
- 파일 최상단에 **목적/역할 주석** 필수
- SOLID 원칍 준수

### 개발 철학
- **MVP 우선**: 최소 스펙으로 구현, 과도한 추상화 지양
- **계층 분리**: UI / Business Logic / Data Access 명확히 분리
- **모듈화**: 역할과 책임 명확히 정의, 의존성 문서화

### 코드 구조 원칙
- **파일 길이 제한**:
  - 200줄 초과 시: 기능별로 모듈화 (utils, hooks, types 분리)
  - 600줄 절대 상한선: 초과 시 강제 리팩토링
  - 예: `ContentUpload.tsx` (250줄) → `ContentUpload.tsx` (150줄) + `hooks/useContentUpload.ts` (100줄)

- **컴포넌트 재사용성**:
  - **집중화**: UI 컴포넌트는 `src/components/ui/` 중앙 관리
  - **합성 패턴**: 작은 컴포넌트 조합으로 복잡한 UI 구성
  - **Props 인터페이스**: 명확한 타입 정의로 재사용성 보장
  - 예: `Button`, `Card`, `Modal` → shadcn/ui 기반 확장

### 문서화
- **DB 스키마 변경**: `/docs/DATABASE_SCHEMA.md` 업데이트 (존재 시)
- **작업 내역**: `task_history/{yyyy_mm_dd_{numbering}_파일명}.md` 작성
- **폴더별 문서화**: 새 폴더 생성 시 `{폴더명}/CLAUDE.md` 필수 작성
  - 폴더 목적 및 역할
  - 주요 파일 설명 및 책임
  - 의존성 관계
  - 사용 예시 (필요 시)
- **폴더 문서 동기화**: 폴더 내 파일 수정/추가/삭제 시 해당 폴더의 `CLAUDE.md` 즉시 업데이트

---

## 기술 스택

### Frontend
- **React 19** + **TypeScript 5.7+** + **Tailwind CSS 4** + **shadcn/ui**
- **TanStack Query v5** (React Query): 서버 상태 관리 및 캐싱
- **Cloudflare Pages** 배포
- **Vite** 또는 **Next.js 15+** (Pages Router 선호)

### Backend
- **Cloudflare Workers**: API 엔드포인트, Webhook 처리
- **D1**: SQLite 기반 데이터베이스 (users, contents, purchases, payouts, reports)
- **R2**: 파일 스토리지 (.safetensors, 이미지 등)
- **Cron Jobs**: 주간 수익 정산

### 외부 서비스
- **Clerk**: 인증 (JWT 기반)
- **Stripe**: 결제 및 Webhook
- **i18n**: 영어/한국어 지원 (react-i18next 또는 next-intl)

### 라이브러리 버전 정책
- **항상 최신 안정 버전** 사용 (breaking changes 주의)
- 주요 의존성 업데이트 시 `package.json` 및 문서 갱신
- 보안 패치는 즉시 적용

---

## 아키텍처

### 계층 구조

```
src/
├── components/          # Presentation Layer (UI)
│   └── **/*.tsx
├── lib/                 # Business Logic Layer
│   ├── types.ts
│   ├── utils/
│   └── validation/
└── workers/             # Data Access Layer (Cloudflare Workers)
    ├── api/
    │   ├── upload.ts
    │   ├── content.ts
    │   ├── search.ts
    │   ├── purchase.ts
    │   └── admin.ts
    ├── webhooks/
    │   └── stripe.ts
    └── cron/
        └── payouts.ts
```

### 의존성 흐름

```
UI Components (React)
    ↓
Business Logic (lib/)
    ↓
Cloudflare Workers API
    ↓
D1 Database / R2 Storage
```

### 핵심 API 엔드포인트

| 메서드 | 경로 | 설명 |
|--------|------|------|
| POST | `/api/upload` | R2 파일 업로드 + D1 메타 저장 |
| GET | `/api/content/:id` | 페이월 체크 → 티저/402 응답 |
| GET | `/api/search` | 정렬(최신/추천) + 필터 |
| POST | `/api/purchase` | Stripe Checkout 세션 생성 |
| POST | `/webhook/stripe` | 결제 완료 → D1 업데이트 |
| GET | `/feed` | RSS XML 생성 |
| GET | `/admin/reports` | 신고 큐 조회 (관리자 전용) |

---

## 데이터베이스 (D1)

### 스키마 개요

- **users**: Clerk 연동, role 기반 (consumer/creator/admin)
- **contents**: 콘텐츠 메타(title, price, is_paid, preview_images, tags)
- **purchases**: 구매 내역(Stripe session_id 포함)
- **payouts**: 크리에이터 수익 정산 내역
- **reports**: 신고 시스템(신고 3회 → 자동 숨김)

> 상세 스키마: [PRD.md 섹션 8](./PRD.md#8-데이터베이스-스키마-d1)

### 규칙
- 스키마 변경 시 마이그레이션 파일 작성 (`migrations/*.sql`)
- 실제 테이블 구조 확인 후 작업
- CASCADE 삭제 정책 신중히 사용

---

## 인증 시스템

### Clerk
- JWT 기반 인증
- Workers에서 `request.headers.get('authorization')` 검증
- 역할: `consumer` | `creator` | `admin`

### 18+ 인증
- 미국: Ondato
- 한국: 휴대폰 본인인증 → 실패 시 IP 차단 (Workers `cf.country`)

---

## 개발 워크플로우

### 로컬 개발
```bash
# Frontend
npm install
npm run dev

# Workers (로컬)
wrangler dev

# D1 마이그레이션
wrangler d1 migrations apply mooka-db --local
```

### 배포
```bash
# Frontend
npm run build
wrangler pages deploy dist

# Workers
wrangler deploy

# D1 마이그레이션 (프로덕션)
wrangler d1 migrations apply mooka-db
```

---

## 보안 & 규칙

### 필수 검증
- XSS, CSRF 방지
- R2 Presigned URL: 1회성, 5분 유효
- 관리자 엔드포인트: JWT role 검증 필수

---

## 🚨 절대 금지 사항 (CRITICAL - 반드시 준수)

### 🔴 데이터베이스 관련 절대 금지 사항

#### 파괴적 명령어 - 사용자 명시적 요청 없이 절대 금지
```bash
# D1 데이터베이스 리셋/삭제 - 절대 사용 금지
wrangler d1 execute mooka-db --command="DROP TABLE users"
wrangler d1 execute mooka-db --command="TRUNCATE TABLE contents"
wrangler d1 execute mooka-db --command="DELETE FROM purchases"
wrangler d1 migrations reset mooka-db
wrangler d1 delete mooka-db
```

#### SQL 파괴적 명령어 - 절대 금지
```sql
-- 사용자 명시적 허가 없이 절대 사용 금지
DROP TABLE / DROP DATABASE
TRUNCATE TABLE
DELETE FROM {table} WHERE 1=1  -- 전체 삭제
ALTER TABLE {table} DROP COLUMN
```

#### 데이터베이스 작업 필수 규칙
1. **데이터 삭제/리셋 전 반드시 사용자에게 명시적 허가 요청**
   - ❌ 잘못된 예: "데이터를 초기화하겠습니다"
   - ✅ 올바른 예: "⚠️ 경고: 모든 데이터가 삭제됩니다. 진행하시겠습니까? (yes/no)"

2. **백업 없이 데이터 삭제 절대 금지**
   ```bash
   # 삭제 전 반드시 백업
   wrangler d1 execute mooka-db --command="SELECT * FROM contents" > backup_contents.json
   ```

3. **프로덕션 환경에서 테스트 쿼리 금지**
   - 로컬 개발: `--local` 플래그 필수
   ```bash
   wrangler d1 execute mooka-db --local --command="SELECT * FROM users"
   ```

4. **마이그레이션 롤백 가능성 확보**
   - 모든 마이그레이션 파일에 `-- Rollback:` 섹션 포함
   ```sql
   -- Migration
   ALTER TABLE contents ADD COLUMN new_field TEXT;

   -- Rollback:
   -- ALTER TABLE contents DROP COLUMN new_field;
   ```

### 🔴 Git 위험 명령어 - 절대 사용 금지

```bash
# 협업 중 절대 금지
git push --force                    # 원격 히스토리 강제 덮어쓰기
git push --force-with-lease         # 조건부 강제 푸시
git reset --hard HEAD~1             # 로컬 커밋 영구 삭제
git commit --no-verify              # Pre-commit hook 우회
git rebase -i main                  # 공유 브랜치 리베이스

# 허용: 개인 브랜치에서만 사용자 명시적 요청 시
git push --force-with-lease origin feature/my-branch
```

#### Git 작업 필수 규칙
1. **main/master 브랜치 직접 수정 금지**
   - 반드시 feature 브랜치 생성 후 PR
2. **커밋 전 린트/테스트 통과 필수**
   ```bash
   npm run lint && npm run type-check
   ```
3. **민감 정보 커밋 방지**
   - `.env`, `*.key`, `credentials.json` 절대 커밋 금지
   - `.gitignore` 검증 필수

### 🔴 npm 위험 명령어

```bash
# 절대 사용 금지
npm audit fix --force              # 의존성 강제 업데이트 (breaking changes 위험)
npm install --legacy-peer-deps     # 의존성 충돌 무시 (장기적으로 문제 발생)
rm -rf node_modules && npm install # 캐시 문제 시에만 사용자 요청 시

# 대신 사용
npm audit fix                      # 안전한 패치만 적용
npm ci                             # package-lock.json 기반 정확한 설치
```

#### npm 작업 필수 규칙
1. **의존성 추가 전 라이선스 확인**
   - MIT, Apache 2.0 등 상업적 사용 가능 라이선스만
2. **package.json 변경 시 lock 파일도 함께 커밋**
   ```bash
   git add package.json package-lock.json
   ```
3. **보안 취약점 발견 시 즉시 보고**
   ```bash
   npm audit
   # 심각도 High 이상 발견 시 사용자에게 알림
   ```

### 🔴 파일 시스템 위험 명령어

```bash
# 절대 사용 금지
rm -rf src/                        # 소스 디렉토리 삭제
rm -rf .git/                       # Git 히스토리 삭제
chmod 777 -R ./                    # 전체 권한 부여 (보안 위험)

# 대신 사용
rm specific-file.ts                # 특정 파일만 삭제
git rm src/deprecated.ts           # Git 추적 파일 안전 삭제
```

### 🔴 Cloudflare 위험 명령어

```bash
# 사용자 명시적 요청 없이 절대 금지
wrangler delete --name mooka-api   # Workers 삭제
wrangler r2 bucket delete mooka-content  # R2 버킷 삭제
wrangler d1 delete mooka-db        # D1 데이터베이스 삭제

# 프로덕션 배포 전 확인 필수
wrangler deploy --env production   # 반드시 사용자 확인 후 실행
```

---

## 프론트엔드 상태 관리 & UX 최적화

### TanStack Query (React Query) 패턴

#### 낙관적 업데이트 (Optimistic Updates)
```typescript
// 예: 콘텐츠 좋아요 기능
const { mutate: toggleLike } = useMutation({
  mutationFn: (contentId: string) => api.toggleLike(contentId),

  // 1. 요청 전 UI 먼저 업데이트
  onMutate: async (contentId) => {
    await queryClient.cancelQueries({ queryKey: ['content', contentId] });
    const previousData = queryClient.getQueryData(['content', contentId]);

    queryClient.setQueryData(['content', contentId], (old) => ({
      ...old,
      isLiked: !old.isLiked,
      likeCount: old.isLiked ? old.likeCount - 1 : old.likeCount + 1
    }));

    return { previousData }; // 롤백용 스냅샷
  },

  // 2. 실패 시 이전 상태로 롤백
  onError: (err, contentId, context) => {
    queryClient.setQueryData(['content', contentId], context.previousData);
    toast.error('좋아요 실패');
  },

  // 3. 성공 시 서버 데이터로 동기화
  onSettled: (contentId) => {
    queryClient.invalidateQueries({ queryKey: ['content', contentId] });
  }
});
```

#### 정교한 캐시 제어
- **queryClient.setQueryData**: 즉시 캐시 업데이트 (낙관적 UI)
- **queryClient.invalidateQueries**: 백그라운드 재검증
- **staleTime / cacheTime**: 캐시 전략 최적화
  - 정적 데이터 (태그, 카테고리): `staleTime: Infinity`
  - 동적 데이터 (검색 결과): `staleTime: 1000 * 60` (1분)
  - 실시간 데이터 (구매 내역): `staleTime: 0`

#### 필수 패턴
- **모든 mutation**에 `onMutate` + `onError` 롤백 구현
- **페이지네이션**: `useInfiniteQuery` 사용
- **Prefetching**: 링크 hover 시 데이터 미리 로드
- **Error Boundary**: 전역 에러 핸들링

---

## i18n (다국어)

### 지원 언어
- 영어 (`en`)
- 한국어 (`ko`)

### 규칙
- 텍스트 하드코딩 금지 → 번역 키 사용
- 키 네이밍: dotted namespace (`common.button.submit`)
- `en.json` ↔ `ko.json` 키 동기화 필수

---

## 폴더별 문서화 규칙

### 폴더 생성 시 필수 작업
새로운 폴더를 만들 때는 반드시 해당 폴더 내에 `CLAUDE.md` 파일을 생성해야 합니다.

#### CLAUDE.md 필수 포함 내용
```markdown
# {폴더명}

## 폴더 목적
- 이 폴더가 담당하는 주요 책임과 역할
- 해결하려는 문제 또는 제공하는 기능

## 주요 파일

### {파일명1.ts}
- **역할**: 파일의 주요 책임
- **주요 함수/클래스**: 핵심 기능 목록
- **의존성**: 다른 파일/모듈과의 관계

### {파일명2.ts}
- **역할**: ...

## 의존성 관계
- **사용하는 모듈**: 이 폴더가 의존하는 외부 모듈/폴더
- **사용되는 위치**: 이 폴더를 사용하는 다른 모듈/폴더

## 사용 예시 (선택 사항)
```typescript
// 실제 사용 예제 코드
```
```

#### 예시: `src/lib/validation/CLAUDE.md`
```markdown
# validation

## 폴더 목적
- 사용자 입력 및 데이터 유효성 검증
- 업로드 파일, 가격 설정, 태그 등의 비즈니스 규칙 적용

## 주요 파일

### content-validation.ts
- **역할**: 콘텐츠 업로드 시 메타데이터 검증
- **주요 함수**:
  - `validateContentMetadata(data)`: title, description, tags 검증
  - `validatePrice(price)`: $0.99 ~ $100 범위 체크
  - `validatePreviewImages(images)`: 최대 3장 제한 검증
- **의존성**: `src/lib/types.ts` (ContentMetadata 타입)

### file-validation.ts
- **역할**: 업로드 파일 검증 (.safetensors, 이미지)
- **주요 함수**:
  - `validateFileType(file)`: 허용된 확장자 체크
  - `validateFileSize(file)`: 최대 파일 크기 제한

## 의존성 관계
- **사용하는 모듈**: `src/lib/types.ts`
- **사용되는 위치**: `src/components/upload/`, `src/workers/api/upload.ts`

## 사용 예시
```typescript
import { validateContentMetadata } from '@/lib/validation/content-validation';

const result = validateContentMetadata({
  title: "My LoRA Model",
  price: 4.99,
  previewImages: ["image1.jpg", "image2.jpg"]
});
```
```

### 폴더 문서 동기화 규칙

#### 파일 추가 시
1. 새 파일의 역할과 주요 함수를 `CLAUDE.md`의 "주요 파일" 섹션에 추가
2. 의존성 변경 시 "의존성 관계" 섹션 업데이트

#### 파일 수정 시
1. 주요 함수 변경 또는 추가 시 해당 파일 설명 업데이트
2. 책임 범위 변경 시 "역할" 섹션 수정

#### 파일 삭제 시
1. `CLAUDE.md`에서 해당 파일 설명 제거
2. 의존성 관계에서 관련 항목 제거

#### 자동화 체크리스트
```bash
# 폴더 생성 시
mkdir src/lib/new-feature
echo "# new-feature\n\n## 폴더 목적\n..." > src/lib/new-feature/CLAUDE.md

# 문서 업데이트 여부 확인
git status | grep "CLAUDE.md"  # 파일 변경 시 폴더의 CLAUDE.md도 수정되었는지 확인
```

---

## 개발 시 주의사항

### 코드 품질
1. **파일 길이**: 200줄 초과 시 모듈화, 600줄 절대 금지
2. **계층 분리**: UI에 비즈니스 로직 포함 금지
3. **타입 안정성**: TypeScript strict mode 준수
4. **컴포넌트 재사용**: `src/components/ui/` 중앙화

### 성능 & UX
5. **낙관적 업데이트**: 모든 mutation에 `onMutate` + `onError` 필수
6. **캐시 전략**: TanStack Query로 불필요한 요청 최소화
7. **R2 업로드**: 스트리밍 업로드, 진행률 표시
8. **D1 쿼리**: 인덱스 활용, N+1 쿼리 방지

### 보안 & 문서화
9. **에러 처리**: Workers에서 명확한 에러 응답 반환
10. **문서화**: 작업 내역 및 스키마 변경사항 기록
11. **폴더별 문서화**: 새 폴더 생성 시 `{폴더명}/CLAUDE.md` 필수, 파일 변경 시 즉시 업데이트
12. **최신 라이브러리**: 보안 패치 즉시 적용
