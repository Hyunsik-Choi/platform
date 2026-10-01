# 이식 지도 — 기존 FRQ 앱 → platform

> 기존 앱 경로: `C:\Users\boria\Desktop\Vibe Coding\Project 1 (frq grading)` (**읽기 전용. 절대 수정하지 않음**)
> 기존 앱 스택: Next.js 15 / React 19 / Supabase / Anthropic SDK. 새 앱은 **Next.js 16** — API 차이는 `node_modules/next/dist/docs/` 확인
>
> 처리 구분:
> - **이식**: 거의 그대로 가져와 import 경로만 맞춤
> - **수정 이식**: 핵심 로직은 쓰되 새 데이터 구조(org_id, 문항은행, 이력)에 맞게 고침
> - **참고**: 동작·UX만 참고하고 새로 작성
> - **폐기**: 가져오지 않음

## 채점 엔진 (P3)

| 기존 파일 | 새 위치 | 구분 | 메모 |
|---|---|---|---|
| `lib/grader.ts` | `lib/scoring/frq/grader.ts` | 수정 이식 | 프롬프트의 "AP Chemistry" 고정을 `grading_profiles` 템플릿으로. 결과는 포인트 행으로 정규화 |
| `lib/schema.ts` (GradingResult) | `lib/scoring/frq/schema.ts` | 이식 | E2에서 포인트별 오류 코드 후보 필드 추가 예정 |
| `lib/rubric.ts` (루브릭 파싱) | `lib/scoring/frq/rubric-parse.ts` | 수정 이식 | 결과를 `frq_parts`/`frq_scoring_points` 행으로 저장 |
| `lib/rubric-points.ts` | `lib/itembank/frq-points.ts` | 참고 | 배점 합산 |
| `lib/grading/shared.ts` | `lib/scoring/frq/load.ts` | 수정 이식 | 문항은행·파트별 답안 기준으로 요청 구성 |
| `lib/grading/immediate.ts`, `enqueue.ts`, `dispatch.ts`, `ingest.ts`, `batch-eta.ts`, `stale.ts` | `lib/scoring/frq/batch/*` | 수정 이식 | DELETE 재채점 → `is_current` 이력. org_id 추가 |
| `lib/grading/notify.ts`, `lib/notifications.ts` | `lib/notifications/*` | 수정 이식 | |
| `lib/grading/release.ts` | `lib/delivery/release.ts` | 참고 | 결과 공개 조건은 설계안 5.4로 새로 |
| `lib/review.ts` | `lib/scoring/frq/review.ts` | 수정 이식 | 파트 index → 포인트/파트 ID, 이력 |
| `lib/classify*.ts`, `lib/ap-units.ts` | `lib/reference/*` | 참고 | 단원은 DB 기준 데이터로. AI 분류는 E1 태깅 파이프라인에 흡수 |
| `app/api/rubric/parse`, `app/api/grade`, `app/api/submissions/*`, `app/api/cron/*` | `app/api/...` | 수정 이식 | |

## 응시·배정 (P1~P2)

| 기존 파일 | 새 위치 | 구분 | 메모 |
|---|---|---|---|
| `lib/exam-window.ts`, `lib/grading/deadline.ts` | `lib/assessment/deadline.ts` | 수정 이식 | `recompute_deadline`, 유예, 연장 규칙(설계안 6.5)으로 확장 |
| `lib/sessions.ts` | `lib/core/sessions.ts` | 이식 | 학생 단일 세션 |
| `lib/categories.ts`, `lib/category-tree.ts`, `components/CategoryTree.tsx` | `lib/assessment/folders.ts`, `lib/org/classes.ts` | 수정 이식 | 폴더·반 분리, soft delete |
| `app/teacher/exams/[id]/assign/*` | `app/teach/assessments/[id]/assign/*` | 수정 이식 | QA에서 발견된 "배정 + 초대메일" 크래시 버그 패턴 피할 것 (모든 액션 try/catch 래퍼) |
| `app/student/exams/[id]/TakeExam.tsx` | `app/learn/attempts/[id]/*` | 참고 | MCQ·FRQ 통합 플레이어로 새로 작성 (이벤트 큐, 그림 사전 다운로드, Wake Lock) |
| `components/ImageUpload.tsx` | `components/ImageUpload.tsx` | 수정 이식 | 제출 후 삭제 불가 (서버 경유) |
| `components/DateTimePicker.tsx` | 같은 이름 | 이식 | |

## 계정·관리 (P1)

| 기존 파일 | 새 위치 | 구분 | 메모 |
|---|---|---|---|
| `lib/auth.ts` | `lib/core/auth.ts` | 수정 이식 | JWT 클레임(pid, orgs) 기반 |
| `lib/supabase/*`, `middleware.ts` | `lib/supabase/*`, `middleware.ts`(또는 Next 16의 proxy) | 수정 이식 | Next 16 변경 확인 |
| `app/login/actions.ts` | — | **폐기** | 이름+이메일 로그인, 이름 덮어쓰기 → 이메일+비밀번호, 초대 방식 |
| `app/teacher/students/*`, `components/StudentTable.tsx`, `StudentForm.tsx`, `ImportPanel.tsx`, `app/teacher/students/import/template` | `app/teach/students/*` | 수정 이식 | 엑셀 일괄 등록 → 일괄 초대 |
| `app/teacher/ta/*`, `components/TaManager.tsx`, `lib/ta.ts` | `app/org/staff/*` | 참고 | 평문 비밀번호 사본(`ta_credentials`) **폐기** |
| `lib/cost.ts`, `app/teacher/cost/*` | `lib/billing/*`, `app/org/cost/*` | 수정 이식 | Batch 50% 할인 반영, `ai_usage_log`(채점 외 용도 포함), 기관별 예산 |

## UI 공통

| 기존 파일 | 새 위치 | 구분 | 메모 |
|---|---|---|---|
| `components/AsyncActionButton.tsx`, `SubmitButton.tsx`, `SortSelect.tsx`, `SignOutButton.tsx` | `components/*` | 이식 | 브라우저 `confirm()` 대신 자체 모달 (기존 앱 백로그) |
| `components/GradingResultView.tsx`, `ResultDocument.tsx` | `components/results/*` | 수정 이식 | 정답·해설은 결과 공개 후에만 |
| `components/WrongQuestionsClient.tsx` | `app/learn/review/*` | 참고 | |
| `app/globals.css`, `tailwind.config.ts` | `app/globals.css` | 참고 | 새 앱은 Tailwind v4 (설정 방식 다름) |

## 데이터 이전 (P3)

| 기존 | 새 | 메모 |
|---|---|---|
| `questions`, `scoring_guidelines.parsed_rubric`, Storage `exam-assets` | `scripts/import-legacy-frq` | 전환 계획 2·4장 |
| `submissions`, `answers`, `gradings`, `teacher_reviews` | 같은 스크립트의 선택 단계 | 설계안 10.2 (루브릭 버전 일치 시만 포인트 단위) |

## 가져오지 않는 것
- `app/spike/` (Phase 1 스파이크)
- `student_details.login_password`, `ta_credentials` (평문 비밀번호)
- 수작성 DB 타입 `lib/db/types.ts` → `supabase gen types`로 대체
- 기존 RLS 정책 (A1~A3 문제 포함) → 새로 설계
