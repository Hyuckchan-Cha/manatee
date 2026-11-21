# manatee 개발 안내

고등학교 과학 수업용 형성평가 · 통계 · 정기고사 출제 도구 **manatee** 의 개발 안내예요.
Next.js 16(App Router, Vercel) + Supabase(Postgres · Auth · Realtime · Storage) + OpenAI API 로 만들었어요.

- 운영 사이트: https://class-tool-web.vercel.app
- 서비스 소개(영문): [README.md](README.md)

## 폴더 구조

```
src/
  app/
    login/             교사 로그인 · 가입 (Supabase Auth, 이메일 + 비밀번호)
    auth/              가입 인증 메일 링크 처리 · 로그아웃
    api/teacher/       교사 기능 호출 창구  POST { fn, args }
    api/student/       학생 기능 호출 창구  POST { fn, args }
    teacher/           교사 화면 골격(teacher-app.tsx) + 스타일(teacher.css)
    teacher/widget/    수업 위젯 (PPT 옆에 띄우는 작은 창)
    student/           학생 태블릿 화면 (다국어)
    terms/ privacy/    이용약관 · 개인정보처리방침
  server/
    core.ts            Ctx(교사 세션) · ApiError · 공통 함수
    mappers.ts         DB 행 → 화면에서 쓰는 객체
    api/               교사 기능 (문제은행 · 수업 · 개념 · 오개념 · 학생 명단 · 설정 · 통계 · 출제 가이드 · 기본 문항 세트)
    student.ts         학생 로그인 · 수업 상태 · 제출 · 틀린 문제 다시 보기
    student-auth.ts    학생 비밀번호 · 로그인 쿠키
    openai.ts          OpenAI 호출 (문제 만들기 · 그림 · 문항 분석 · 출제 포인트)
    translate.ts       학생 화면 문제 번역
    realtime.ts        수업 화면에 "바뀌었어요" 신호 보내기
    seed/              기본 문항 세트 (physics-items.json)
  lib/
    database.types.ts  DB 타입
    supabase/          client · server · admin · proxy 클라이언트
    live.ts            실시간 수업 공통 상수
    langs.ts           학생 화면 언어 목록
    site.ts            서비스 · 팀 정보 (푸터 · 약관)
  proxy.ts             Next.js 16 Proxy (로그인 확인)
public/
  teacher-app/app.js       교사 화면 — 문제은행 · 수업하기 · 개념 지도 · 학생 목록 · 설정
  teacher-app/analysis.js  교사 화면 — 형성평가 분석 · 정기고사 출제 가이드
  i18n/student/            학생 화면 번역 파일
  sample/                  기본 문항 그림
  video/                   튜토리얼 영상
supabase/migrations/   DB 스키마
data/concept-links.json    기본 개념 지도의 개념 ↔ 개념 연결
scripts/               기본 문항 세트 만들기 · 시연 데이터 · 시험 스크립트
design/                발표 자료
```

## 교사 화면

`src/app/teacher/teacher-app.tsx` 가 화면 골격을 그리고 `public/teacher-app/app.js` · `analysis.js` 를 순서대로 불러와요.
두 파일은 전역 범위를 함께 쓰는 일반 스크립트예요.

| 메뉴 | 하는 일 |
|---|---|
| 문제은행 | 폴더 · 문제 관리, AI 문제 만들기(그림 포함), 선지마다 오개념 연결, 문제마다 개념 연결 |
| 수업하기 | 클래스 · 차시 · 문제 구성, 학생 목록 관리(반 · 학습코드 · 학번 · 처음 비밀번호 · 안내문 인쇄), 수업 시작 |
| 통계 | 과목별 개념 지도(정답률 색 · 오개념 · 연결 문제), 형성평가 분석 |
| 정기고사 출제 가이드 | 시험 목적 · 범위 · 문항 수 · 총점을 고르면 문항 구성표 추천 |

## 교사 API (`POST /api/teacher`)

화면에서 `fetch('/api/teacher', { method: 'POST', body: JSON.stringify({ fn: 'bootstrap', args: [] }) })` 처럼 불러요.
응답은 `{ ok: true, data }` 또는 `{ ok: false, error }` 예요 (로그인이 필요하면 401 · `code: 'AUTH'`).
기능 목록은 `src/server/api/index.ts` 에 있어요.

- 모든 테이블에 `owner_id` 가 있고, RLS로 **교사마다 자기 데이터만** 봐요.
- 문제 그림은 Supabase Storage `question-images/<교사 uid>/…` 에 저장해요(공개 읽기).
- OpenAI 키는 환경변수 `OPENAI_API_KEY` 로만 받아요. 키가 없으면 AI 기능 자리에 안내 메시지가 나와요.

## 실시간 수업

| 화면 | 주소 | 로그인 |
|---|---|---|
| 교사 수업 위젯 | `/teacher/widget?lesson=<차시 id>` (교사 화면의 [수업 시작]) | 교사 쿠키 |
| 학생 태블릿 | `/student` | 학습코드(반마다 4자리) + 학번 + 비밀번호 |

- 흐름: 수업 시작 → [문제 출제하기] → 학생 제출 → [정답 공개] → 다음 문제 → [수업 종료].
- 위젯에서 접속 QR 코드 · 문제 크게 보기, 선지별 응답 수와 그 선지를 고른 학생 · 오개념을 볼 수 있어요.
- 갱신: 교사 조작 · 학생 접속 · 제출 때 서버가 Supabase Realtime Broadcast(`live-<세션 uuid>`, 이벤트 `changed`)로 신호를 보내고, 화면이 새로 불러와요(`src/lib/use-live-channel.ts`). 신호를 놓칠 때를 대비해 느린 폴링도 함께 돌아요.
- 학생 API(`POST /api/student`)는 secret key로 동작하므로 `src/server/student.ts` 에서 학생 · 수업 · 단계 · 문제를 직접 검증해요. 정답 여부와 정답은 [정답 공개] 뒤에만 보내요.
- 수업이 끝나면 학생은 [틀린 문제 다시 보기]로 끝난 수업에서 틀린 문제를 볼 수 있어요.

### 학생 로그인 · 사칭 방지 (`src/server/student-auth.ts`)

- 처음 비밀번호는 학생마다 다른 6자리 숫자예요. 교사가 [안내문 인쇄]로 나눠 줘요.
- 처음 비밀번호로 들어오면 그 자리에서 자기 비밀번호를 정해야 해요(너무 쉬운 비밀번호는 거절).
- 로그인 상태는 서명한 httpOnly 쿠키예요. 새로 로그인하면 이전 기기는 로그아웃돼요.
- 같은 학교 · 학번 학생이 다른 반 명단에도 있고 비밀번호가 같으면 함께 로그인해요.
- 비밀번호를 잊으면 교사가 처음 비밀번호를 새로 만들어요.

## 통계 · 정기고사 출제 가이드

계산은 `src/server/api/analysis.ts` 에 있어요. 방법은 [README.md](README.md) 의 *How the Statistics Work* · *How the Exam Recommendation Works* 에 정리했어요.

## 기본 문항 세트

새 계정과 모든 교사 계정에 물리학 기본 문항 · 개념 · 오개념 · 개념 연결이 들어가요.

- 세트 파일: `src/server/seed/physics-items.json` (직접 고치지 말고 아래 명령으로 만들어요) + 그림 `public/sample/*.svg`
- 만들기:
  ```bash
  npm run sample:update     # = sample:export (기준 교사: .env.local 의 SAMPLE_EMAIL) + sample:build
  ```
  `sample:export` 는 기준 교사 계정을 읽어 `data/sample-export.json` 을 만들고, `sample:build` 가 세트 파일 · 그림을 써요.
  개념 연결은 `data/concept-links.json` 에서 가져와요. 내용이 바뀌면 세트 `version` 이 1 올라가요.
- 적용(`src/server/api/sample.ts`): 교사가 접속할 때 계정의 세트 버전이 낮으면 한 번 맞춰요.
  교사가 만든 문항 · 폴더 · 개념은 그대로 두고, 기본 문항을 고쳐 저장하면 그때부터 그 교사의 문항이 돼요.

## 처음 설정

```bash
npm install
cp .env.example .env.local     # 값 채우기
```

| 환경변수 | 설명 |
|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` · `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` | Supabase 프로젝트 주소 · 공개 키 |
| `SUPABASE_SECRET_KEY` | 서버 전용 키 (학생 API · 스크립트). `NEXT_PUBLIC_` 를 붙이지 마세요 |
| `OPENAI_API_KEY` | AI 기능 (문제 만들기 · 그림 · 분석 · 번역) |
| `STUDENT_SESSION_SECRET` (선택) | 학생 로그인 쿠키 서명 키. 없으면 `SUPABASE_SECRET_KEY` 에서 만들어 써요 |
| `OPENAI_IMAGE_MODEL` · `OPENAI_TRANSLATE_MODEL` · `OPENAI_LINK_MODEL` · `OPENAI_BASE_URL` (선택) | 모델 · 주소 바꾸기 |
| `SAMPLE_EMAIL` (선택) | 기본 문항 세트를 내보낼 기준 교사 계정 |

Supabase 프로젝트에 스키마 적용:

```bash
npx supabase login
npx supabase link --project-ref <프로젝트 ref>
npm run db:push
```

개발 서버: `npm run dev`

### 내 컴퓨터에서만 돌려 보기

Docker Desktop 을 켠 뒤:

```bash
npm run local:up      # 로컬 Supabase 켜기 + 마이그레이션 적용 + .env.development.local 생성
npm run local:demo    # 시험용 교사 계정 · 학생 명단 · 클래스 · 오늘 차시
npm run dev           # .env.development.local 이 있으면 로컬 DB 를 써요
npm run local:down    # 끄고 .env.development.local 삭제
```

`npm run local:reset` 은 로컬 DB 를 비우고 처음부터 다시 만들어요.

## 스크립트

| 명령 | 하는 일 |
|---|---|
| `npm run smoke:teacher -- <주소>` | 교사 API 통합 시험 (임시 교사를 만들고 끝나면 삭제) |
| `npm run smoke:live -- <주소>` | 실시간 수업 · 학생 로그인 통합 시험 |
| `npm run smoke:ai-mock` | 가짜 OpenAI 서버로 AI 기능 시험 (방법은 스크립트 맨 위 주석) |
| `npm run i18n:student` | 학생 화면 번역 파일 만들기 (`public/i18n/student/`) |
| `npm run typecheck` · `npm run lint` | 타입 · 린트 검사 |
| `node --env-file=.env.local scripts/demo-ab-classes.mjs --email <교사>` | 시연용 A반 · B반(각 30명)과 형성평가 응답 만들기 |
| `node --env-file=.env.local scripts/demo-copy-classes.mjs --from <교사> --to <교사>` | 한 교사의 명단 · 클래스 · 응답을 다른 교사 계정에 복사 |

시험 스크립트는 `npm run build && npx next start -p 3123` 으로 앱을 띄운 뒤 `http://localhost:3123` 을 주소로 넘겨요.
