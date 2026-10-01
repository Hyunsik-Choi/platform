# 현황 분석서 — AP Chemistry FRQ 자동 채점 프로그램

> PROJECT_BRIEF_학생약점분석시스템.md 0장 1번 산출물 · 단계 0
> 작성 2026-10-01 · **승인 2026-10-01**
> 근거: 저장소 코드, `supabase/migrations/0001~0021`, 운영 DB 읽기 전용 백업(2026-10-01 11:34 KST)

---

## 0. 운영 현황

| 항목 | 내용 |
|---|---|
| 운영 URL | https://chs-ap-chemistry.vercel.app (Vercel, 서울 엣지 icn1). GitHub `main` push 시 자동 배포로 추정 (Vercel CLI 미사용) |
| DB | Supabase 프로젝트 `tbfm…` **1개를 운영과 로컬 개발이 공용**. 개발 환경 없음 → 그동안의 로컬 개발·QA·마이그레이션이 모두 운영 DB에 직접 적용됨 |
| 사용 규모 (현재) | 실제 학생 19명 등록, 11명 응시. 주당 FRQ 약 2문항(파트 7~8개). 첫 제출 2026-08-31, 최근 2026-09-27 |
| 목표 규모 | 학생 60~80명, 주당 FRQ 최대 14문항 (현재의 약 50배) |

## 1. 사용 기술

| 항목 | 내용 |
|---|---|
| 언어·프레임워크 | TypeScript, Next.js 15 (App Router, Server Actions), React 19, Tailwind 3 |
| DB | Supabase Postgres. 권한은 RLS (`is_staff()`, `is_owner()`) |
| 인증 | Supabase Auth + `@supabase/ssr` 쿠키 세션 |
| 파일 | Supabase Storage 비공개 버킷 `exam-assets`(문제·루브릭 이미지), `answers`(학생 답안 이미지) |
| AI | Anthropic SDK. 채점·루브릭 파싱 `claude-opus-5`, 단원 분류 `claude-sonnet-5`. 즉시 채점 + Message Batches API |
| 배포 | Vercel. `vercel.json`: 리전 hnd1, Cron 2개(배치 발송 매일 23:00 KST, 월 마감) |
| 마이그레이션 | SQL 파일을 Supabase SQL Editor에 수동 실행. **적용 기록 장치 없음** (단, 2026-10-01 기준 운영 DB 컬럼은 0001~0021과 일치 확인) |
| 백업 | 별도 장치 없음 (2026-10-01 첫 논리 백업 수행) |
| 테스트 | 자동 테스트 없음. typecheck·build·수동 브라우저 QA |
| DB 타입 | `lib/db/types.ts` 수작성 |

## 2. 폴더 구조

```
app/
  login, staff-login        학생 / 교사·TA 로그인 분리
  student/                  대시보드, 클래스, 응시(TakeExam.tsx), 결과, 오답 노트
  teacher/                  시험지 편집·배정·제출물·검수, 학생·TA 관리, 대시보드, 비용, 알림
  view-student/[id]         교사가 학생 화면 열람
  share/[token]             결과 공유 링크 (로그인 필요)
  api/                      grade, rubric/parse, submissions/[id]/{enqueue,grade,status}, cron 2개
  spike/                    Phase 1 스파이크 (보존)
lib/
  grader.ts                 채점 프롬프트·호출·결과 검증
  rubric.ts                 채점 기준 → ParsedRubric JSON
  schema.ts                 GradingResult 계약 (zod + JSON schema)
  grading/                  즉시·배치 파이프라인 (shared, immediate, enqueue, dispatch, ingest, deadline, notify, stale)
  classify*.ts, ap-units.ts AP Unit 1~9 + 자유 텍스트 topic 자동 분류
  analytics.ts              학생별·단원별 정답률
  sessions.ts, auth.ts      역할 가드, 학생 단일 세션
  cost.ts                   토큰 비용 계산, 월 예산 가드
supabase/migrations/        0001~0021
docs/PRD.md                 FRQ 채점기 PRD (Phase 1~8)
```

## 3. 데이터 구조

### 3.1 관계

```
auth.users ─1:1─ profiles(role: owner|admin_ta|student)
                   ├─ student_details (학교·성별·전화·student_no·memo·login_password)
                   └─ class_members ── categories(kind='class')

categories(kind='exam') ── exams ─┬─ questions ── sub_questions (문항당 1행 컨테이너)
                                  │      └ scoring_guidelines (ref_id = question.id, FK 없음)
                                  ├─ exam_assignments (via_class_id, 응시 기간 override)
                                  └─ submissions (UNIQUE exam × student)
                                         ├─ answers (UNIQUE submission × sub_question)
                                         ├─ gradings (UNIQUE submission × question) ── teacher_reviews (1:1)
                                         └─ grading_jobs, notifications, result_shares
기타: cost_settings, usage_rollups, cost_budget_approvals, ta_credentials, user_sessions
```

### 3.2 항목별 저장 형태와 브리프 대비 차이

| 항목 | 현재 | 브리프 대비 |
|---|---|---|
| 문항 | `questions`가 시험지에 종속 (`exam_id` FK CASCADE). 복제 시 새 ID | 문항은행·문항 버전 없음 |
| 파트 | `sub_questions`는 문항당 1행(label ""). 파트 (a)(b)는 JSON 안에만 존재 | 파트 ID·파트 태그·파트 의존성 없음 |
| 루브릭 | `scoring_guidelines.parsed_rubric` JSONB `{sub_questions[{label,max_points,scoring_points[{description,points,acceptable_alternates,notes}]}],notes}` + 원본 이미지·텍스트 | 채점 포인트 ID 없음. 수정 시 덮어쓰기, `version` 숫자만 증가 |
| 분류 | `sub_questions.ap_unit`(1~9) + `ap_topic`(자유 텍스트), AI 분류, 교사 잠금 가능 | 공통 개념 ID, 스킬, 오류 코드 없음 |
| 학생 | `profiles.id`(Auth UUID), 로그인 키 이메일, 선택 `student_no` | 6절 |
| 답안 | `answers` 문항당 1행 (`answer_text` + `image_paths[]`), 900ms 디바운스 자동저장 덮어쓰기 | 파트별 분리 없음, 편집 이력 없음 |
| 채점 결과 | `gradings.result` JSONB: 파트별 `{label, awarded/max, rubric_points[{earned, earned_points, evidence, reasoning}], mistakes[{location, issue, correction}], ambiguous, confidence, comment}` + model·usage·guideline_version | 포인트별 판정 있음. `mistakes`는 자유 서술(오류 코드 없음). 판정 이력 없음 |
| 교사 검수 | `teacher_reviews.point_overrides` `{"<파트 index>": {points, comment}}`, AI 원본 보존 | 파트 단위(포인트 단위 아님), 이력 1개 |

## 4. 학생 답안 원본 보존 여부

**보존됨:** 제출 후 답안 텍스트·이미지(RLS상 `in_progress`에서만 쓰기 가능), Storage 원본 이미지, 교사 검수 시 AI 원본 결과.

**잃는 경로 (운영 데이터에 현재 적용 중인 위험):**
1. 재채점: `resetSubmission`이 `gradings` DELETE → 이전 AI 판정과 교사 검수(CASCADE) 소실
2. 루브릭 수정: `parsed_rubric` 덮어쓰기 → 과거 채점 기준 내용 소실
3. 시험지·폴더·문항 삭제: CASCADE로 해당 제출·답안·채점 전부 DB 삭제. Storage 이미지·`scoring_guidelines`(FK 없음)는 고아로 남음 (운영 DB에 고아 루브릭 40행, 미참조 이미지 exam-assets 46개·answers 16개 확인)
4. 제출본 삭제(교사): 이미지까지 삭제 (의도된 기능)
5. 응시 1회 제한: `submissions` UNIQUE(exam_id, student_id) → 재응시 기록 불가
6. 답안 편집 이력 없음

## 5. FRQ 입력과 채점 흐름

```
[교사] 시험지 → 문항(지문 텍스트/이미지) → 채점 기준 이미지 업로드 → ParsedRubric(Opus) 검토·수정
       → 배점 자동 합산, AP Unit 자동 분류(Sonnet) → 배정(개별/클래스) + 응시 기간 + 제한 시간 → 공개
[학생] 문항별 텍스트 + 이미지(드래그·붙여넣기·파일, 브라우저→Storage 직접 업로드) → 자동저장
       → 제출 (마감·시간 초과 시 자동 제출, 손대지 않은 문항은 채점 제외)
[채점] immediate: 제출 즉시 문항별 gradeAnswer → gradings upsert
       batch: grading_jobs 큐 → Cron/수동 Batch API → 화면 열람 시 결과 수거 (운영 시험 전부 batch)
       hybrid: 구현됨, 플래그로 숨김
       입력 = 루브릭(텍스트+이미지) + 지문 + 답안(텍스트+이미지), 문항 단위 1회 호출
[결과] ambiguous면 학생 공개 보류 + 교사 알림 → 교사 검수 → 결과 화면, 오답 노트, 공유 링크
[집계] 학생별·AP Unit별 정답률. 월 예산(KRW) 초과 시 채점 중지
```

## 6. 로그인·권한·학생 ID

- 역할: owner / admin_ta / student. **학부모 역할 없음.**
- 교사·TA: `/staff-login` 이메일+비밀번호. TA 비밀번호 평문 사본 `ta_credentials` (Owner 전용).
- 학생 (임시 방식): 이름+이메일만, 서버가 매직링크 생성 후 즉시 소비. 미등록 이메일 거부. 단일 세션 강제, 응시 중 타 기기 로그인 차단.
  - ⚠️ 이메일만 알면 로그인 가능, 입력한 이름으로 `profiles.name` 덮어씀
  - ⚠️ `student_details.login_password` 평문 저장 (인증 미사용)
- 학생 ID: `profiles.id` UUID가 모든 FK의 기준. 의미 없는 불변 식별자 → **브리프 원칙 3에 그대로 재사용 가능.** 학번 `student_no`는 선택·중복 검사 없음.
- 클래스: `categories(kind='class')` 트리, 다중 소속.
- 개인정보: 실명 저장, 보호자 동의 필드 없음.

## 7. 통합 관점 요약

| 구분 | 내용 |
|---|---|
| 재사용 | 학생 UUID·역할·RLS, 클래스·배정·응시 기간·제한 시간, Storage 원본 보존, 포인트별 GradingResult, 비용 기록·예산 가드, `exams.kind`('frq'\|'mcq') |
| 구조 변경 | 문항은행 + 시험 세트(버전 고정), 파트·채점 포인트 ID, 파트별 답안, 판정 이력 누적, 루브릭 버전별 보존, soft delete, 재응시 |
| 신규 | 기준 데이터(개념·스킬·오류 코드), MCQ 선택지, 응답 원자료(선택 이력·다시 보기·소요 시간), 오류 판정, 학부모 역할, 보호자 동의 |
| 운영 기반 | 개발·운영 분리, 마이그레이션 기록, 자동 백업, 테스트 |

## 부록 A. 운영 DB 집계 (2026-10-01 백업 기준)

| 테이블 | 행 | 테이블 | 행 |
|---|---|---|---|
| profiles | 25 (owner 1, TA 2, 학생 22) | submissions | 43 (reviewed 17, graded 26) |
| student_details | 22 | answers | 88 (이미지 포함 63, 텍스트 포함 25) |
| categories | 17 | gradings | 88 (batch 76, immediate 12) |
| class_members | 21 | teacher_reviews | 22 |
| exams | 6 (published 5, draft 1) | grading_jobs | 76 |
| questions / sub_questions | 13 / 13 | notifications | 3 |
| scoring_guidelines | 53 (고아 40) | result_shares | 3 |
| exam_assignments | 102 | user_sessions | 106 |
| cost_settings | 7 | ta_credentials | 2 |
| usage_rollups, cost_budget_approvals | 0 | auth.users | 25 |

Storage: `answers` 84개(6.3MB, 답안이 참조하는 것 68개), `exam-assets` 73개(4.0MB, 현존 문항·루브릭이 참조하는 것 27개).

테스트 계정: `student1~3@test.com`, `ta1·ta2@test.com`. 2026-09-14 QA 데이터(`[QA테스트]` 시험, QA 학생·반)는 이미 운영 DB에 없음.
