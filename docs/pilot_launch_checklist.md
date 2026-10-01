# 파일럿 런칭 체크리스트 — 운영 DB(platform-prod)와 Vercel

> 2026-10-01 작성 · 개발 계획 P2.5.0 · **파일럿 시작 전에 반드시 실행**
> 이 문서는 새 프로젝트 저장소의 `docs/`로 복사되고, `CLAUDE.md`에서 가리킨다 (세션이 바뀌어도 리마인드되도록).

---

## 1. 선생님이 하실 일 (웹)

| # | 어디서 | 할 일 |
|---|---|---|
| 1 | Supabase | 새 프로젝트 **`platform-prod`** 생성. 리전 **Seoul (ap-northeast-2)**. **Enable automatic RLS 켬**, Data API 켬, 나머지 기본값 (`platform-dev`와 동일). **Pro 요금제** 권장 (일일 백업). DB 비밀번호는 생성 전에 비밀번호 관리자에 보관 |
| 2 | Supabase | Project Settings → API에서 URL, anon key, service role key를 확인 (`.env`/Vercel에 직접 입력, 채팅에 붙여넣지 않기) |
| 3 | Vercel | 요금제 확인 (상업 이용이면 **Pro**). 프로젝트 → Settings → Environment Variables에서 **Production** 값을 운영 DB로 교체 (아래 3장) |
| 4 | 도메인 (선택) | 제품 이름·도메인이 정해졌으면 Vercel에 도메인 연결 (Q7) |
| 5 | 메일 발송 서비스 | 운영용 발송 도메인 인증 (SPF/DKIM) 확인 |
| 6 | CLI 로그인 (필요 시) | `! npx supabase login`, `! npx vercel login` |

---

## 2. Claude가 할 일

| # | 작업 | 확인 |
|---|---|---|
| 1 | `supabase link --project-ref <prod>` → **`supabase db push`**로 마이그레이션 전체 적용 | `supabase db diff` 결과 개발 DB와 차이 0 |
| 2 | **설정 체크리스트**(P0.3에서 작성)대로 대시보드 설정 재현: Custom Access Token Hook, JWT 15분(900초), SMTP, 사이트 URL·리디렉트 URL, Storage 버킷·정책, 비밀번호 정책 | 체크리스트 항목 전부 체크 |
| 3 | 기관 1개 등록 (`scripts/seed-org`), 최고 관리자 계정 초대 | 선생님 계정으로 로그인 성공 |
| 4 | AP Chem 기준 데이터 시드 (과목 계층, CED 단원, Science Practices, Taxonomy v0.1) | 조회 화면 확인 |
| 5 | 파일럿용 문항·시험 세트를 운영 DB에 등록 (개발 DB에서 검수한 것을 스크립트로 옮김) | 문항 수·그림 일치 |
| 6 | **백업 체계 적용**: 주간 `pg_dump` → 암호화 → 2차 저장소, 보존 기간 자동 삭제 | 첫 암호화 백업 생성 |
| 7 | **복원 훈련 1회**: 운영 백업 → 빈 프로젝트 복원 → 행 수·해시 확인 | 통과 |
| 8 | **RLS 테스트를 운영 DB 대상으로 실행** | 전부 통과 |
| 8.5 | **Vercel 함수 리전을 Seoul(`icn1`)로 설정** (`vercel.json` 또는 vercel.ts의 regions). DB가 Seoul이므로 함수도 같은 리전에 둬야 DB 왕복이 짧다. 기존 앱은 DB·함수 모두 도쿄(hnd1)로 맞춰져 있었음 | 배포 후 응답 헤더 `x-vercel-id`에 icn1 확인 |
| 9 | Vercel 환경변수 확인: Production → 운영 DB, Preview·Development → 개발 DB 유지 | 운영 URL은 운영 DB, 미리보기 URL은 개발 DB를 사용 (실제 요청으로 확인) |
| 10 | 운영 배포 → 스모크 테스트 (4개 역할 로그인, 응시 1회, 채점, 결과) | 오류 0 |

---

## 3. Vercel 환경변수 (Production만 운영 값)

| 변수 | Production | Preview / Development |
|---|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | platform-prod URL | platform-dev URL |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | prod anon key | dev anon key |
| `SUPABASE_SERVICE_ROLE_KEY` | prod service role key | dev service role key |
| `ANTHROPIC_API_KEY` | 새 프로젝트 키 | 같은 키 또는 개발용 키 |
| `NEXT_PUBLIC_SITE_URL` | 운영 도메인 | 미리보기 URL |
| 메일 발송 키, `CRON_SECRET` 등 | 운영 값 | 개발 값 |

---

## 4. 파일럿 시작 전 나머지 (개발 계획 P2.5.1)
- 1개 반 학생 초대 (엑셀)
- **보호자 동의 기록**
- 학생·보호자에게 **"응시 중 화면 이탈·네트워크 끊김이 기록됨"** 사전 안내
- 지원 기기·브라우저 안내 (Q16), 화면 자동 꺼짐 안내
