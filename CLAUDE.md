@AGENTS.md

# platform — 학습 진단 플랫폼 (작업명)

## 사용자와 소통
- **항상 한국어로** 답한다. 사용자는 18년 경력의 AP·Honors·Regular Chemistry 강사이자 운영자다.
- 결정이 필요한 사항은 임의로 정하지 않고, 추천과 함께 질문한다 (`docs/open_questions.md`).
- 새 개발 작업이 시작될 때 **보류·리마인드 항목**(아래)을 먼저 확인시킨다.

## 이 프로젝트는 무엇인가
- AP Chemistry FRQ 자동 채점 앱(기존, 운영 중)을 대체하는 **새 플랫폼**. MCQ + FRQ를 **하나의 시험 엔진**에서 처리하고, 학생 오답을 오류 코드로 분석한다.
- **멀티테넌트**(다른 학원에도 서비스), **다과목 확장**(AP Chem 먼저 → Regular·Honors → 다른 화학 과정·다른 분야) 전제로 설계됐다.
- 기준 문서 (반드시 먼저 읽을 것):
  - `docs/PROJECT_BRIEF_학생약점분석시스템.md` — 원 요구사항 (v3.1)
  - `docs/integration_design.md` — **통합 설계안 v4.1 (확정)**: 테이블·RLS·이벤트·마감·가명 처리 규칙
  - `docs/dev_plan.md` — 단계별 작업과 완료 기준 (P-1 → P0 → P1 → P2 → P2.5 파일럿 → P3 → 출시 → E1/E2/F)
  - `docs/open_questions.md` — 미결정 질문과 추천
  - `docs/transition_plan.md` — 기존 앱 → 새 플랫폼 전환
  - `docs/porting_map.md` — 기존 코드 이식 지도
  - `docs/current_state.md` — 기존 앱 현황 분석
  - `docs/supabase_settings_checklist.md` — 마이그레이션에 안 남는 Supabase 설정
  - `docs/pilot_launch_checklist.md` — 파일럿 직전 운영 DB·Vercel 작업

## 환경
- 저장소: https://github.com/Hyunsik-Choi/platform (private). 기본 브랜치 `main`
- Supabase: **`platform-dev`만 존재** (ref `vblnedomxgchnyzglcjo`, Seoul, automatic RLS 켬). `platform-prod`는 파일럿 직전(P2.5.0)에 생성
- Vercel: 아직 연결 안 함
- Next.js **16** (AGENTS.md 경고 참고 — 학습 데이터와 API가 다를 수 있음), Tailwind v4, Supabase CLI는 `npx supabase` (devDependency)
- 기존 앱: `C:\Users\boria\Desktop\Vibe Coding\Project 1 (frq grading)` — https://chs-ap-chemistry.vercel.app 운영 중. **읽기 전용으로만 참조. 파일 수정·DB 쓰기 금지** (P-1 작업은 사용자 승인 시에만, 그 폴더의 세션에서)
- 진단고사 자료: `C:\Users\boria\Desktop\Vibe Coding\Project 2 (AP diagnostic test)` (E1에서 통합)

## 규칙
- **비밀 값**(키, DB 비밀번호)은 사용자가 `.env.local`에 직접 넣는다. 채팅에 붙여넣게 하지 않는다. `.env.local`은 커밋 금지 (`.env.example`만 커밋)
- **개발 DB에 실제 학생 데이터를 넣지 않는다** (테스트 계정·운영자만)
- DB 변경은 전부 `supabase/migrations/`의 마이그레이션으로. 대시보드에서만 하는 설정은 `docs/supabase_settings_checklist.md`에 기록
- 모든 테이블에 RLS. 학생은 문항·정답·채점 테이블을 직접 읽지 못한다 (설계안 5.4). 학생 데이터는 서버 API 경유
- 원자료(응답 이벤트, 답안)는 덮어쓰거나 지우지 않는다. 재채점·태그 수정은 이력으로 쌓는다
- 커밋은 사용자가 요청할 때만

## 반드시 리마인드할 항목
| 항목 | 시점 |
|---|---|
| **운영 DB `platform-prod` 생성 + Vercel Production 설정** (`docs/pilot_launch_checklist.md`) | 파일럿(P2.5) 직전 — 사용자가 "꼭 리마인드" 요청 |
| 기존 앱 운영 DB **완전 백업**(`pg_dump`) + 복원 시험 | 기존 DB를 건드리는 작업 전, 늦어도 전환 T-2일 |
| 2026-10-01 **평문 백업 암호화** 후 평문 삭제 (`Desktop\Vibe Coding\backups\frq-prod-20261001-1134`, 학생 개인정보 포함) | 주간 백업(P-1.2) 시작 전 |
| AP FRQ 3~5문항 **토큰 실측** + 비용 절감 검토 → 모델·예산 결정 | P3 시작(3.0) |
| 기존 앱 P-1: 학생 로그인 시 이름 덮어쓰기 제거 (코드 1곳), 주간 백업 | 사용자 승인 시 |

## 현재 상태 (2026-10-01)
- 브리프 0장 산출물 1~6 완료, 설계안 v4.1 확정
- 2027년 1월 목표: MCQ 모의고사 **내부 파일럿**. 전체 출시는 2~3월 예상
- **P0 진행 중**: 저장소·Next.js 초기화, Supabase CLI init + `platform-dev` link, `.env.local` 키 3개 확인(anon/service/Anthropic), auth 설정 원격 반영(JWT 15분·가입 차단) 완료. 남은 것: CI(0.5), 백업 스크립트 골격(0.6), 메일 발송(0.7, Q1 결정 필요), Vercel 연결(0.4, 선택)
