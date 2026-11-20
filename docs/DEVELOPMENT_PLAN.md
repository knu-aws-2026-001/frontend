# 이어온 하루 — Frontend 개발계획서

| 항목 | 내용 |
|---|---|
| 대상 저장소 | `knu-aws-2026-001/frontend` |
| 관련 문서 | [`MVP.md`](../MVP.md), [`REQUIREMENTS.md`](./REQUIREMENTS.md), [backend `DEVELOPMENT_PLAN.md`](https://github.com/knu-aws-2026-001/backend/blob/main/docs/DEVELOPMENT_PLAN.md) |
| 스택 | React 18, TypeScript, Vite, React Router, TanStack Query, aws-amplify/auth, MSW |

요구사항 ID(`FE-*`)는 [`REQUIREMENTS.md`](./REQUIREMENTS.md)를 가리킨다. 일정(날짜)은 팀 인원과 기간이 정해지면 6장 표에 채운다.

---

## 1. 개발 원칙

1. **목업으로 먼저, 실제 API로 나중에.** 백엔드 `MVP.md` §7 계약으로 MSW 핸들러를 만들고 모든 화면을 목업으로 완성한다. 백엔드 단계가 배포될 때마다(연동 지점 S1–S5) 해당 기능만 실제 API로 바꾼다.
2. **서버 상태가 기준.** 화면 분기(이어받기 가능 여부, 완료 문구, 종료 판단)는 누른 버튼이 아니라 서버 응답 값(`claimable`, `hasPhoto`, `claimStatus`, `photoStatus`)으로 정한다.
3. **API 호출은 한 곳에서.** 모든 요청은 `src/api/client.ts`를 거친다. 토큰 부착, 에러 파싱, 401 처리, `TRY_AGAIN` 재시도를 여기서 한다.
4. **글은 잃지 않는다.** 하루 남기기 입력과 방문 기록 글은 실패·새로고침에도 남아 있어야 한다.

## 2. 프로젝트 구조

```
src/
  api/
    client.ts           # fetch 래퍼: ID 토큰 부착, ApiError 변환, 401 → 로그인, TRY_AGAIN 1회 재시도
    types.ts            # Plan, PlanSummary, Claim, Record, InboxItem, AiCard, Flag, ErrorCode (백엔드 §7과 같은 이름)
    queries/            # usePlans, usePlan, useMyClaims, useRecord(폴링), useInbox, useInboxItem, useMe
    mutations/          # useCreatePlan, useClaim, useAbandon, usePutRecord, useReplacePhoto, useComplete, useReply, useAiCard
  auth/                 # amplify.ts(설정), AuthProvider, RequireAuth(redirect 처리)
  components/           # Header, BottomTab, StatusBadge, RouteTimeline, PrimaryButton, TextField, TextArea, Toast, ResultScreen
  pages/
    login/
    browse/             # 둘러보기
    plan/               # 상세, 작성(사연·AI 카드·폼·장소 추가·공개 완료)
    me/                 # 내 하루, 포기
    record/             # 작성, 보내는 중, 실패, 교체, 완료, 종료 안내
    inbox/              # 목록, 돌아온 하루
  state/
    planDraft.ts        # 하루 남기기 작성 상태 (sessionStorage)
    letterDraft.ts      # 방문 기록 임시 글 (localStorage, claimId별)
  mocks/
    handlers/           # 경로별 MSW 핸들러
    db.ts               # 메모리 상태 (지연·수현·민준, 예시 계획)
    browser.ts
  styles/               # tokens.css, global.css
  utils/                # 날짜(KST), 검증 규칙, uuid
docs/
  REQUIREMENTS.md
  DEVELOPMENT_PLAN.md
```

핵심 모듈 설계
- `client.ts`: 응답이 4xx/5xx면 `{ error: { code, message, details } }`를 `ApiError`로 던진다. 화면은 `error.code`로만 분기한다.
- `mocks/db.ts`: 메모리 상태 머신으로 Plan·Claim·Record·InboxItem 상태 전이를 흉내 낸다. 경합(`ALREADY_CLAIMED`), 사진 처리 지연·실패, AI 실패를 쿼리 파라미터나 개발용 토글로 재현할 수 있게 한다(예: `?mock=photo-fail`, `?mock=ai-fail`).
- `planDraft.ts`: `{ story, draftId, card, flags, form, mode, startedAt, idempotencyKey, keyBodyHash }`를 저장·복원한다. 본문 해시가 바뀌면 키를 새로 만든다(FE-CRT-10).
- `useRecord`: TanStack Query `refetchInterval: 2000`. 판정 함수 `decideSendingState(response, elapsedMs)`를 따로 두고 단위 테스트한다(FE-REC-06, 07).

## 3. 단계별 계획

순서는 `MVP.md` §11을 따른다. 각 단계는 MSW 목업으로 동작하는 상태까지 완성한다.

### 1단계. 프로젝트 세팅과 공통 레이아웃

| 작업 | 요구사항 |
|---|---|
| Vite + TS + React Router + TanStack Query, ESLint/Prettier, `.env.example` | FE-TECH-01, 03 |
| 스타일 방식 결정(Q3), 디자인 토큰(`tokens.css`), 모바일 레이아웃(최대 480px) | FE-TECH-06, FE-NFR-01, 02 |
| 공통 컴포넌트: Header(뒤로가기·제목·편지 아이콘), BottomTab, PrimaryButton(하단 고정), StatusBadge, RouteTimeline, TextField/TextArea(글자 수·에러), Toast, ResultScreen(원형 아이콘 + 제목 + 설명 + 요약 카드) | FE-COM-01, 02 |
| `api/types.ts`, `client.ts`, MSW 설정과 `mocks/db.ts` 시드 데이터 | FE-TECH-04, 05, FE-COM-04, 05 |
| 라우트 골격(§5 화면 목록의 모든 경로, 빈 페이지) | — |

- 완료 조건: `npm run dev`(`VITE_USE_MOCK=true`)로 탭 이동이 되고, 모든 경로가 열린다.
- 이 단계는 백엔드 연동 지점 S0(계약 확정)만 필요하다.

### 2단계. 둘러보기 → 하루 상세 → 이어받기

| 작업 | 요구사항 |
|---|---|
| 둘러보기 카드 목록, 빈 상태 | FE-BRW-01–04 |
| 하루 상세: 계획자·취소 이유·동선 타임라인, 7가지 상태 분기 | FE-PLN-01, 02 |
| 이어받기 버튼, 201/200 → `/me`, 409 `ALREADY_CLAIMED` 안내 박스, `CANNOT_CLAIM_OWN_PLAN` 토스트 | FE-PLN-03–05 |

- 검증: 목업에서 사용자 전환(지연/수현/민준)으로 각 분기가 보이는지 확인. 경합은 두 브라우저 탭에서 목업 토글로 재현.

### 3단계. 하루 남기기 (직접 작성 경로 먼저)

| 작업 | 요구사항 |
|---|---|
| 사연 입력 화면(3), 직접 작성 → 폼 | FE-CRT-01, 05 |
| 폼(3-2) 검증, 장소 추가(3-3), 공개 완료(3-4) | FE-CRT-06, 07, 11 |
| `planDraft` sessionStorage 보관·복원, `startedAt` 1회 기록 | FE-CRT-08, 09 |
| `POST /plans` + `Idempotency-Key`, `IDEMPOTENCY_CONFLICT` 재요청, `VALIDATION_ERROR` details 표시 | FE-CRT-10, FE-COM-04 |

- 검증: 폼 작성 중 장소 추가 화면 왕복·새로고침 후에도 값과 `startedAt` 유지. 공개 요청 본문에 `authoring` 포함.

### 4단계. 내 하루 → 방문 기록(글만) → 수신함 → 답장

| 작업 | 요구사항 |
|---|---|
| 내 하루(4): 기한 문구(KST), 동선, `photoStatus`별 이동 | FE-ME-01–03, FE-COM-06 |
| 포기 확인(4-1), `CLAIM_NOT_ACTIVE` 처리 | FE-ME-04 |
| 방문 기록 작성(5): 질문 표시, 서버 글 → localStorage 순서로 채우기 | FE-REC-01–03 |
| 글로 완료: `PUT /record` → `complete`, `RECORD_REQUIRED`·`TRY_AGAIN` 재시도, 전송 완료(5-4) | FE-REC-05, 10 |
| 종료 안내(8) | FE-REC-11 |
| 수신함(6), 돌아온 하루(7), 읽음 처리·캐시 무효화, 답장·보낸 답장(7-1), 헤더 미확인 점 | FE-INB-01–05, FE-COM-02 |

- 이 단계가 끝나면 글 경로 기준으로 시연 시나리오 전체가 목업에서 동작한다.
- **백엔드 연동 S2**가 이 시점 전후로 열리면 2–4단계 기능을 실제 API로 전환한다(4장 참고).

### 5단계. 사진 업로드·폴링·실패·교체

| 작업 | 요구사항 |
|---|---|
| 사진 선택·미리보기, 형식·크기 1차 검사 | FE-REC-03 |
| `PUT /record { photo }` → presigned PUT 업로드 → 보내는 중(5-1) | FE-REC-04 |
| `useRecord` 폴링과 `decideSendingState` 판정, 60초 강조, `UPLOAD_PENDING` 30초 실패 처리 | FE-REC-06, 07 |
| 실패(5-2), 사진 교체(5-3), 사진 포함 완료 문구 | FE-REC-08–10 |
| 돌아온 하루의 사진·캡션, URL 만료 시 재조회 | FE-INB-03, 04 |

- 목업: `mocks/db.ts`가 업로드 후 일정 시간 뒤 `READY`+`delivered` 또는 `FAILED`로 바뀌게 한다.
- **백엔드 연동 S3** 이후 실제 S3 업로드로 확인한다. presigned URL은 `Content-Type`과 `Content-Length`가 서명에 포함되므로 업로드 시 `upload.headers`를 그대로 쓰고 파일을 변환하지 않는다.

### 6단계. AI 하루 카드와 작성 시간 측정

| 작업 | 요구사항 |
|---|---|
| "AI로 하루 카드 만들기" → `POST /ai/day-card`, 로딩 상태(최대 30초) | FE-CRT-02 |
| AI 카드 확인(3-1): `flags[].path` ↔ 입력칸 연결, 강조 문구 | FE-CRT-03 |
| AI 실패 → 사연 유지, 참고용 사연 표시, 빈 폼, `MANUAL` | FE-CRT-04 |
| `authoring.mode = AI_EDITED`, `aiDraftId` 전송 | FE-CRT-09 |

- `flags[].path`(`places[0].name` 형식)와 `VALIDATION_ERROR`의 `details` 키가 같은 표기이므로, 폼 필드 경로 → 에러/플래그 메시지 매핑 함수를 하나로 만들어 둘 다 쓴다.
- **백엔드 연동 S4** 이후 실제 Bedrock 응답으로 확인한다.

### 7단계. Cognito 연동, 실제 API 전환, 배포, 리허설

| 작업 | 요구사항 |
|---|---|
| 로그인·회원가입(0), `RequireAuth`, 401 redirect | FE-AUTH-01–03, FE-COM-03 |
| `VITE_USE_MOCK=false`로 전 기능 점검 | FE-NFR-07 |
| 배포 스크립트: `npm run build` → `aws s3 sync dist/ s3://{WebBucketName} --delete` → `aws cloudfront create-invalidation --distribution-id {DistributionId} --paths "/*"` | FE-TECH-07 |
| 360/390/480px 기기에서 시안 비교, 접근성 점검 | FE-NFR-01, 06 |
| 백엔드와 90초 시연 리허설(지연·수현·민준 3개 브라우저) | §6 인수 기준 |

- 로그인 화면은 마지막 단계에 있지만 Cognito 설정(S1)은 백엔드 1단계에서 나오므로, 인원이 있으면 S1 직후 앞당겨 진행해도 된다.

## 4. 백엔드와의 연동 지점

| 지점 | 백엔드가 넘기는 것 | 프론트 작업 |
|---|---|---|
| S0 | 백엔드 `MVP.md` §7 계약 | `types.ts`, MSW 핸들러 작성 (1단계) |
| S1 | `ApiUrl`, `UserPoolId`, `UserPoolClientId` | `.env.local` 설정, 로그인·가입, `GET /me` |
| S2 | 계획·이어받기·포기·글 기록·완료·수신함·답장 API | 2–4단계 기능을 실제 API로 전환 |
| S3 | 사진 업로드 URL, 사진 처리 파이프라인, 버킷 CORS | 5단계 실제 업로드·폴링 확인 |
| S4 | `POST /ai/day-card` | 6단계 실제 AI 응답 확인 |
| S5 | `WebBucketName`, `DistributionId`, `DistributionDomain`, seed 데이터 | 배포, 리허설 |

전환 절차(각 지점마다)
1. `.env.local`의 `VITE_API_BASE_URL`을 `dev` 스택으로 맞추고 `VITE_USE_MOCK=false`.
2. 해당 기능의 실제 응답을 MSW 핸들러 응답과 비교한다. 다르면 계약 문서(백엔드 `MVP.md` §7) 기준으로 어느 쪽을 고칠지 정하고 이슈를 남긴다.
3. 목업에서 재현하던 경합·실패 시나리오를 실제 API로 한 번씩 다시 확인한다.

계약 변경은 백엔드 `MVP.md` §7 수정 PR에 프론트 담당이 리뷰로 참여하고, 승인되면 `types.ts`와 MSW 핸들러를 같은 PR에서 고친다.

## 5. 테스트 전략

| 수준 | 도구 | 대상 |
|---|---|---|
| 단위 | Vitest | 폼 검증 규칙, `decideSendingState` 판정 순서, 기한 문구(KST), 상세 화면 분기 함수, `planDraft` 키 재생성 규칙, 경로 ↔ 메시지 매핑 |
| 컴포넌트 | Vitest + Testing Library + MSW | 이어받기 경합 안내, AI 실패 전환, 사진 실패 → 글로 완료, 답장 중복(409) 처리 |
| E2E (선택) | Playwright + MSW | 90초 시나리오를 목업으로 자동 실행. 시간이 부족하면 수동 체크리스트로 대체 |
| 수동 | 체크리스트 | `REQUIREMENTS.md` §6 인수 기준, 실제 기기(모바일 브라우저) 확인 |

## 6. 일정 표 (팀이 채움)

| 단계 | 담당 | 시작 | 완료 | 필요한 연동 지점 |
|---|---|---|---|---|
| 1. 세팅·공통 레이아웃 | | | | S0 |
| 2. 둘러보기·상세·이어받기 | | | | (S2에서 전환) |
| 3. 하루 남기기(직접 작성) | | | | (S2에서 전환) |
| 4. 내 하루·기록(글)·수신함 | | | | S2 |
| 5. 사진 | | | | S3 |
| 6. AI 하루 카드 | | | | S4 |
| 7. 인증·배포·리허설 | | | | S1, S5 |

병렬 작업이 가능한 조합: 1단계 공통 컴포넌트가 끝나면 "2·4단계(실행자·수신함 화면)"와 "3·6단계(하루 남기기 화면)"를 나눠 진행할 수 있다. 두 흐름은 `types.ts`와 공통 컴포넌트만 공유한다.

## 7. 작업 규칙 (제안)

- 브랜치: `main`은 빌드 가능한 상태 유지. 작업은 `feat/<단계>-<요약>`(예: `feat/5-photo-polling`) 브랜치에서 하고 PR로 병합한다.
- PR: 관련 요구사항 ID, 확인한 화면 번호, 목업/실제 API 중 어디서 확인했는지 적는다. 화면 변경은 스크린샷(390px) 첨부. 리뷰어 1명 이상.
- 커밋 메시지: `feat:`, `fix:`, `test:`, `style:`, `chore:`, `docs:` 접두어.
- 비밀값: `.env.local`은 gitignore. 저장소에는 `.env.example`만 둔다.

## 8. 위험과 대응

| 위험 | 영향 | 대응 |
|---|---|---|
| 목업과 실제 응답 형태 차이 | 연동 시 화면 오류 | `types.ts`를 계약과 같은 이름으로 유지, S2–S4마다 응답 비교 |
| 사진 업로드 CORS·서명 오류 | 사진 기능 시연 실패 | S3 직후 실제 업로드 확인, `upload.headers` 그대로 사용, 실패 시 "글로 완료" 경로가 항상 동작하도록 유지 |
| AI 응답 지연(최대 약 30초) | 사용자가 이탈하거나 중복 클릭 | 로딩 화면과 버튼 비활성화, 실패 시 즉시 직접 작성으로 전환 |
| 폴링 상태 판정 오류 | 완료 화면이 안 뜨거나 잘못된 문구 | 판정 함수를 분리해 단위 테스트, 서버 값으로만 문구 결정 |
| SPA 새로고침 시 CloudFront 404 | 배포 환경에서 하위 경로 접근 실패 | 백엔드에 403/404 → `/index.html` 설정 요청(백엔드 BE-INFRA-02) |
| 시안 색상 미확정 | 막바지 디자인 수정 | 토큰으로만 색을 쓰고 값만 교체 |

## 9. 완료 체크리스트

[`REQUIREMENTS.md`](./REQUIREMENTS.md) §6 인수 기준(= `MVP.md` §10)을 모두 만족하면 완료로 본다.
