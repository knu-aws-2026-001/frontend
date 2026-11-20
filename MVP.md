# 이어온 하루 — Frontend MVP

> 프로젝트: **취소된 하루를 양도합니다** (AWS Skill Training 1조)
> 서비스명: **이어온 하루** — 취소된 동네 산책을 이어받고, 그날의 이야기를 돌려주는 모바일 웹

이 문서는 프론트엔드 저장소에서 구현할 MVP 범위를 정리한다. API 상세 계약(요청/응답 스키마, 에러 코드)은 백엔드 저장소의 [`MVP.md`](https://github.com/knu-aws-2026-001/backend/blob/main/MVP.md)를 기준으로 하며, 이 문서에는 화면 구현에 필요한 요약만 둔다.

참고 자료 (팀 공유 PDF)
- `aws1조_초안.pdf` — 문제 정의, 목표, 아키텍처, 주요 기능, 데모 계획
- `이어온 하루-동네산책.pdf` — **최신 화면 시안 (동네 산책 버전, MVP 기준)**
- `이어온 하루.pdf` — 이전 화면 시안 (캠퍼스 산책 버전). 레이아웃 참고용이며 문구는 동네 버전을 따른다.

---

## 1. 한 줄 요약

계획자가 하지 못한 산책 계획(공개 장소 1~3곳)을 남기면, 같은 동네의 실행자 한 명이 이어받아 다녀오고, 방문 기록(글 + 선택 사진 1장)이 계획자의 수신함에 도착한다. 계획자는 한 줄 답장을 보낸다.

## 2. 사용자와 핵심 흐름

| 역할 | 설명 | 데모 계정 |
|---|---|---|
| 계획자 | 이사·일정 변경으로 산책을 하지 못한 사람 | 지연 |
| 실행자 | 새로 이사 왔거나 주말에 근처를 둘러보고 싶은 사람 | 수현 (+ 인수 경합 시연용 민준) |

```
[지연] 사연 입력 → AI 하루 카드 확인·수정 → 하루 공개
[수현] 둘러보기 → 하루 들여다보기 → 이어받기 → (산책) → 방문 기록 작성(+사진) → 전송
[지연] 수신함 알림 → 돌아온 하루 읽기 → 한 줄 답장
```

## 3. MVP 범위

### 포함 (P0)
1. 로그인/회원가입 (Cognito, 닉네임 필수)
2. 둘러보기: 이어받기 가능한 하루 목록
3. 하루 들여다보기: 계획 상세, 이어받기, 경합 실패 안내
4. 하루 남기기: 사연 입력 → AI 하루 카드 초안 → 편집 → 공개 (AI 실패 시 직접 작성으로 전환)
5. 내 하루: 이어받은 하루와 기한, 이어받기 포기
6. 방문 기록: 글 작성 + 선택 사진 1장 업로드, 사진 처리 대기/실패/교체, 글로 완료
7. 수신함: 새 편지 / 읽은 기록 구분, 미확인 표시
8. 돌아온 하루: 계획과 방문 기록 나란히 보기, 한 줄 답장
9. 작성 소요 시간 측정 값 전송 (직접 작성 vs AI 초안 편집 비교용)

### 이후 (P1, 시간이 남으면)
- 실행자가 계획자의 답장을 내 하루에서 확인
- 하루 대표 사진 (시안의 커버 이미지). MVP에서는 기본 일러스트/플레이스홀더로 대체
- 내가 남긴 하루 목록과 상태 보기

### 제외
- 지도·위치 기반 탐색, 여러 동네 선택 (MVP는 한 동네 고정)
- 푸시/이메일 알림 (수신함 + 헤더 배지만 제공)
- 문화공간 운영자용 화면 (시범 운영에서 수요만 확인)
- 다중 사진, 채팅, 소셜 로그인

## 4. 기술 스택 (권장)

| 영역 | 선택 | 비고 |
|---|---|---|
| 프레임워크 | React 18 + TypeScript + Vite | |
| 라우팅 | React Router | |
| 서버 상태 | TanStack Query | 폴링(사진 처리 상태)에 `refetchInterval` 사용 |
| 인증 | AWS Amplify `aws-amplify/auth` (Auth 모듈만) | Cognito User Pool, ID 토큰을 `Authorization` 헤더로 전달 |
| 스타일 | CSS Modules 또는 Tailwind (팀 합의) | 모바일 우선, 최대 폭 480px 가운데 정렬 |
| API 목업 | MSW | 백엔드 배포 전 계약 기준으로 개발 |
| 배포 | S3(비공개) + CloudFront | 버킷·배포는 백엔드 SAM 스택이 생성, 이 저장소는 빌드 산출물 sync |

환경 변수
```
VITE_API_BASE_URL=
VITE_AWS_REGION=ap-northeast-2
VITE_COGNITO_USER_POOL_ID=
VITE_COGNITO_CLIENT_ID=
VITE_USE_MOCK=false
```

## 5. 화면 목록

하단 탭: **둘러보기 / 내 하루 / 수신함**. 상단 헤더 오른쪽 편지 아이콘에 미확인 수신함 점 표시.

| # | 화면 | 경로 (예시) | 시안 | 주요 동작 |
|---|---|---|---|---|
| 0 | 로그인 / 회원가입 | `/login` | 없음 | 이메일·비밀번호·닉네임. 백엔드가 가입 즉시 자동 확인하므로 인증 코드 단계 없음 |
| 1 | 둘러보기 | `/` | 있음 | 이어받기 가능한 하루 카드 목록(제목, `summary`, 소요 시간, 장소 수), "취소된 하루 남기기" 버튼 |
| 2 | 하루 들여다보기 | `/plans/:planId` | 있음 | 계획자 이름·취소한 이유(`reason`), 산책 동선(장소·하고 싶었던 일·고른 이유), "이 하루 이어받기" |
| 2-1 | 이미 이어받은 하루 | 같은 경로, 상태 분기 | 있음 | 409 `ALREADY_CLAIMED`, 또는 `status === 'CLAIMED'`이고 내 것이 아닐 때 안내 + "다른 하루 둘러보기" |
| 3 | 사연 입력 | `/plans/new` | **없음** | 자유 서술 입력, "AI로 하루 카드 만들기" / "직접 작성하기" |
| 3-1 | AI 하루 카드 확인 | `/plans/new/card` | **없음** | 활동 목록·계획 이유·방문 질문 편집, 원문에 없던 항목 강조 |
| 3-2 | 하루 남기기 (폼) | `/plans/new/form` | 있음 | 제목, 하고 싶었던 일 요약, 취소한 이유, 예상 소요 시간, 산책 동선(1~3곳), 방문 질문 |
| 3-3 | 장소 추가 | `/plans/new/place` | 있음 | 장소 이름, 하고 싶었던 일, 이 장소를 고른 이유 |
| 3-4 | 공개 완료 | `/plans/:planId/published` | 있음 | "둘러보기로 가기" |
| 4 | 내 하루 (이어받음) | `/me` | 있음 | 기한("11월 21일까지…"), 동선, "다녀온 이야기 쓰기", "이어받기 포기" |
| 4-1 | 이어받기 포기 확인 | `/claims/:claimId/abandon` | 있음 | "계속 이어가기" / "이어받기 포기" |
| 5 | 그날의 이야기 (방문 기록 작성) | `/claims/:claimId/record` | 있음 | "{계획자}에게" 편지, 하루 카드의 방문 질문 표시, 사진 1장 선택(선택 사항) |
| 5-1 | 보내는 중 (사진 처리 중) | `/claims/:claimId/sending` | 있음 | 진행 표시, 상태 폴링, "글로 완료하기" |
| 5-2 | 사진 처리 실패 | 같은 경로, 상태 분기 | 있음 | "사진 바꾸기" / "글로 완료하기", 작성한 글 유지 |
| 5-3 | 사진 교체 | `/claims/:claimId/record/photo` | 있음 | 새 사진 첨부, "글로 완료하기" |
| 5-4 | 전송 완료 | `/claims/:claimId/done` | 있음 | 사진+글 / 글만 두 가지 문구, "둘러보기로 돌아가기" |
| 6 | 수신함 | `/inbox` | 있음 | 새 편지 / 읽은 기록 배지, 사진 유무에 따라 문구 다름 |
| 7 | 돌아온 하루 | `/inbox/:itemId` | 있음 | 사진, "내가 남긴 계획" vs "{실행자}의 방문 기록", 한 줄 답장 입력 |
| 7-1 | 답장 보낸 상태 | 같은 경로, 상태 분기 | 있음 | "보낸 답장" 읽기 전용, "수신함으로" |
| 8 | 종료된 이어받기 안내 | 상태 분기 | 없음 | 포기·기한 만료로 이어받기가 끝났을 때. 작성한 글 복사 가능, "다른 하루 둘러보기" |

시안이 없는 화면(0, 3, 3-1, 8)은 기존 시안의 톤(큰 제목 + 보조 문구 + 하단 고정 주 버튼)을 따라 만든다.

공통 인증 처리: 401(`UNAUTHORIZED`)을 받으면 현재 경로를 `/login?redirect=...`로 넘기고, 로그인 후 그 경로로 돌아온다.

## 6. 화면별 상세 요구사항

### 6.1 둘러보기
- `GET /plans?status=OPEN` 결과를 최신순 카드로 표시. 빈 목록이면 "아직 남겨진 하루가 없어요" + 하루 남기기 유도.
- 카드: 상태 배지(이어받기 가능), 제목, `summary` 2줄 말줄임, `약 {n}분 · 장소 {k}곳`. (기한이 지난 이어받기는 서버가 `OPEN`으로 내려준다.)
- 내가 남긴 하루도 목록에 보일 수 있다. 이어받기 가능 여부는 상세의 `claimable`로 판단한다.

### 6.2 하루 들여다보기 / 이어받기
상세 화면 분기 (위에서부터 먼저 맞는 것 하나):
1. 404 `NOT_FOUND` → "이 하루는 더 이상 볼 수 없어요" + 둘러보기
2. `myClaim?.status === 'ACTIVE'` → "내가 이어받은 하루" 배지 + "내 하루로 가기" 버튼
3. `isOwner` → "내가 남긴 하루" 배지, 상태(공개/이어받음/완료)만 표시, 버튼 없음
4. `claimable` → "이 하루 이어받기" 버튼
5. `status === 'CLAIMED'` → 2-1 안내
6. `status === 'COMPLETED'` → 완료 배지, 읽기 전용
7. 그 외 → 동선만 보여주고 "다른 하루 둘러보기" 버튼 (방어용)

`myClaim`은 진행 중(`ACTIVE`)이거나 완료(`COMPLETED`)한 내 이어받기만 담긴다. 포기했던 하루는 다시 `claimable`이 될 수 있다.

이어받기 클릭 → `POST /plans/{planId}/claims`
- 201 또는 200(같은 사용자의 중복 요청 — 기존 이어받기 반환): 내 하루(`/me`)로 이동.
- 409 `ALREADY_CLAIMED`: 2-1 상태로 전환 (페이지 이동 없이 같은 화면에서 안내 박스 표시).
- 409 `CANNOT_CLAIM_OWN_PLAN`: 버튼 비노출이 정상이지만 방어적으로 토스트 처리.
- 버튼은 요청 중 비활성화.

### 6.3 하루 남기기 (AI 하루 카드 포함)
흐름: **사연 입력 → (AI 초안 생성) → 카드 확인 → 폼 편집 → 공개**

- 사연 입력(최대 1,000자). 공개 장소만 적도록 안내 문구 표시 ("집 주소나 개인 정보는 적지 말아주세요").
- "AI로 하루 카드 만들기" → `POST /ai/day-card { story }`.
  - 성공: 응답 `card`의 필드 이름이 `POST /plans` 요청과 같으므로(`title`, `summary`, `reason`, `estimatedMinutes`, `places[].name/activity/whyChosen`, `question`) 그대로 폼 초기값으로 쓴다. `null`인 값은 빈칸.
  - 응답의 `flags[].path`(예: `places[0].name`)를 같은 경로의 입력칸에 연결한다. `NOT_IN_SOURCE` → "원문에 없던 내용이에요, 확인해 주세요", `MISSING` → "직접 채워주세요".
  - 응답의 `draftId`는 공개 요청의 `authoring.aiDraftId`로 보낸다.
  - 실패(`AI_UNAVAILABLE`, `AI_INVALID_OUTPUT`, 타임아웃, 5xx): **입력한 사연을 그대로 유지**하고 "AI 카드를 만들지 못했어요. 직접 정리해 주세요" 안내 후 빈 폼(사연은 참고용으로 상단에 표시)으로 이동. 이때 `mode`는 `MANUAL`.
- "직접 작성하기"는 AI 호출 없이 폼으로 이동. `summary` 기본값으로 사연 앞부분을 채운다.
- 폼 검증: 제목 1–40자, 하고 싶었던 일 요약(`summary`) 1–120자, 취소한 이유 1–300자, 예상 소요 시간 10–240분, 장소 1–3곳(이름 1–30자, 하고 싶었던 일 1–60자, 고른 이유 0–120자), 방문 질문 1–80자(필수, 1개).
- 사연(1–1,000자)은 두 방식 모두 필수다. 원문에 없던 장소·활동 건수를 서버가 두 방식 모두에 대해 계산하기 때문이다.
- **작성 중 상태 보관**: 사연, `draftId`, AI 카드, `flags`, 폼 값, `startedAt`, `idempotencyKey`를 하나의 상태로 묶어 `sessionStorage`에 저장한다(장소 추가 화면 이동·새로고침에도 유지). 공개에 성공하면 삭제.
- **공개**: `POST /plans`, 헤더 `Idempotency-Key`.
  - 키는 "이 하루 공개하기"를 누를 때 요청 본문과 함께 만들어 보관하고, 같은 본문을 재전송할 때만 재사용한다. 폼 내용이 바뀌면 새 UUID를 만든다.
  - 409 `IDEMPOTENCY_CONFLICT`를 받으면 새 키로 한 번 다시 요청한다.
- **작성 시간 측정**: `startedAt`은 사연 입력 화면에 처음 들어올 때 한 번만 기록하고(하위 경로 이동 시 재설정하지 않음), 공개 요청에 `authoring`으로 함께 전송한다.
  ```json
  "authoring": { "mode": "AI_EDITED" | "MANUAL", "durationMs": 183000, "aiDraftId": "..." | null, "story": "사연 원문" }
  ```

### 6.4 내 하루 / 이어받기 포기
- `GET /me/claims?status=ACTIVE`로 현재 이어받은 하루 표시. 없으면 둘러보기 유도.
- 기한 문구: `dueAt` 기준 "{M}월 {D}일까지 다녀온 이야기를 보내주세요." 기한이 지났으면 "기한이 지났어요. 다른 사람이 이어받기 전까지는 보낼 수 있어요."
- "다녀온 이야기 쓰기" 버튼은 응답의 `record.photoStatus`에 따라 이동한다: 기록 없음/`NONE` → 작성 화면, `UPLOAD_PENDING`/`PROCESSING` → 보내는 중, `FAILED` → 사진 처리 실패.
- 포기: 확인 화면 → `POST /claims/{claimId}/abandon` → 둘러보기로 이동. 409 `CLAIM_NOT_ACTIVE`는 "이미 종료된 이어받기예요" 안내 후 새로고침.

### 6.5 방문 기록 작성 / 사진 / 완료
상태 흐름 (서버의 `record.photoStatus` 기준):

```
글 작성 ─┬─ 사진 없음 ─ PUT /record { letter } ─▶ POST /complete (TEXT_ONLY) ─▶ 전송 완료
         └─ 사진 선택 ─ PUT /record { letter, photo } ─▶ upload.url로 S3 PUT
                             └▶ 보내는 중 (폴링 GET /record, 2초 간격)
                                 ├ delivered ─────────▶ 전송 완료
                                 ├ FAILED ────────────▶ 실패 화면: 사진 바꾸기 / 글로 완료
                                 └ (언제든) "글로 완료하기" ─▶ PUT /record { letter } ─▶ POST /complete (TEXT_ONLY)
```

- **"글로 완료"는 항상 `PUT /record`로 현재 글을 먼저 저장한 뒤 `complete`를 호출한다.** 400 `RECORD_REQUIRED`를 받으면 `PUT /record` 후 1회 재시도.
- 작성 화면 상단에 계획 제목, "{계획자 닉네임}에게" 라벨, 하루 카드의 **방문 질문**을 보여준다.
- 작성 화면 진입 시 `GET /claims/{claimId}/record`로 서버에 저장된 글을 먼저 채우고, 없을 때만 `localStorage` 임시 글을 쓴다. 임시 글은 전송 완료·종료 안내 시 삭제.
- 글 1–1,000자. 사진은 JPEG/PNG, 10MB 이하, 1장. 클라이언트에서 크기·형식 1차 검증.
- `PUT /claims/{claimId}/record` 응답의 `upload`로 `fetch(upload.url, { method: 'PUT', body: file, headers: upload.headers })`. 업로드 실패 시 실패 화면과 동일하게 처리.
- 사진 교체: `POST /claims/{claimId}/record/photo` → 새 업로드 URL로 다시 업로드 → 보내는 중.
- 폴링 응답(`GET /record`) 판정 순서 — 위에서부터 먼저 맞는 것:
  1. `delivered === true` → 전송 완료 (`photoStatus === 'READY'`면 사진+글 문구, 아니면 글 문구)
  2. `claimStatus`가 `ABANDONED`/`EXPIRED`, 또는 응답이 403/404/409 → 화면 8
  3. `photoStatus === 'FAILED'` → 사진 처리 실패 화면
  4. 그 외 → 계속 폴링. 60초가 지나도 처리 중이면 "글로 완료하기"를 강조 표시. `UPLOAD_PENDING`이 30초 넘게 바뀌지 않으면 업로드 실패로 보고 실패 화면(사진 바꾸기 / 글로 완료)을 보여준다 (새로고침으로 돌아온 경우 포함).
- **종료된 이어받기**: `PUT /record`, `POST /record/photo`, `POST /complete`가 409 `CLAIM_NOT_ACTIVE`를 반환하거나 폴링 판정 2에 걸리면 화면 8("이 이어받기는 종료되었어요")을 띄운다. 작성한 글은 복사할 수 있게 보여준다. (`photoStatus = DISCARDED`는 글로 완료 성공 때도 나오므로 종료 판단에 쓰지 않는다.)
- `POST /claims/{claimId}/complete`는 재시도해도 안전하다(서버 멱등). 네트워크 오류나 503 `TRY_AGAIN`이면 같은 요청을 자동 1회 재시도.
- **전송 완료 화면 문구**("사진과 방문 기록이…" / "방문 기록이…")는 누른 버튼이 아니라 서버 결과로 정한다: `complete` 응답의 `hasPhoto`, 폴링 경로에서는 `photoStatus === 'READY'`. 완료 화면에서 새로고침하면 `GET /record`로 다시 판단한다.

### 6.6 수신함 / 돌아온 하루 / 답장
- `GET /inbox` → 최신순. `read === false`면 "새 편지" 배지 + 주황 점, 아니면 "읽은 기록". 사진 유무로 "사진과 방문 기록을 보냈어요" / "방문 기록을 보냈어요".
- 헤더 편지 아이콘 점은 `GET /me`의 `unreadInboxCount > 0`일 때 표시.
- 상세 진입 시 `POST /inbox/{itemId}/read` 호출 후 목록/배지 캐시 무효화.
- 상세: 사진(있으면, `record.photoUrl` presigned GET URL — 만료되면 상세를 다시 조회), 사진 아래 `record.photoCaption`, "내가 남긴 계획"(베이지: `plan.summary`) / "{실행자}의 방문 기록"(연녹색: `record.letter`) 2단 카드.
- 한 줄 답장 1–100자 → `POST /inbox/{itemId}/reply`. 성공 또는 409 `REPLY_ALREADY_SENT`면 "보낸 답장" 읽기 전용 상태로 전환.

## 7. API 요약

모든 요청에 `Authorization: Bearer {Cognito ID token}` (닉네임 클레임 때문에 액세스 토큰이 아닌 ID 토큰). 에러 응답은 `{ "error": { "code": "...", "message": "...", "details"?: { "places[0].name": "..." } } }` — `details`는 `VALIDATION_ERROR`일 때 요청 본문 경로별 메시지.

| 메서드 | 경로 | 용도 |
|---|---|---|
| GET | `/me` | 내 프로필(닉네임), 미확인 수신함 수 |
| GET | `/plans?status=OPEN` | 둘러보기 목록 |
| GET | `/plans/{planId}` | 하루 상세 (`isOwner`, `myClaim` 포함) |
| POST | `/ai/day-card` | 사연 → AI 하루 카드 초안 |
| POST | `/plans` | 하루 공개 |
| POST | `/plans/{planId}/claims` | 이어받기 |
| GET | `/me/claims?status=ACTIVE` | 내 하루 |
| POST | `/claims/{claimId}/abandon` | 이어받기 포기 |
| PUT | `/claims/{claimId}/record` | 방문 기록 저장 (+ 사진 업로드 URL 발급) |
| POST | `/claims/{claimId}/record/photo` | 사진 교체용 업로드 URL 재발급 |
| GET | `/claims/{claimId}/record` | 기록·`claimStatus`·사진 처리 상태 조회 (폴링) |
| POST | `/claims/{claimId}/complete` | 글로 완료 (멱등) |
| GET | `/inbox` | 수신함 목록 |
| GET | `/inbox/{itemId}` | 돌아온 하루 상세 |
| POST | `/inbox/{itemId}/read` | 읽음 처리 |
| POST | `/inbox/{itemId}/reply` | 한 줄 답장 (1회) |

프론트에서 분기해야 하는 에러 코드

| code | 처리 |
|---|---|
| `UNAUTHORIZED` (401) | `/login?redirect=현재경로` |
| `NOT_FOUND` (404) | "찾을 수 없어요" 안내 후 둘러보기/수신함 |
| `FORBIDDEN` (403) | 안내 후 둘러보기 |
| `VALIDATION_ERROR` | `details`의 필드별 메시지를 입력칸에 표시 |
| `ALREADY_CLAIMED` | 2-1 화면 |
| `CANNOT_CLAIM_OWN_PLAN` | 토스트 |
| `CLAIM_NOT_ACTIVE` | 포기 화면: 새로고침 / 기록·사진·완료: 화면 8 |
| `RECORD_REQUIRED` | `PUT /record` 후 `complete` 1회 재시도 |
| `IDEMPOTENCY_CONFLICT` | 새 키로 1회 재요청 |
| `AI_UNAVAILABLE`, `AI_INVALID_OUTPUT` | 사연 유지, 직접 작성 폼으로 |
| `REPLY_ALREADY_SENT` | "보낸 답장" 상태로 전환 |
| `TRY_AGAIN` (503) | 같은 요청 1회 재시도 후 실패 시 토스트 |

## 8. 디자인 가이드 (시안 기준)

색상은 시안에서 추정한 값이므로 디자인 확정 시 교체한다.

| 토큰 | 값(추정) | 쓰임 |
|---|---|---|
| `--color-primary` | `#234D3A` | 주 버튼, 제목, 로고, 선택된 탭 |
| `--color-bg` | `#FAF8F3` | 화면 배경 |
| `--color-surface` | `#FFFFFF` | 카드, 입력칸 |
| `--color-plan` | `#F4ECDD` | "내가 남긴 계획", 계획자 사연 박스, 경고 박스 |
| `--color-record` | `#E4EDE5` | "방문 기록" 카드, 상태 배지 배경 |
| `--color-accent` | `#F0B86E` | 미확인 점 |
| `--color-danger-bg` | `#F8E8E4` | 사진 처리 실패 아이콘 배경 |

- 레이아웃: 상단 헤더(뒤로가기 + 가운데 제목) / 본문 / 하단 고정 주 버튼(라운드 12–16px, 꽉 찬 폭) / 하단 탭(목록형 화면만).
- 큰 제목(2줄, 굵게) + 보조 설명(회색) 패턴을 모든 단계 화면에 공통 적용.
- 산책 동선은 번호 원 + 세로 연결선 타임라인. 마지막 장소 번호 원은 채운 primary.
- 상태 배지: 이어받기 가능 / 이어받음 / 완료 / 공개 / 새 편지 / 읽은 기록.
- 완료/확인 화면: 원형 아이콘(체크 · 느낌표) + 제목 + 설명 + 요약 카드 + 하단 버튼.

## 9. 폴더 구조 (제안)

```
src/
  api/            # fetch 클라이언트, 타입(백엔드 계약과 동일), TanStack Query 훅
  auth/           # Amplify 설정, 로그인 가드
  components/     # Header, BottomTab, StatusBadge, RouteTimeline, PrimaryButton, ...
  pages/
    browse/       # 둘러보기
    plan/         # 상세, 작성(사연·AI 카드·폼·장소 추가·공개 완료)
    me/           # 내 하루, 포기
    record/       # 작성, 보내는 중, 실패, 교체, 완료
    inbox/        # 목록, 돌아온 하루
  mocks/          # MSW 핸들러 + 시안 데이터(지연/수현)
  styles/         # 디자인 토큰
```

## 10. 완료 기준 (Definition of Done)

- [ ] 데모 계정 3개(지연, 수현, 민준)로 90초 시연 시나리오가 끊김 없이 동작한다: 지연 사연 입력 → AI 카드 확인 → 공개 → 수현·민준 동시 이어받기 → 수현 방문 기록(사진) 전송 → 지연 수신함 확인 → 답장
- [ ] 수현·민준 두 브라우저에서 같은 하루를 동시에 이어받으면 한 명만 확정되고, 다른 쪽은 "이미 다른 사람이 이어받았어요" 화면을 본다
- [ ] 완료 버튼을 여러 번 눌러도 수신함 항목은 한 건이다
- [ ] 사진 처리 실패 시 작성한 글이 남아 있고, 사진 교체 또는 글로 완료가 가능하다
- [ ] 사진 없이 "글로 완료"해도 바로 전송된다 (`PUT /record` → `complete`)
- [ ] AI 생성 실패 시 사연이 유지되고 직접 작성으로 넘어갈 수 있다
- [ ] 공개 요청에 `authoring.mode`, `durationMs`, `story`가 포함된다
- [ ] 하루 남기기 도중 새로고침해도 입력값과 측정 시작 시각이 유지된다
- [ ] 360–480px 폭 모바일 브라우저에서 시안과 같은 배치로 보인다
- [ ] CloudFront 배포 URL에서 동작하며, 토큰 없이 API를 호출하는 경로가 없다

## 11. 작업 순서 (제안)

1. 프로젝트 세팅, 디자인 토큰, 공통 레이아웃(헤더·탭·버튼), MSW 목업
2. 둘러보기 → 하루 상세 → 이어받기 (경합 화면 포함)
3. 하루 남기기 폼·장소 추가·공개 완료 (직접 작성 경로 먼저)
4. 내 하루 / 포기 → 방문 기록 작성(글만) → 수신함 → 돌아온 하루·답장
5. 사진 업로드·폴링·실패·교체 화면
6. AI 하루 카드 화면, 작성 시간 측정
7. Cognito 연동, 실제 API 전환, CloudFront 배포, 시연 리허설
