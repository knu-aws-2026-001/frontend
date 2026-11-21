# 이어온 하루 — Frontend 개발계획서

| 항목 | 내용 |
|---|---|
| 대상 저장소 | `knu-aws-2026-001/frontend` |
| 관련 문서 | [`MVP.md`](../MVP.md), [`REQUIREMENTS.md`](./REQUIREMENTS.md), [backend `DEVELOPMENT_PLAN.md`](https://github.com/knu-aws-2026-001/backend/blob/main/docs/DEVELOPMENT_PLAN.md) |
| 스택 | React 18, TypeScript, Vite, React Router, TanStack Query, aws-amplify/auth, MSW |

요구사항 ID(`FE-*`)는 [`REQUIREMENTS.md`](./REQUIREMENTS.md)를 가리킨다. 일정(날짜)은 팀 인원과 기간이 정해지면 6장 표에 채운다.

---

## 1. 개발 원칙

1. **목업으로 먼저, 실제 API로 나중에.** 백엔드 `MVP.md` §7 계약으로 MSW 핸들러를 만들고 모든 화면을 목업으로 완성한다. 백엔드 단계가 배포될 때마다(연동 지점 S1–S5) 해당 기능만 실제 API로 바꾼다. 백엔드 기능이 로컬(docker compose)에 먼저 올라오면 로컬 백엔드 모드로 흐름을 미리 확인할 수 있지만, 계약 비교와 인수 판정은 `dev` 스택으로 한다(4.1).
2. **서버 상태가 기준.** 화면 분기(이어받기 가능 여부, 완료 문구, 종료 판단, AI 정리 단계)는 누른 버튼이 아니라 서버 응답 값(`claimable`, `hasPhoto`, `claimStatus`, `photoStatus`, AI 세션 `status`)으로 정한다.
3. **API 호출은 한 곳에서.** 모든 요청은 `src/api/client.ts`를 거친다. 토큰 부착, 에러 파싱, 401 처리, `TRY_AGAIN` 재시도를 여기서 한다.
4. **글은 잃지 않는다.** 하루 남기기 입력과 방문 기록 글은 실패·새로고침에도 남아 있어야 한다.

## 2. 프로젝트 구조

```
src/
  api/
    client.ts           # fetch 래퍼: ID 토큰 부착, ApiError 변환, 401 → 로그인, TRY_AGAIN 1회 재시도
    types.ts            # Plan, PlanSummary, Claim, Record, InboxItem, SessionView, AiQuestion, AiNote, AiTraceStep(step, tool, summary), Flag, ErrorCode (백엔드 §7과 같은 이름)
    queries/            # usePlans, usePlan, useMyClaims, useRecord(폴링), useInbox, useInboxItem, useMe
    mutations/          # useCreatePlan, useClaim, useAbandon, usePutRecord, useReplacePhoto, useComplete, useReply, useAiSession(start/turn/get)
  auth/                 # amplify.ts(설정, VITE_COGNITO_ENDPOINT가 있으면 로컬 엔드포인트), AuthProvider, RequireAuth(redirect 처리),
                        # devJwt.ts(VITE_AUTH_MODE=dev-jwt, 개발 빌드 전용)
  components/           # Header, BottomTab, StatusBadge, RouteTimeline, PrimaryButton, TextField, TextArea, Toast, ResultScreen
  pages/
    login/
    browse/             # 둘러보기
    plan/               # 상세, 작성(사연·AI 정리·폼·장소 추가·공개 완료)
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
.env.example            # 세 연동 모드의 예시 값(4.1). 실제 .env.local은 gitignore
```

핵심 모듈 설계
- `client.ts`: 응답이 4xx/5xx면 `{ error: { code, message, details } }`를 `ApiError`로 던진다. 화면은 `error.code`로만 분기한다.
- `mocks/db.ts`: 메모리 상태 머신으로 Plan·Claim·Record·InboxItem·AI 세션 상태 전이를 흉내 낸다. 경합(`ALREADY_CLAIMED`), 사진 처리 지연·실패, AI 경로를 쿼리 파라미터나 개발용 토글로 재현할 수 있게 한다(예: `?mock=photo-fail`).
  - AI 세션(`/ai/card-sessions*`)은 `turn`·`expectedTurn` 규칙을 계약대로 지킨다. 파라미터가 없으면 레벨 0처럼 생성 응답에서 바로 `READY`.
  - `?mock=ai-ask`: 데모 사연 경로. 생성 → `ASKING`(소요 시간 질문 1개, [30분][45분][1시간]) → 답변 → `READY`(`REWRITTEN`·`ASKED` 메모). 건너뛰기·`finish`도 같은 핸들러가 처리한다.
  - `?mock=ai-working`: 생성 → `WORKING` 연속 2회 → `finish: true` 요청 → `READY`.
  - `?mock=ai-degraded`: `READY` + `degraded: true`, 남은 `NOT_IN_SOURCE`·`PRIVACY` 플래그와 `REMOVED`·`KEPT_FLAGGED` 메모.
  - `?mock=ai-busy`: 첫 `turns`에 409 `AI_SESSION_BUSY`, 이어지는 첫 `GET`은 같은 `turn`의 `WORKING`(처리 중), 그다음 `GET`은 진행된 `SessionView`. `READY` 뒤 같은 `expectedTurn`의 `turns`에는 409 `AI_SESSION_CLOSED`.
  - `?mock=ai-fail`: 생성에 503 `AI_UNAVAILABLE`(세션 `FAILED`). `?mock=ai-fail-turn`: 생성은 `ASKING`, 답변 `turns`에 503 `AI_UNAVAILABLE`, 이후 `GET`은 `FAILED`.
  - `?mock=ai-quota`: 생성에 429 `AI_QUOTA_EXCEEDED`. `?mock=ai-invalid`: 생성에 502 `AI_INVALID_OUTPUT`(레벨 0).
- `planDraft.ts`: `{ story, sessionId, turn, card, flags, notes, form, mode, startedAt, idempotencyKey, keyBodyHash }`를 저장·복원한다. 본문 해시가 바뀌면 키를 새로 만든다(FE-CRT-10).
- `useAiSession`: `start`(`POST /ai/card-sessions`), `turn`(`POST .../turns`), `get`(`GET .../{sessionId}`)을 묶는다. 받은 `SessionView`로 다음 요청을 정하는 함수 `decideAiNextStep(view, consecutiveWorking, finishRequested)`를 따로 두고 단위 테스트한다(FE-CRT-02, 12, 13).
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
- 로컬 백엔드 모드에서는 Floci S3(`http://localhost:4566`)로 업로드와 폴링 흐름을 미리 볼 수 있다. 서명 강제(ContentType/ContentLength)와 CloudFront Origin 기준 버킷 CORS는 로컬에서 실제와 같다고 볼 수 없으므로 S3 연동 확인은 `dev` 스택에서 한다.

### 6단계. AI와 함께 정리하기와 작성 시간 측정

| 작업 | 요구사항 |
|---|---|
| 사연 입력의 "AI와 함께 정리하기" → `POST /ai/card-sessions`, 외부 AI 전송 안내 문구(Q6) | FE-CRT-01, 02 |
| `useAiSession`(start/turn/get), `turn` → `expectedTurn` 관리, `decideAiNextStep` | FE-CRT-02, 13 |
| 3-1 새 화면: 단계 칩(`trace`), 질문 말풍선(선택지·직접 입력·건너뛰기), "이대로 카드 받기", 카드 미리보기("원문 대조 통과" 배지) | FE-CRT-03, 12 |
| `WORKING` 자동 continue와 연속 2회 후 `finish`, `AI_SESSION_BUSY` 2초 뒤 `GET` 재동기화, `AI_SESSION_CLOSED` 처리, 새로고침 세션 복구 | FE-CRT-13 |
| "AI가 바꾼 것" 메모(출처 칩, "다시 넣기"), 폼(3-2)의 `flags[].path` 연결(`PRIVACY` 포함) | FE-CRT-03, 14 |
| AI 실패(`FAILED`, `AI_INVALID_OUTPUT`, `AI_QUOTA_EXCEEDED`, 세션 404) → 사연 유지, 참고용 사연 표시, 빈 폼, `MANUAL` | FE-CRT-04 |
| `authoring.mode = AI_EDITED`, `aiSessionId` 전송 | FE-CRT-09 |
| MSW AI 세션 상태 머신(`ai-ask`, `ai-working`, `ai-degraded`, `ai-busy`, `ai-fail`, `ai-fail-turn`, `ai-quota`, `ai-invalid`) | FE-TECH-04 |

- `flags[].path`(`places[0].name` 형식)와 `VALIDATION_ERROR`의 `details` 키가 같은 표기이므로, 폼 필드 경로 → 에러/플래그 메시지 매핑 함수를 하나로 만들어 둘 다 쓴다. 메모 "다시 넣기"도 같은 경로 표기를 쓴다.
- 요청당 응답은 30초 안에 오지만 한 세션은 여러 턴이다. 로딩은 전체 화면 대기가 아니라 단계 칩으로 보여 준다.
- **백엔드 연동 S4**: 백엔드 6a(레벨 0) 배포 후 실제 API로 전환한다. 이때는 생성 응답이 바로 `READY`이고 질문 말풍선이 나오지 않는다. 백엔드 6b(LangGraph 에이전트 그래프)는 계약 변경 없이 교체되므로, 질문·`WORKING`·`AI_SESSION_BUSY`·`AI_SESSION_CLOSED` 경로는 6b 배포 후 같은 화면에서 실제 API로 다시 확인한다.

### 7단계. Cognito 연동, 실제 API 전환, 배포, 리허설

| 작업 | 요구사항 |
|---|---|
| 로그인·회원가입(0), `RequireAuth`, 401 redirect | FE-AUTH-01–03, FE-COM-03 |
| `VITE_USE_MOCK=false`로 전 기능 점검 | FE-NFR-07 |
| 배포 스크립트: `npm run build` → `aws s3 sync dist/ s3://{WebBucketName} --delete` → `aws cloudfront create-invalidation --distribution-id {DistributionId} --paths "/*"` | FE-TECH-07 |
| 360/390/480px 기기에서 시안 비교, 접근성 점검 | FE-NFR-01, 06 |
| 백엔드와 90초 시연 리허설(지연·수현·민준 3개 브라우저) | §6 인수 기준 |
| AI 정리 시연 리허설: 데모 사연으로 질문 1개 경로 10회 이상 재현 확인 | FE-CRT-02, 12 |

- 로그인 화면은 마지막 단계에 있지만 Cognito 설정(S1)은 백엔드 1단계에서 나오므로, 인원이 있으면 S1 직후 앞당겨 진행해도 된다. 백엔드 로컬 환경(1단계)이 먼저 뜨면 로컬 백엔드 모드에서 시드 사용자로 로그인·`GET /me`를 먼저 확인한다(Q8). 회원가입 자동 확인은 로컬에서 동작하지 않으므로 FE-AUTH-02는 `dev` 스택에서 확인한다.
- AI 정리 시연 구간(약 20초, 90초 시나리오 안에서)
  - 0:00 지연이 사연을 붙여 넣고 "AI와 함께 정리하기"를 누른다.
  - 0:03 단계 칩이 `trace` 순서대로 켜진다: '골목 빵집' 원문에 없음, 소요 시간 정보 없음 → '출근길 빵집'으로 고침, 개인정보 확인 통과 → 소요 시간 질문.
  - 0:07 질문 말풍선 [30분][45분][1시간]. 지연이 45분을 탭한다.
  - 0:12 카드, "원문 대조 통과" 배지, 메모 1줄("'골목 빵집' 대신 원문의 '출근길 빵집'으로 적었어요 · 원문").
  - 0:17 "폼에서 다듬기" → 공개.
  - 발표 멘트: "에이전트는 검사 도구로 스스로 고쳤고, 모르는 것은 지어내지 않고 한 번만 물었습니다."
  - 슬라이드: 같은 세션의 `trace`, 오프라인 평가 표(비공개 장소·연락처·인젝션 함정 사연 포함).
- 시연 안정장치: 직전에 한 번 호출해 콜드 스타트를 없앤다(에이전트 함수는 LangGraph 의존성이 커서 첫 호출이 느릴 수 있다, 백엔드 명세 Q18). 지연이 길면 백엔드가 `AGENT_MODE=pipeline`으로 바꾸고, 같은 화면에서 질문 말풍선만 빠진다.

## 4. 백엔드와의 연동 지점

| 지점 | 백엔드가 넘기는 것 | 프론트 작업 |
|---|---|---|
| S0 | 백엔드 `MVP.md` §7 계약 (card-sessions 계약 포함) | `types.ts`, MSW 핸들러(AI 세션 상태 머신 포함) 작성 (1단계) |
| S1 | `ApiUrl`, `UserPoolId`, `UserPoolClientId` (로컬 백엔드 모드는 init 스크립트가 출력한 로컬 값) | `.env.local` 설정, 로그인·가입, `GET /me` |
| S2 | 계획·이어받기·포기·글 기록·완료·수신함·답장 API | 2–4단계 기능을 실제 API로 전환 |
| S3 | 사진 업로드 URL, 사진 처리 파이프라인, 버킷 CORS | 5단계 실제 업로드·폴링 확인 |
| S4 | `/ai/card-sessions*` (백엔드 6a, 레벨 0) | 6단계 실제 연동. 백엔드 6b는 계약 변경 없이 교체되며 6b 배포 후 질문·`WORKING`·`AI_SESSION_BUSY`·`AI_SESSION_CLOSED` 경로를 다시 확인 |
| S5 | `WebBucketName`, `DistributionId`, `DistributionDomain`, seed 데이터 | 배포, 리허설 |

전환 절차(각 지점마다)
1. `.env.local`의 `VITE_API_BASE_URL`을 `dev` 스택으로 맞추고 `VITE_USE_MOCK=false`.
2. 해당 기능의 실제 응답을 MSW 핸들러 응답과 비교한다. 다르면 계약 문서(백엔드 `MVP.md` §7) 기준으로 어느 쪽을 고칠지 정하고 이슈를 남긴다.
3. 목업에서 재현하던 경합·실패 시나리오를 실제 API로 한 번씩 다시 확인한다.

계약 변경은 백엔드 `MVP.md` §7 수정 PR에 프론트 담당이 리뷰로 참여하고, 승인되면 `types.ts`와 MSW 핸들러를 같은 PR에서 고친다.

### 4.1 개발 연동 모드

백엔드 [`DEVELOPMENT_PLAN.md`](https://github.com/knu-aws-2026-001/backend/blob/main/docs/DEVELOPMENT_PLAN.md) 10.6과 같다. 백엔드는 예산 문제로 개발 단계에서 AWS 서비스 대신 AWS 호환 에뮬레이터(Floci) 컨테이너를 쓴다. 로컬 환경은 개발과 단위·로컬 통합 테스트용이고, 데모·시범 운영·동시성 통합 테스트는 실제 AWS에 SAM으로 배포한 스택에서 한다(데모까지 Docker로 운영하는 대안은 [`REQUIREMENTS.md`](./REQUIREMENTS.md) Q9).

| 모드 | API | 인증 | 쓰는 때 |
|---|---|---|---|
| MSW 목업 | 브라우저 안 목업 | 목업 | S0부터, 계약 기준 화면 개발 |
| 로컬 백엔드 | `http://localhost:3000`(어댑터) 또는 Vite 프록시 | Floci Cognito(`http://localhost:4566`)의 시드 사용자, 또는 dev JWT | 백엔드 기능이 로컬에 올라온 뒤 실제 흐름 확인 |
| `dev`/`demo` 스택 | SAM 출력 `ApiUrl` | 실제 Cognito(`UserPoolId`, `UserPoolClientId`) | S1–S5 연동, 리허설, 시연 |

모드별 `.env.local` 값

| 변수 | MSW 목업 | 로컬 백엔드 | `dev`/`demo` 스택 |
|---|---|---|---|
| `VITE_USE_MOCK` | `true` | `false` | `false` |
| `VITE_API_BASE_URL` | (쓰지 않음) | `http://localhost:3000` (Vite 프록시를 쓰면 프록시 경로) | SAM 출력 `ApiUrl` |
| `VITE_AWS_REGION` | `ap-northeast-2` | `ap-northeast-2` | `ap-northeast-2` |
| `VITE_COGNITO_USER_POOL_ID` / `VITE_COGNITO_CLIENT_ID` | (쓰지 않음) | init 스크립트가 출력한 로컬 값 | SAM 출력 `UserPoolId` / `UserPoolClientId` |
| `VITE_COGNITO_ENDPOINT` | (쓰지 않음) | `http://localhost:4566` | 비움 |
| `VITE_AUTH_MODE` | (쓰지 않음) | `cognito`(기본) / `dev-jwt`(Q8) | `cognito` |

로컬 백엔드 모드 사용 방법
- 백엔드 저장소에서 `docker compose up`을 실행하면 init 스크립트가 테이블·버킷·큐·유저 풀·시드 사용자를 만든다. 프론트는 호스트에서 `npm run dev`(Vite `http://localhost:5173`)로 띄운다. 어댑터의 CORS 허용 Origin은 `http://localhost:5173`이다(또는 Vite 프록시).
- 인증: Floci Cognito의 시드 사용자(`AdminCreateUser`, `AdminSetUserPassword --permanent`, `nickname`)로 로그인한다. 어댑터가 ID 토큰을 로컬 User Pool JWKS로 검증해 클레임을 넣으므로, API 호출 방식(`Authorization: Bearer {ID token}`)은 AWS와 같다. Amplify가 사용자 지정 엔드포인트를 지원하지 않거나 ID 토큰에 `nickname`이 없으면(Q8) `VITE_AUTH_MODE=dev-jwt`로 어댑터의 dev JWT 발급 엔드포인트(`STAGE=local` 전용)에서 토큰을 받는다. 어댑터는 `STAGE=local`에서 Floci User Pool JWKS와 dev JWT 공개키를 둘 다 신뢰하므로 dev JWT도 같은 검증을 통과하고, API 호출 방식은 바뀌지 않는다. 발급 경로와 요청 형식은 백엔드 1단계에서 정해지는 대로 `auth/devJwt.ts`에 맞춘다. 자동화 테스트(Playwright)도 dev JWT를 쓴다.
- 회원가입: 자동 확인(PreSignUp 트리거)이 로컬에서 동작하지 않으므로 로컬에서는 시드 사용자로만 로그인하고, FE-AUTH-02는 `dev` 스택에서 확인한다.
- 사진: presigned URL은 `http://localhost:4566` 호스트와 경로 방식 주소로 서명되어 온다. `upload.url`과 `upload.headers`를 그대로 쓴다. init 스크립트가 사진 버킷 CORS에 `http://localhost:5173`을 넣는다.
- AI 정리: 로컬 백엔드의 LLM 기본값은 Fake 채팅 모델(`LLM_PROVIDER=fake`)이라 응답은 백엔드 `.env`의 `FAKE_LLM_SCENARIO`로 정해진다. 화면 분기 확인은 MSW `?mock=ai-*`로 한다. 백엔드가 네이티브 Ollama를 붙이면 요청당 30초 상한이 지켜지지 않을 수 있으므로 타임아웃 동작은 `dev` 스택에서 확인한다.

로컬 백엔드 모드에서 확인할 수 없어 `dev` 스택에서 확인하는 항목(백엔드 `DEVELOPMENT_PLAN.md` 10.4 중 프론트 관련)
1. HTTP API JWT authorizer의 401 응답 형태, API Gateway 수준의 CORS preflight와 오류 응답의 CORS 헤더
2. Cognito: 회원가입 자동 확인, `nickname` 필수 속성, 실제 ID 토큰 클레임과 만료
3. 동시 이어받기 경합 판정(T3, 수현·민준 두 브라우저). 로컬 에뮬레이터는 TransactionConflict와 실제 병렬성을 재현하지 않는다(DynamoDB Local 기준. Floci는 재현 언급이 없어 미확인)
4. S3: presigned PUT의 ContentType/ContentLength 서명 강제, CloudFront Origin 기준 버킷 CORS
5. CloudFront: SPA 대체 응답(Q4), 캐시 무효화, HTTPS 도메인
6. AI 정리의 실제 LLM 공급자 지연과 도구 호출 동작(질문 1개 경로 재현, 콜드 스타트)

로컬 백엔드 모드의 동작은 계약 일치의 근거가 아니다. 계약 비교(S2·S3·S4)는 `dev` 스택 응답으로 한다.

## 5. 테스트 전략

| 수준 | 도구 | 대상 |
|---|---|---|
| 단위 | Vitest | 폼 검증 규칙, `decideSendingState` 판정 순서, `decideAiNextStep`, 기한 문구(KST), 상세 화면 분기 함수, `planDraft` 키 재생성 규칙, 경로 ↔ 메시지 매핑 |
| 컴포넌트 | Vitest + Testing Library + MSW | 이어받기 경합 안내, AI 실패 전환, AI 질문 → READY, WORKING 자동 continue(연속 2회 후 `finish`), `AI_SESSION_BUSY` 재동기화(같은 `turn`의 `WORKING`은 연속 횟수에 넣지 않음), `turns` 503 후 `FAILED` 전환, 사진 실패 → 글로 완료, 답장 중복(409) 처리 |
| E2E (선택) | Playwright + MSW | 90초 시나리오를 목업으로 자동 실행. 시간이 부족하면 수동 체크리스트로 대체 |
| 로컬 통합 (선택) | 수동 또는 Playwright + 로컬 백엔드(docker compose) | 시드 사용자 로그인(또는 dev JWT), 글 경로, 사진 presigned 업로드·폴링을 연동 전에 확인. 동시 이어받기는 조건식 로직(한 명만 성공, 나머지 409)만 본다 |
| 수동 | 체크리스트 | `REQUIREMENTS.md` §6 인수 기준, 실제 기기(모바일 브라우저) 확인. 인수 판정은 `dev`/`demo` 스택에서 한다 |

## 6. 일정 표 (팀이 채움)

| 단계 | 담당 | 시작 | 완료 | 필요한 연동 지점 |
|---|---|---|---|---|
| 1. 세팅·공통 레이아웃 | | | | S0 |
| 2. 둘러보기·상세·이어받기 | | | | (S2에서 전환) |
| 3. 하루 남기기(직접 작성) | | | | (S2에서 전환) |
| 4. 내 하루·기록(글)·수신함 | | | | S2 |
| 5. 사진 | | | | S3 |
| 6. AI와 함께 정리하기 | | | | S4 (백엔드 6a), 질문·`WORKING`·`AI_SESSION_BUSY`·`AI_SESSION_CLOSED` 경로는 6b 후 재확인 |
| 7. 인증·배포·리허설 | | | | S1, S5 |

6단계는 3-1 새 화면, `useAiSession`, MSW 상태 머신으로 이전 계획보다 약 2인일 늘어난다.

병렬 작업이 가능한 조합: 1단계 공통 컴포넌트가 끝나면 "2·4단계(실행자·수신함 화면)"와 "3·6단계(하루 남기기 화면)"를 나눠 진행할 수 있다. 두 흐름은 `types.ts`와 공통 컴포넌트만 공유한다.

## 7. 작업 규칙 (제안)

- 브랜치: `main`은 빌드 가능한 상태 유지. 작업은 `feat/<단계>-<요약>`(예: `feat/5-photo-polling`) 브랜치에서 하고 PR로 병합한다.
- PR: 관련 요구사항 ID, 확인한 화면 번호, 어느 연동 모드(MSW 목업 / 로컬 백엔드 / `dev` 스택)에서 확인했는지 적는다. 화면 변경은 스크린샷(390px) 첨부. 리뷰어 1명 이상.
- 커밋 메시지: `feat:`, `fix:`, `test:`, `style:`, `chore:`, `docs:` 접두어.
- 비밀값: `.env.local`은 gitignore. 저장소에는 `.env.example`만 둔다. `VITE_*`는 번들에 들어가므로 비밀값(LLM API 키 등)을 넣지 않는다.

## 8. 위험과 대응

| 위험 | 영향 | 대응 |
|---|---|---|
| 목업과 실제 응답 형태 차이 | 연동 시 화면 오류 | `types.ts`를 계약과 같은 이름으로 유지, S2–S4마다 응답 비교 |
| 사진 업로드 CORS·서명 오류 | 사진 기능 시연 실패 | S3 직후 실제 업로드 확인, `upload.headers` 그대로 사용, 실패 시 "글로 완료" 경로가 항상 동작하도록 유지 |
| AI 정리 턴 지연(요청당 상한 30초, 세션은 여러 턴) | 사용자가 이탈하거나 중복 클릭 | 단계 칩으로 진행 표시, 버튼 비활성화, `WORKING` 연속 2회 후 `finish`, "이대로 카드 받기"로 언제든 폼 진입, 실패 시 즉시 직접 작성으로 전환 |
| 에이전트가 불안정해 레벨 0(`AGENT_MODE=pipeline`)으로 출시 | 질문 말풍선이 나오지 않음 | 화면은 `status`로만 분기하고 생성 응답의 `READY`를 바로 처리. 레벨 0 경로를 MSW 기본값으로 둔다 |
| 폴링 상태 판정 오류 | 완료 화면이 안 뜨거나 잘못된 문구 | 판정 함수를 분리해 단위 테스트, 서버 값으로만 문구 결정 |
| 로컬 백엔드와 실제 AWS의 동작 차이(JWT authorizer, CORS preflight, presigned 서명 강제, 동시성, 회원가입 자동 확인) | 로컬에서 되던 흐름이 `dev` 스택에서 실패 | 로컬 결과를 계약·인수 근거로 쓰지 않고, S1–S5마다 `dev` 스택에서 다시 확인(4.1) |
| 로컬 Cognito에 Amplify가 붙지 않음(Q8) | 로컬 백엔드 모드에서 로그인 불가 | `VITE_AUTH_MODE=dev-jwt`로 전환, 로그인 화면은 MSW와 `dev` 스택에서 확인 |
| SPA 새로고침 시 CloudFront 404 | 배포 환경에서 하위 경로 접근 실패 | 백엔드에 403/404 → `/index.html` 설정 요청(백엔드 BE-INFRA-02) |
| 시안 색상 미확정 | 막바지 디자인 수정 | 토큰으로만 색을 쓰고 값만 교체 |

## 9. 완료 체크리스트

[`REQUIREMENTS.md`](./REQUIREMENTS.md) §6 인수 기준(= `MVP.md` §10)을 모두 만족하면 완료로 본다.
