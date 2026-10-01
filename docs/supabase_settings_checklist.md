# Supabase 설정 체크리스트

> 마이그레이션 파일에 남지 않는 프로젝트 설정 목록. `platform-dev`에 적용한 그대로 `platform-prod`(파일럿 직전)에 재현한다.
> `supabase/config.toml`로 관리되는 항목은 `npx supabase config push`로 원격에 반영할 수 있다. 나머지는 대시보드에서 수동으로 설정한다.

| # | 항목 | 값 | 관리 방식 | dev | prod |
|---|---|---|---|---|---|
| 1 | 리전 | Seoul (ap-northeast-2) | 생성 시 | ✅ | ☐ |
| 2 | Enable automatic RLS | 켬 | 생성 시 | ✅ | ☐ |
| 3 | Data API | 켬 | 생성 시 | ✅ | ☐ |
| 4 | JWT 유효 시간 | 900초 (15분) | config.toml `[auth] jwt_expiry` | ✅ | ☐ |
| 5 | 공개 가입 | 끔 (초대 방식) | config.toml `[auth] enable_signup=false` | ✅ | ☐ |
| 6 | Site URL / 리디렉트 URL | dev: `http://localhost:3000` + `http://localhost:3000/**` (Vercel 미리보기 URL은 연결 시 추가) / prod: 운영 도메인 | config.toml `[auth]` + 대시보드 | ✅ (로컬만) | ☐ |
| 7 | Custom Access Token Hook | `public.custom_access_token_hook` (P1.2에서 작성) | 대시보드 Auth → Hooks | ☐ | ☐ |
| 8 | SMTP (메일 발송) | Q1 결정 후 | 대시보드 Auth → SMTP | ☐ | ☐ |
| 9 | 메일 템플릿 (초대, 비밀번호 재설정) | 한국어 문안 | 대시보드 또는 config.toml | ☐ | ☐ |
| 10 | 비밀번호 정책 | 최소 길이 등 (P1.4에서 결정) | config.toml / 대시보드 | ☐ | ☐ |
| 11 | Storage 버킷 | 마이그레이션으로 생성 | 마이그레이션 | ☐ | ☐ |
| 12 | 백업 | prod: Pro 일일 백업 + 주간 암호화 백업 (설계안 7.1) | 대시보드 + 스크립트 | — | ☐ |

| 13 | 기타 Auth 값 (원격 기본값 유지) | 이메일 인증 켬, 메일 발송 간격 1분, OTP 8자리, TOTP MFA 등록·검증 켬(화면은 F) | config.toml | ✅ | ☐ |

**반영 방법:** `npx supabase config push` → 바뀌는 항목을 확인하고 y. prod는 `supabase link --project-ref <prod>` 후 같은 명령 (리디렉트 URL은 운영 도메인으로 바꿔서).

> 진행하면서 항목이 생기면 이 표에 추가한다.
