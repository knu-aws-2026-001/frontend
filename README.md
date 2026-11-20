# 이어온 하루 — Frontend

취소된 동네 산책을 이어받고, 그날의 이야기를 돌려주는 모바일 웹입니다.
AWS Skill Training 1조 프로젝트 **"취소된 하루를 양도합니다"** 의 프론트엔드 저장소입니다.

계획자가 하지 못한 산책 계획(공개 장소 1–3곳)을 남기면, 같은 동네의 실행자 한 명이 이어받아 다녀옵니다. 실행자의 방문 기록(글 + 선택 사진 1장)이 계획자의 수신함에 도착하고, 계획자는 한 줄 답장을 보냅니다.

## 문서

아래 순서로 읽으면 됩니다.

| 문서 | 내용 |
|---|---|
| [`MVP.md`](./MVP.md) | MVP 범위, 화면 목록과 화면별 동작, 디자인 가이드 |
| [`docs/REQUIREMENTS.md`](./docs/REQUIREMENTS.md) | 요구사항 명세서 (`FE-*` ID, 인수 기준, 확인이 필요한 사항) |
| [`docs/DEVELOPMENT_PLAN.md`](./docs/DEVELOPMENT_PLAN.md) | 개발계획서 (7단계 작업, 백엔드 연동 지점 S0–S5, 테스트 전략) |

## 관련 저장소

- 백엔드: [knu-aws-2026-001/backend](https://github.com/knu-aws-2026-001/backend)
- **API 계약의 기준은 백엔드 [`MVP.md`](https://github.com/knu-aws-2026-001/backend/blob/main/MVP.md) §7입니다.** 이 저장소의 API 타입과 MSW 목업은 그 계약을 따릅니다. 계약 변경은 백엔드 `MVP.md` 수정 PR에서 먼저 합의합니다.

## 기술 스택

| 영역 | 선택 |
|---|---|
| 프레임워크 | React 18 + TypeScript + Vite |
| 라우팅 | React Router |
| 서버 상태 | TanStack Query |
| 인증 | AWS Amplify `aws-amplify/auth` (Cognito User Pool, ID 토큰 사용) |
| 스타일 | CSS Modules 또는 Tailwind (팀 합의 예정) |
| API 목업 | MSW |
| 배포 | S3(비공개) + CloudFront (백엔드 SAM 스택이 생성) |

## 진행 상황

[`docs/DEVELOPMENT_PLAN.md`](./docs/DEVELOPMENT_PLAN.md) 3장의 단계입니다.

- [ ] 1단계. 프로젝트 세팅, 디자인 토큰, 공통 레이아웃, MSW 목업
- [ ] 2단계. 둘러보기 → 하루 상세 → 이어받기
- [ ] 3단계. 하루 남기기 (직접 작성 경로)
- [ ] 4단계. 내 하루 → 방문 기록(글) → 수신함 → 답장
- [ ] 5단계. 사진 업로드·폴링·실패·교체
- [ ] 6단계. AI 하루 카드, 작성 시간 측정
- [ ] 7단계. Cognito 연동, 실제 API 전환, 배포, 시연 리허설

## 환경 변수

`.env.example`을 복사해 `.env.local`을 만들어 씁니다. `.env.local`은 커밋하지 않습니다.

| 변수 | 값의 출처 |
|---|---|
| `VITE_API_BASE_URL` | 백엔드 SAM 출력값 `ApiUrl` |
| `VITE_AWS_REGION` | `ap-northeast-2` |
| `VITE_COGNITO_USER_POOL_ID` | 백엔드 SAM 출력값 `UserPoolId` |
| `VITE_COGNITO_CLIENT_ID` | 백엔드 SAM 출력값 `UserPoolClientId` |
| `VITE_USE_MOCK` | `true`면 백엔드 없이 MSW 목업으로 동작 |

백엔드가 배포되기 전에는 `VITE_USE_MOCK=true`로 모든 화면을 개발합니다.

## 실행 방법

1단계 이후 추가

## 빌드·배포

1단계 이후 추가

## 작업 규칙

자세한 내용은 [`docs/DEVELOPMENT_PLAN.md`](./docs/DEVELOPMENT_PLAN.md) 7장을 따릅니다.

- `main`은 항상 빌드 가능한 상태로 유지하고, 작업은 `feat/<단계>-<요약>` 브랜치(예: `feat/5-photo-polling`)에서 PR로 병합합니다.
- PR 본문에 관련 요구사항 ID(`FE-*`), 확인한 화면 번호, 목업/실제 API 중 어디서 확인했는지 적습니다. 화면 변경은 390px 폭 스크린샷을 첨부합니다.
- 커밋 메시지 접두어: `feat:`, `fix:`, `test:`, `style:`, `chore:`, `docs:`
- 실제 API URL, User Pool ID, 테스트 계정 정보는 저장소에 올리지 않습니다.
