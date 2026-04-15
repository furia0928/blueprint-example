---
title: 인증 인프라 설정
---

# 인증 인프라 설정

GitHub 통합 구조, Supabase Auth 설정, 구현 스니펫, 보안 체크리스트. 도메인 규칙은 [features/AUTH.md](./features/AUTH.md), 플로우는 [flows/AUTH.md](./flows/AUTH.md) 참조.

## GitHub 통합 주체

플랫폼이 사용하는 OAuth App · GitHub App · Email magic link 3종의 용도·수명·관리 위치 및 용도별 토큰 선택(App vs OAuth) 규칙은 [INTEGRATION.md — GitHub 통합 주체](./INTEGRATION.md#github-통합-주체) 참조.

### OAuth scope: `repo` 결정 사유

- private 저장소의 이슈 생성/댓글/close에는 `repo` scope가 필수 (GitHub OAuth에 이슈 전용 scope가 없음)
- `public_repo`로는 public 저장소만 가능하므로 불충분
- GitHub App user-to-server 토큰으로 대체하면 scope를 줄일 수 있으나, Supabase Auth가 이 플로우를 네이티브 지원하지 않아 인증 직접 구현이 필요 → 복잡도 대비 이점이 적어 `repo` 유지
- 토큰은 서버에서만 사용하고 암호화 저장하여 유출 위험 최소화

## 관련 테이블

`profiles`, `app_installations`, `invitations`, `invitation_members` 테이블 스키마 및 RLS 정책은 [DB.md](./DB.md) 참조.

## Supabase 이메일 인증 설정

게스트 인증에 매직링크(OTP)와 이메일+비밀번호 두 가지 방식을 사용한다.

- **매직링크**: 최초 가입 시 이메일 소유 확인용
- **비밀번호**: 가입 완료 후 재로그인용 (`signInWithPassword`)

Supabase 프로젝트에서 다음 설정 필요:

1. **Authentication → Providers → Email** 활성화
2. **Confirm email** 켜기 (매직링크 방식)
3. **Site URL**: `https://blueprint-platform-pink.vercel.app` (프로덕션 도메인)
4. **Redirect URLs** (4개 등록):
   - `http://localhost:3000/api/auth/callback` — 로컬 개발 OAuth/매직링크 콜백
   - `http://localhost:3000/reset-password` — 로컬 개발 비밀번호 재설정
   - `https://blueprint-platform-pink.vercel.app/api/auth/callback` — Vercel OAuth/매직링크 콜백
   - `https://blueprint-platform-pink.vercel.app/reset-password` — Vercel 비밀번호 재설정
5. **이메일 템플릿**: 기본 템플릿 사용 (추후 커스터마이즈 가능)

> **참고**: 코드에서 `redirectTo`를 `window.location.origin` 기반으로 동적 생성하므로, 로컬이면 localhost로, Vercel이면 Vercel 도메인으로 자동 분기된다. Redirect URLs에 양쪽 다 등록되어 있어야 Supabase가 허용한다.

## 구현 스니펫

### 매직링크 발송 (게스트 최초 가입)

```ts
supabase.auth.signInWithOtp({
  email,
  options: { emailRedirectTo: `${origin}/api/auth/callback?next=/invite/${token}` }
})
```

### 비밀번호 로그인 (초대 페이지 + /login)

```ts
supabase.auth.signInWithPassword({ email, password })
```

### 비밀번호 설정 (registerGuest 서버측)

```ts
admin.auth.admin.updateUserById(userId, { password })
```

### 비밀번호 재설정 이메일 발송 (로그인 페이지 클라이언트측)

```ts
supabase.auth.resetPasswordForEmail(email, {
  redirectTo: `${origin}/reset-password`
})
```

### 새 비밀번호 설정 (/reset-password 클라이언트측)

```ts
// onAuthStateChange에서 PASSWORD_RECOVERY 이벤트 감지 후
supabase.auth.updateUser({ password: newPassword })
```

## 필요 라우트

- `src/app/api/auth/callback/route.ts` — OAuth/매직링크 콜백 공통
- `src/app/api/github/install/callback/route.ts` — GitHub App 설치 콜백
- `src/app/invite/[token]/page.tsx` — 게스트 초대 페이지
- `src/app/reset-password/page.tsx` — 비밀번호 재설정 페이지

## 보안 정책

- **Rate limiting**: 당장 미적용. 추후 필요 시 Vercel WAF 또는 미들웨어에서 처리
- **CSRF**: Server Action은 Next.js 기본 보호 (Origin 헤더 검증). Webhook은 서명 검증으로 보호
- **github_token 암호화**: Supabase Vault 사용 (pgsodium 위에 구축된 상위 API)

## 보안 체크리스트

- [ ] github_token 컬럼 암호화 (Supabase Vault 또는 pgsodium)
- [ ] RLS: github_token SELECT 차단 (모든 클라이언트)
- [ ] RLS: app_installations 클라이언트 접근 전면 차단
- [ ] Server Action에서만 service role로 토큰 접근
- [ ] GitHub App private key는 환경변수로만 관리
- [ ] 401 응답 시 토큰 자동 무효화 처리
- [ ] Webhook 수신 시 `X-Hub-Signature-256` HMAC 서명 검증
- [ ] Webhook secret 환경변수 관리
- [ ] 초대 토큰 만료 검증 (expire된 토큰으로 접근 차단)
- [ ] 뷰어는 초대된 프로젝트만 접근 가능 (RLS 또는 Server Action 검증)
- [ ] 뷰어의 쓰기 행동 차단 (코멘트, 설정 변경, 발행 등)

## 변경 이력

| 날짜 | 내용 |
|------|------|
| 2026-04-07 | 초안 작성. 토큰 저장 전략, RLS 정책, 로그인 플로우, 무효화 대응 확정 |
| 2026-04-07 | GitHub App 설치 콜백 플로우, provider_token 저장 실패 재시도, Installation 토큰 경합 대응, 사용자 플로우 요약 추가 |
| 2026-04-07 | OAuth scope `repo` 유지 결정 및 사유 명시 (이슈 전용 scope 부재, Supabase Auth 제약) |
| 2026-04-07 | 테이블 스키마 DB.md로 이동. SPEC.md에서 보안 정책 이동 |
| 2026-04-08 | 게스트 초대 플로우 추가. 이메일 매직링크 인증, 초대 링크 처리, 뷰어 권한 범위 정의 |
| 2026-04-08 | 오너 승인 플로우, Supabase 매직링크 설정 가이드, 게스트 플로우 상세화 (이름 입력 → pending → approved) |
| 2026-04-08 | 구현 반영: 게스트 비밀번호 인증 추가 (매직링크+비밀번호 등록+이메일 재로그인), 로그인 페이지 이메일+비밀번호 폼 추가, auth callback의 email provider 분기, AuthListener의 provider 체크, 미들웨어 viewer 접근 제어, registerGuest 오너 차단 및 기존 역할 보존, pending 페이지 이름+이메일 표시 |
| 2026-04-08 | 비밀번호 재설정 플로우 추가. 로그인 페이지 UX 재구성 (GitHub 기본 + 게스트 로그인 토글), 비밀번호 찾기 → resetPasswordForEmail → /reset-password 페이지 → updateUser |
| 2026-04-14 | AUTH.md를 rules(features/AUTH.md) + 플로우(flows/AUTH.md) + 설정(AUTH-SETUP.md)로 분리 |
