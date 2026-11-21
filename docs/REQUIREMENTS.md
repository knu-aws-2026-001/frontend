# 이어온 하루 — Frontend 요구사항 명세서

| 항목 | 내용 |
|---|---|
| 프로젝트 | 취소된 하루를 양도합니다 (AWS Skill Training 1조) |
| 서비스 | 이어온 하루 (모바일 웹) |
| 대상 저장소 | `knu-aws-2026-001/frontend` |
| 기준 문서 | [`MVP.md`](../MVP.md) (화면 요구사항), [backend `MVP.md`](https://github.com/knu-aws-2026-001/backend/blob/main/MVP.md) (API 계약의 Source of Truth) |
| 범위 | MVP P0 |

이 문서는 `MVP.md`의 내용을 구현·검증 가능한 요구사항 단위(ID 부여)로 정리한 것이다. 화면별 세부 문구와 분기 순서는 `MVP.md` §6을 따른다. API 요청·응답 형태가 이 문서와 백엔드 `MVP.md` §7이 다르면 백엔드 문서가 우선한다.

---

## 1. 목적과 범위

계획자가 하지 못한 산책 계획을 남기고, 같은 동네의 실행자 한 명이 이어받아 다녀온 뒤 방문 기록(글 + 선택 사진 1장)을 돌려주며, 계획자가 한 줄 답장을 보내는 모바일 웹 화면을 만든다.

### 1.1 포함 (P0)
로그인/회원가입, 둘러보기, 하루 들여다보기·이어받기(경합 안내 포함), 하루 남기기(사연 → AI 정리(질문 최대 2개, 건너뛰기 가능) → 폼 → 공개, AI 실패 시 직접 작성), 내 하루·이어받기 포기, 방문 기록(글 + 사진 1장, 처리 대기·실패·교체, 글로 완료), 수신함, 돌아온 하루·한 줄 답장, 작성 소요 시간 전송.

### 1.2 이후 (P1, 이 명세의 필수 요건 아님)
실행자의 답장 확인, 다녀온 이야기 질문 도우미(조건부, 시작 조건은 `MVP.md` §3), 하루 대표 사진(MVP는 플레이스홀더)·내가 남긴 하루 목록(이 둘은 이번 일정에서 하지 않음).

### 1.3 제외
지도·위치 탐색, 여러 동네, 푸시/이메일 알림, 운영자 화면, 다중 사진, 채팅, 소셜 로그인.

## 2. 사용자와 핵심 흐름

| 역할 | 데모 계정 | 흐름 |
|---|---|---|
| 계획자 | 지연 | 사연 입력 → AI와 함께 정리(질문에 답하기)·수정 → 공개 → 수신함 → 돌아온 하루 → 한 줄 답장 |
| 실행자 | 수현, 민준(경합 시연) | 둘러보기 → 들여다보기 → 이어받기 → 방문 기록(+사진) → 전송 |

## 3. 기술 요구사항

| ID | 요구사항 |
|---|---|
| FE-TECH-01 | React 18 + TypeScript + Vite, React Router, TanStack Query를 쓴다. |
| FE-TECH-02 | 인증은 `aws-amplify/auth`(Auth 모듈만)로 Cognito User Pool에 연결한다. API 호출에는 **ID 토큰**을 `Authorization: Bearer`로 붙인다(닉네임 클레임 때문). |
| FE-TECH-03 | 환경 변수 `VITE_API_BASE_URL`, `VITE_AWS_REGION`, `VITE_COGNITO_USER_POOL_ID`, `VITE_COGNITO_CLIENT_ID`, `VITE_USE_MOCK`. 값은 `.env.local`에 두고 저장소에는 `.env.example`만 올린다. |
| FE-TECH-04 | `VITE_USE_MOCK=true`이면 MSW가 백엔드 계약과 같은 형태로 응답한다. 목업 데이터는 시안 인물(지연·수현·민준)을 쓴다. AI 정리 세션은 상태 머신(`WORKING`·`ASKING`·`READY`·`FAILED`)으로 흉내 내고, 쿼리 파라미터로 경로를 고른다([`DEVELOPMENT_PLAN.md`](./DEVELOPMENT_PLAN.md) 2장). |
| FE-TECH-05 | API 타입은 백엔드 `MVP.md` §7과 같은 이름으로 `src/api/types.ts`에 정의한다. |
| FE-TECH-06 | 스타일은 CSS Modules 또는 Tailwind 중 팀 합의로 하나만 쓴다. 디자인 토큰은 `src/styles`에 CSS 변수로 둔다. |
| FE-TECH-07 | 빌드 산출물은 백엔드 SAM 스택이 만든 S3 웹 버킷에 sync하고 CloudFront 캐시를 무효화한다. |

## 4. 기능 요구사항

근거는 `MVP.md` 절 번호와 화면 번호(§5 표의 `#`).

### 4.1 공통 (COMMON)

| ID | 요구사항 | 근거 |
|---|---|---|
| FE-COM-01 | 하단 탭(둘러보기 / 내 하루 / 수신함)은 목록형 화면에만 보인다. | §5 |
| FE-COM-02 | 헤더 오른쪽 편지 아이콘에 `GET /me`의 `unreadInboxCount > 0`이면 주황 점을 표시한다. | §6.6 |
| FE-COM-03 | 401 `UNAUTHORIZED`를 받으면 `/login?redirect={현재 경로}`로 보내고, 로그인 후 그 경로로 돌아온다. | §5 |
| FE-COM-04 | 공통 에러 처리: `NOT_FOUND` → "찾을 수 없어요" 후 둘러보기/수신함, `FORBIDDEN` → 안내 후 둘러보기, `VALIDATION_ERROR` → `details`의 경로별 메시지를 해당 입력칸에 표시, `TRY_AGAIN` → 같은 요청 1회 재시도 후 실패 시 토스트. | §7 |
| FE-COM-05 | 모든 API 호출은 토큰을 붙이는 공통 클라이언트를 거친다. 토큰 없이 API를 호출하는 경로가 없어야 한다. | §10 |
| FE-COM-06 | 서버 시간(UTC ISO 8601)은 한국 시간(Asia/Seoul)으로 표시한다. | 백엔드 §7 |

### 4.2 로그인·회원가입 (AUTH) — 화면 0

| ID | 요구사항 |
|---|---|
| FE-AUTH-01 | 이메일·비밀번호로 로그인한다. |
| FE-AUTH-02 | 회원가입은 이메일·비밀번호·닉네임(필수)을 받는다. 백엔드가 즉시 자동 확인하므로 인증 코드 화면이 없고, 가입 성공 후 바로 로그인한다. |
| FE-AUTH-03 | 로그인하지 않은 사용자가 보호된 경로에 들어오면 로그인 화면으로 보낸다(FE-COM-03과 같은 redirect 규칙). |

### 4.3 둘러보기 (BROWSE) — 화면 1, §6.1

| ID | 요구사항 |
|---|---|
| FE-BRW-01 | `GET /plans?status=OPEN` 결과를 최신순 카드로 표시한다. 카드: "이어받기 가능" 배지, 제목, `summary` 2줄 말줄임, `약 {n}분 · 장소 {k}곳`. |
| FE-BRW-02 | 목록이 비면 "아직 남겨진 하루가 없어요"와 하루 남기기 유도 버튼을 보여준다. |
| FE-BRW-03 | "취소된 하루 남기기" 버튼으로 사연 입력 화면에 간다. |
| FE-BRW-04 | 내가 남긴 하루도 목록에 보일 수 있다. 이어받기 가능 여부는 상세의 `claimable`로만 판단한다. |

### 4.4 하루 들여다보기·이어받기 (PLAN) — 화면 2, 2-1, §6.2

| ID | 요구사항 |
|---|---|
| FE-PLN-01 | 계획자 닉네임, 취소한 이유(`reason`), 산책 동선 타임라인(장소·하고 싶었던 일·고른 이유)을 보여준다. |
| FE-PLN-02 | 상세 분기는 위에서부터 먼저 맞는 하나만 적용한다: 404 → "이 하루는 더 이상 볼 수 없어요" / `myClaim.status === 'ACTIVE'` → "내가 이어받은 하루" + "내 하루로 가기" / `isOwner` → "내가 남긴 하루" + 상태만 / `claimable` → "이 하루 이어받기" / `status === 'CLAIMED'` → 2-1 안내 / `COMPLETED` → 완료 배지 읽기 전용 / 그 외 → "다른 하루 둘러보기". |
| FE-PLN-03 | 이어받기는 `POST /plans/{planId}/claims`. 201 또는 200이면 `/me`로 이동한다. 요청 중에는 버튼을 비활성화한다. |
| FE-PLN-04 | 409 `ALREADY_CLAIMED`면 페이지 이동 없이 같은 화면에 2-1 안내 박스("이미 다른 사람이 이어받았어요" + "다른 하루 둘러보기")를 띄운다. |
| FE-PLN-05 | 409 `CANNOT_CLAIM_OWN_PLAN`은 토스트로 처리한다. |

### 4.5 하루 남기기 (CREATE) — 화면 3 ~ 3-4, §6.3

| ID | 요구사항 |
|---|---|
| FE-CRT-01 | 사연 입력(1–1,000자)과 "집 주소나 개인 정보는 적지 말아주세요" 안내, 국외 처리 안내("AI 정리 요청은 국외 리전에서 처리될 수 있어요", Q6)를 보여준다. "AI와 함께 정리하기"와 "직접 작성하기" 두 버튼을 둔다. |
| FE-CRT-02 | AI 경로는 세션 API를 쓴다. `POST /ai/card-sessions { story }` → `SessionView`. `sessionId`와 마지막 `turn`을 보관하고, 다음 턴은 `POST /ai/card-sessions/{sessionId}/turns { expectedTurn: 마지막 turn, answer?, finish? }`로 보낸다. 화면 분기는 응답 `status`(`WORKING`·`ASKING`·`READY`·`FAILED`)로만 한다. 생성 응답이 바로 `READY`인 경우(레벨 0)도 처리한다. `READY`면 `card`를 폼 초기값으로 그대로 쓰고(`null`은 빈칸) 모드는 `AI_EDITED`. |
| FE-CRT-03 | AI와 함께 정리하기 화면(3-1): `trace[]` 단계 칩(`step` 순서대로 켬), 질문 말풍선(`ASKING`), "이대로 카드 받기", 카드 미리보기(`flags`에 `NOT_IN_SOURCE`가 없으면 "원문 대조 통과" 배지), "AI가 바꾼 것" 메모(`notes[].reason` + 출처 칩 `STORY` → "원문", `ANSWER` → "내 답변"), "폼에서 다듬기". `flags[].path` 연결은 폼(3-2)에 그대로 둔다: `NOT_IN_SOURCE` → "원문에 없던 내용이에요, 확인해 주세요", `MISSING` → "직접 채워주세요", `PRIVACY` → "공개하기 어려운 정보일 수 있어요". |
| FE-CRT-04 | AI 실패(`FAILED`/503 `AI_UNAVAILABLE`, 502 `AI_INVALID_OUTPUT`, 429 `AI_QUOTA_EXCEEDED`, 세션 404, 타임아웃, 5xx) 시 사연을 유지하고 "AI 카드를 만들지 못했어요. 직접 정리해 주세요" 안내 후 빈 폼으로 이동한다. 폼 상단에 사연을 참고용으로 보여준다. 모드는 `MANUAL`이고 `aiSessionId`는 보내지 않는다. AI 세션 요청의 클라이언트 타임아웃은 서버 상한 30초보다 길게 잡고, `turns`의 네트워크 오류·타임아웃은 `GET`으로 한 번 재동기화한 뒤 판단한다. |
| FE-CRT-05 | 직접 작성 경로: AI 호출 없이 폼으로 가고, `summary` 기본값을 사연 앞부분(120자 이내)으로 채운다. 모드는 `MANUAL`. |
| FE-CRT-06 | 폼 검증: 제목 1–40, `summary` 1–120, 취소한 이유 1–300, 예상 소요 시간 10–240분, 장소 1–3곳(이름 1–30, 하고 싶었던 일 1–60, 고른 이유 0–120), 방문 질문 1–80(1개 필수). 사연은 두 모드 모두 필수. |
| FE-CRT-07 | 장소 추가 화면(3-3)에서 장소 이름·하고 싶었던 일·고른 이유를 입력해 동선에 추가한다(최대 3곳). |
| FE-CRT-08 | 작성 상태(사연, `sessionId`, 마지막 `turn`, READY 카드, `flags`, `notes`, 폼 값, `startedAt`, `idempotencyKey`)를 하나로 묶어 `sessionStorage`에 저장한다. 하위 경로 이동·새로고침에도 유지하고, 공개 성공 시 삭제한다. |
| FE-CRT-09 | `startedAt`은 사연 입력 화면에 처음 들어올 때 한 번만 기록한다. 공개 요청에 `authoring { mode, durationMs, aiSessionId?, story }`를 포함한다. `aiSessionId`는 `AI_EDITED`일 때만 넣는다. |
| FE-CRT-10 | 공개는 `POST /plans` + `Idempotency-Key`. 키는 공개 버튼을 누를 때 본문과 함께 만들고, 같은 본문 재전송 때만 재사용한다. 폼이 바뀌면 새 UUID. 409 `IDEMPOTENCY_CONFLICT`면 새 키로 1회 재요청한다. |
| FE-CRT-11 | 공개 성공 시 공개 완료 화면(3-4, `/plans/:planId/published`)과 "둘러보기로 가기"를 보여준다. |
| FE-CRT-12 | 질문 답변(`ASKING`): `question.text`와 `reason`을 말풍선으로 보여준다. 선택지 → `answer { questionId, choice }`, `allowFreeText === true`일 때만 직접 입력(1–100자) → `answer { questionId, text }`, "건너뛰기" → `answer { questionId, skipped: true }`(그 필드는 `MISSING`으로 남음). "이대로 카드 받기" → `finish: true`(`answer` 없이 보내면 현재 질문은 건너뜀). |
| FE-CRT-13 | 턴 진행·재동기화: `WORKING`이면 `answer` 없이 `turns`를 자동으로 보내고, `WORKING`을 연속 2회 받으면 다음 요청에 `finish: true`를 넣고, 그 뒤 `WORKING`이 와도 `finish: true`로 이어서 보낸다. 409 `AI_SESSION_BUSY` → 2초 뒤 `GET /ai/card-sessions/{sessionId}`로 다시 맞춘다. 재동기화 `GET`이 보낸 `expectedTurn`과 같은 `turn`의 `WORKING`이면 2초 간격으로 `GET`을 반복하고, 이 응답은 연속 `WORKING` 횟수에 넣지 않는다(잠금 시간 30초가 지나도 같으면 `turns`를 다시 보낸다). 409 `AI_SESSION_CLOSED` → `GET` 후 현재 상태를 보여준다. 3-1 진입 시 `sessionId`가 있고 READY 카드가 없으면 `GET`으로 세션을 복구한다(새로고침 포함). |
| FE-CRT-14 | 메모 "다시 넣기": `action === 'REMOVED'` 메모에 버튼을 두고, 누르면 그 항목을 폼에 되살리며 `PRIVACY` 또는 `NOT_IN_SOURCE` 경고를 함께 붙인다(복원할 값·유형·위치는 Q7). |

### 4.6 내 하루·포기 (ME) — 화면 4, 4-1, §6.4

| ID | 요구사항 |
|---|---|
| FE-ME-01 | `GET /me/claims?status=ACTIVE`로 이어받은 하루를 보여준다. 없으면 둘러보기 유도. |
| FE-ME-02 | 기한 문구: 기한 전 "{M}월 {D}일까지 다녀온 이야기를 보내주세요.", 기한 후 "기한이 지났어요. 다른 사람이 이어받기 전까지는 보낼 수 있어요." |
| FE-ME-03 | "다녀온 이야기 쓰기"는 `record.photoStatus`에 따라 이동한다: 기록 없음/`NONE` → 작성, `UPLOAD_PENDING`/`PROCESSING` → 보내는 중, `FAILED` → 사진 처리 실패. |
| FE-ME-04 | 포기: 확인 화면("계속 이어가기" / "이어받기 포기") → `POST /claims/{claimId}/abandon` → 둘러보기. 409 `CLAIM_NOT_ACTIVE`면 "이미 종료된 이어받기예요" 안내 후 새로고침. |

### 4.7 방문 기록·사진·완료 (REC) — 화면 5 ~ 5-4, 8, §6.5

| ID | 요구사항 |
|---|---|
| FE-REC-01 | 작성 화면 상단에 계획 제목, "{계획자 닉네임}에게", 하루 카드의 방문 질문을 보여준다. |
| FE-REC-02 | 진입 시 `GET /claims/{claimId}/record`로 서버 글을 먼저 채우고, 없을 때만 `localStorage` 임시 글을 쓴다. 임시 글은 입력 중 저장하고, 전송 완료·종료 안내 시 삭제한다. |
| FE-REC-03 | 글 1–1,000자. 사진은 JPEG/PNG, 10MB 이하, 1장(선택). 형식·크기는 클라이언트에서 먼저 검사한다. |
| FE-REC-04 | 사진 있음: `PUT /record { letter, photo: { contentType, size } }` → 응답 `upload`로 `fetch(upload.url, { method: 'PUT', body: file, headers: upload.headers })` → 보내는 중 화면. 업로드 실패는 사진 처리 실패와 같게 처리한다. |
| FE-REC-05 | 사진 없음 또는 "글로 완료하기": 항상 `PUT /record { letter }` 후 `POST /complete { mode: 'TEXT_ONLY' }`. 400 `RECORD_REQUIRED`면 `PUT /record` 후 1회 재시도. 네트워크 오류·503 `TRY_AGAIN`은 같은 요청 1회 자동 재시도. |
| FE-REC-06 | 보내는 중(5-1): `GET /record`를 2초 간격으로 폴링한다. 판정 순서: `delivered` → 완료 / `claimStatus`가 `ABANDONED`·`EXPIRED` 또는 403·404·409 → 화면 8 / `photoStatus === 'FAILED'` → 실패 화면 / 그 외 계속 폴링. |
| FE-REC-07 | 폴링 60초가 지나도 처리 중이면 "글로 완료하기"를 강조한다. `UPLOAD_PENDING`이 30초 넘게 그대로면(새로고침 후 돌아온 경우 포함) 실패 화면을 보여준다. |
| FE-REC-08 | 실패 화면(5-2): 작성한 글을 유지하고 "사진 바꾸기" / "글로 완료하기"를 제공한다. |
| FE-REC-09 | 사진 교체(5-3): `POST /record/photo { contentType, size }` → 새 URL로 업로드 → 보내는 중. |
| FE-REC-10 | 전송 완료(5-4) 문구는 서버 결과로 정한다: `complete` 응답의 `hasPhoto`, 폴링 경로는 `photoStatus === 'READY'`. 새로고침 시 `GET /record`로 다시 판단한다. |
| FE-REC-11 | 종료된 이어받기(화면 8): `PUT /record`, `POST /record/photo`, `POST /complete`의 409 `CLAIM_NOT_ACTIVE` 또는 폴링 판정에서 띄운다. "이 이어받기는 종료되었어요" + 작성 글 복사 + "다른 하루 둘러보기". `photoStatus = DISCARDED`는 종료 판단에 쓰지 않는다. |

### 4.8 수신함·돌아온 하루·답장 (INBOX) — 화면 6 ~ 7-1, §6.6

| ID | 요구사항 |
|---|---|
| FE-INB-01 | `GET /inbox`를 최신순으로 표시한다. `read === false`면 "새 편지" 배지 + 주황 점, 아니면 "읽은 기록". `hasPhoto`에 따라 "사진과 방문 기록을 보냈어요" / "방문 기록을 보냈어요". |
| FE-INB-02 | 상세 진입 시 `POST /inbox/{itemId}/read`를 호출하고 수신함 목록과 `GET /me` 캐시를 무효화한다. |
| FE-INB-03 | 상세: 사진(있으면, `record.photoUrl`) + `photoCaption`, "내가 남긴 계획"(베이지, `plan.summary`) / "{실행자}의 방문 기록"(연녹색, `record.letter`) 2단 카드. |
| FE-INB-04 | 사진 URL이 만료돼 이미지 로드가 실패하면 상세를 다시 조회한다(1회). |
| FE-INB-05 | 한 줄 답장 1–100자 → `POST /inbox/{itemId}/reply`. 성공 또는 409 `REPLY_ALREADY_SENT`면 "보낸 답장" 읽기 전용(7-1)과 "수신함으로" 버튼을 보여준다. 이미 `reply`가 있는 항목은 처음부터 7-1로 보인다. |

## 5. 비기능 요구사항

| ID | 분류 | 요구사항 |
|---|---|---|
| FE-NFR-01 | 반응형 | 360–480px 폭에서 시안과 같은 배치. 그보다 넓으면 최대 폭 480px 가운데 정렬. |
| FE-NFR-02 | 디자인 | `MVP.md` §8 토큰과 레이아웃 패턴(큰 제목 2줄 + 회색 보조 문구 + 하단 고정 주 버튼, 번호 원 타임라인, 상태 배지)을 공통 컴포넌트로 만든다. 시안이 없는 화면(0, 3, 3-1, 8)도 같은 패턴을 따른다. 3-1(AI와 함께 정리하기)은 큰 제목 + 단계 칩 + 질문 말풍선 + 카드 미리보기·메모 + 하단 고정 주 버튼("폼에서 다듬기"), 보조 버튼 "이대로 카드 받기"로 구성한다. |
| FE-NFR-03 | 중복 요청 방지 | 상태를 바꾸는 버튼(이어받기, 공개, 포기, 완료, 답장, AI 질문 답변·건너뛰기)은 요청 중 비활성화한다. "이대로 카드 받기"는 `WORKING` 자동 요청 중에도 누를 수 있고, 다음 요청에 `finish`를 예약만 한다(중복 예약 방지). |
| FE-NFR-04 | 데이터 보존 | 하루 남기기 입력과 AI 세션 ID는 새로고침에도 유지되고(sessionStorage, 세션 상태는 `GET`으로 복구), 방문 기록 글은 실패·새로고침에도 유지된다(서버 저장 + localStorage). |
| FE-NFR-05 | 보안 | 토큰은 Amplify가 관리하는 저장소에만 둔다. 로그·에러 메시지에 토큰과 presigned URL을 출력하지 않는다. 사용자 입력과 AI 응답(질문문·선택지·메모·단계 칩)은 React 기본 이스케이프로만 렌더링한다(`dangerouslySetInnerHTML` 금지). |
| FE-NFR-06 | 접근성 | 입력칸마다 라벨, 에러 메시지는 입력칸과 연결(`aria-describedby`), 버튼은 키보드로 동작. |
| FE-NFR-07 | 계약 일치 | MSW 핸들러 응답은 백엔드 `MVP.md` §7 예시와 같은 형태다. 계약이 바뀌면 타입과 핸들러를 같은 PR에서 고친다. |

## 6. 인수 기준

`MVP.md` §10 완료 기준을 그대로 쓴다.

- [ ] 데모 계정 3개로 90초 시연 시나리오가 끊김 없이 동작
- [ ] 수현·민준 동시 이어받기 시 한 명만 확정, 다른 쪽은 "이미 다른 사람이 이어받았어요"
- [ ] 완료 버튼을 여러 번 눌러도 수신함 항목 1건
- [ ] 사진 처리 실패 시 글 유지, 사진 교체 또는 글로 완료 가능
- [ ] 사진 없이 "글로 완료" 시 바로 전송(`PUT /record` → `complete`)
- [ ] AI 정리 실패(`FAILED`, 한도 초과 포함) 시 사연 유지, 직접 작성으로 전환
- [ ] AI 정리 중 새로고침해도 세션 복구
- [ ] 질문 건너뛰기와 "이대로 카드 받기"로 언제든 폼 진입
- [ ] 공개 요청에 `authoring.mode`, `durationMs`, `story` 포함
- [ ] 하루 남기기 도중 새로고침해도 입력값과 측정 시작 시각 유지
- [ ] 360–480px 폭에서 시안과 같은 배치
- [ ] CloudFront 배포 URL에서 동작, 토큰 없이 API를 호출하는 경로 없음

## 7. 확인이 필요한 사항

| # | 내용 | 제안 |
|---|---|---|
| Q1 | 둘러보기 페이지네이션(`nextCursor`) 처리가 정해지지 않았다. | MVP는 첫 페이지만 표시. 백엔드는 `nextCursor: null` 고정(백엔드 명세 Q3). |
| Q2 | 공개 완료 화면(3-4) 경로가 `/plans/:planId/published`이므로 공개 직후 작성 상태를 지워도 이 화면이 렌더링되는지. | 공개 응답의 `plan`을 라우터 state로 넘기고, 새로고침 시 `GET /plans/{planId}`로 다시 읽는다. |
| Q3 | CSS Modules / Tailwind 선택. | 1단계 시작 전에 팀 합의. |
| Q4 | SPA 직접 접근(예: `/inbox/abc` 새로고침) 시 CloudFront가 `index.html`을 돌려줘야 한다. | 백엔드 SAM에서 403/404 → `/index.html` 설정(백엔드 BE-INFRA-02). |
| Q5 | 시안 색상은 추정값이다. | 디자인 확정 시 토큰 값만 교체. |
| Q6 | 국외 처리 안내 문구와 위치. 경량 모델은 서울에서 Global CRIS로만 호출되어 AI 정리 요청이 국외 리전에서 처리될 수 있다(백엔드 명세 Q7). | 사연 입력 화면(3)의 개인 정보 안내 아래에 둔다. 문구는 백엔드 시범 운영 동의 문구와 맞춘다. |
| Q7 | "다시 넣기"로 복원할 원래 값과 경고 유형. `SessionView.notes`에는 `path`·`action`·`reason`·`source`만 있고 뺀 항목의 원래 값과 뺀 이유의 유형(`PRIVACY`/`NOT_IN_SOURCE`)이 없다. 또 뺀 장소는 `places`에서 빠지므로 `notes[].path`의 인덱스 기준이 정해지지 않았다(백엔드 명세 Q13). | 백엔드 `MVP.md` §7.2 `notes`에 원래 값과 유형을 담는 필드를 추가하도록 계약 변경을 요청한다. `REMOVED` 메모의 `path`는 제거 전 인덱스 기준, 복원은 `places` 끝에 추가하는 안으로 맞춘다. 확정 전까지 FE-CRT-14는 MSW로만 구현한다. |
