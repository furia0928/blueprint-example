---
domain: AUTH
title: 기능_인증
toc:
  - id: OAUTH_LOGIN
    label: GitHub OAuth 로그인
  - id: PASSWORD_LOGIN
    label: 게스트 비밀번호 로그인
  - id: PASSWORD_RESET
    label: 비밀번호 재설정
  - id: GUEST_INVITE
    label: 게스트 초대 링크
  - id: GUEST_REGISTER
    label: 게스트 등록
  - id: GUEST_APPROVAL
    label: 오너 승인 관리
  - id: VIEWER_ACCESS
    label: 뷰어 권한 범위
  - id: GUEST_COMMENT
    label: 게스트 코멘트 대리 작성
  - id: TOKEN_MANAGE
    label: GitHub 토큰 관리
  - id: GITHUB_APP_INSTALL
    label: GitHub App 설치
---

# AUTH (인증)

플랫폼의 인증 도메인. 내부 사용자(기획자/개발자/디자이너)는 GitHub OAuth, 외부 게스트(뷰어)는 이메일 매직링크+비밀번호로 구분된다.

- 플로우 다이어그램: [flows/AUTH.md](../flows/AUTH.md)
- 인프라 설정 (Supabase, GitHub 통합): [AUTH-SETUP.md](../AUTH-SETUP.md)

## OAUTH_LOGIN (GitHub OAuth 로그인)

**rules**
- OAuth 로그인은 제공자, 세션, 프로필 토큰을 가진다
- 내부 사용자(기획자/개발자/디자이너)는 GitHub OAuth로만 로그인한다
- OAuth scope는 `repo` — private 저장소 이슈 생성·댓글·close에 필수
- 콜백에서 Supabase 세션과 `provider_token`을 수집한다
- `provider_token`은 최초 로그인 시에만 반환되며, 세션 refresh 시에는 반환되지 않는다
- 로그인 성공 후 역할(role)이 없으면 역할 선택 화면으로 이동한다
- `next` 파라미터가 `/invite/`로 시작하면 역할 선택을 스킵한다

## PASSWORD_LOGIN (게스트 비밀번호 로그인)

**rules**
- 게스트는 이메일과 비밀번호로 재방문 로그인할 수 있다
- 게스트 로그인 폼은 로그인 페이지에서 기본 접힘 상태로, 내부 사용자 경로와 분리된다
- email provider는 GitHub 토큰 체크를 스킵한다
- 승인되지 않은(pending) 뷰어가 `/projects` 또는 `/onboarding`에 접근하면 `/invite/pending`으로 rewrite된다

## PASSWORD_RESET (비밀번호 재설정)

**rules**
- 비밀번호 재설정은 재설정 이메일, recovery 세션, 새 비밀번호를 가진다
- 로그인 페이지의 게스트 폼에서만 "비밀번호 찾기"에 진입한다
- 재설정 링크는 Supabase가 이메일로 발송하며 `/reset-password`로 돌아온다
- recovery 세션이 감지된 경우에만 새 비밀번호 입력을 허용한다
- 비밀번호 정책: 8자 이상 + 대문자 + 숫자 + 특수문자 (게스트 등록과 동일)
- 새 비밀번호와 확인이 일치해야 한다
- 변경 완료 후 자동 로그인하지 않고 로그인 페이지로 안내한다

## GUEST_INVITE (게스트 초대 링크)

**rules**
- 초대 링크는 토큰, 프로젝트, 만료일, 생성자를 가진다
- 프로젝트 오너만 초대 링크를 생성할 수 있다
- 만료되지 않은 기존 링크가 있으면 재사용한다
- 만료 기간은 7일이다
- 하나의 링크를 여러 게스트에게 공유할 수 있다
- 만료된 링크로 접근하면 만료 안내를 표시한다
- 무효한 토큰으로 접근하면 404를 표시한다

## GUEST_REGISTER (게스트 등록)

**rules**
- 게스트 등록은 이메일, 이름(`display_name`), 비밀번호, 멤버십 상태를 가진다
- 기존 계정 여부를 `checkGuestEmail`로 판별해 비밀번호 로그인 또는 매직링크로 분기한다
- 신규 이메일은 매직링크 발송 후 이메일 인증을 거친다
- 비밀번호 정책: 8자 이상 + 대문자 + 숫자 + 특수문자
- 프로젝트 오너 본인이 자기 프로젝트로 게스트 등록을 시도하면 거부한다
- `profiles.role`이 이미 설정되어 있으면 덮어쓰지 않는다. null인 경우에만 `viewer`로 설정한다
- 등록 완료 시 `invitation_members`에 `pending`으로 upsert (중복 없음)
- 등록 후 오너의 승인을 대기한다
- 재방문은 로그인 페이지의 이메일+비밀번호 폼으로 가능하다

## GUEST_APPROVAL (오너 승인 관리)

**rules**
- 멤버십은 상태(`pending`, `approved`), 가입자, 프로젝트, 가입일을 가진다
- 오너는 프로젝트 설정 시트에서 대기 중 게스트를 수락/거절할 수 있다
- 수락 시 상태는 `approved`가 되어 프로젝트 접근이 가능해진다
- 거절 시 `invitation_members`에서 삭제되며, 초대 링크를 다시 클릭하면 새로 pending 등록된다
- 승인 해제 시 `approved`에서 `pending`으로 되돌아간다
- 승인 해제된 게스트는 다시 승인 대기 목록으로 복귀한다

## VIEWER_ACCESS (뷰어 권한 범위)

**rules**
- 뷰어는 와이어프레임 열람, 화면명세 열람, 코멘트 작성을 할 수 있다
- 뷰어는 버전 발행, 프로젝트 설정 변경, 게스트 초대를 할 수 없다
- 뷰어의 파일 접근은 GitHub App Installation 토큰으로 서버에서 대리 처리한다
- 뷰어는 GitHub 저장소에 직접 접근하지 않는다
- 뷰어는 자신이 초대된 프로젝트에만 접근할 수 있다

## GUEST_COMMENT (게스트 코멘트 대리 작성)

**rules**
- 게스트 코멘트는 본문, 좌표, 실제 작성자 이메일, 대리 작성자(봇)를 가진다
- 게스트는 GitHub 토큰이 없으므로 GitHub App(봇)이 Installation 토큰으로 Issue를 대리 생성한다
- 이슈에는 `guest` 라벨이 자동 부착된다
- 이슈 본문에 실제 작성자 이름과 이메일을 표기한다
- 이슈 메타데이터(HTML 주석)에 `author_email`을 포함해 플랫폼 UI에서는 실제 작성자로 표시한다
- 이슈 작성자는 봇(GitHub App)으로 GitHub에 기록된다

## TOKEN_MANAGE (GitHub 토큰 관리)

**rules**
- 토큰은 제공자, 수명, 발급 시각, 만료 시각, 대상 사용자 또는 설치를 가진다
- OAuth 토큰은 만료되지 않으나 사용자 연동 해제·비밀번호 변경·직접 revoke 시 무효화된다
- Installation 토큰은 1시간 수명이며 자동 재발급된다
- OAuth 토큰은 `profiles`에 암호화 저장한다
- Installation 토큰은 `app_installations`에 저장하며 만료 시 GitHub App private key로 재발급한다
- GitHub API 호출이 401을 반환하면 OAuth 토큰을 무효화(`profiles.github_token = null`)하고 재로그인을 요청한다
- 콜백에서 `provider_token` 저장이 실패하면 즉시 3회 재시도하고, 실패 시 `prompt: 'consent'`로 강제 재인증한다
- Installation 토큰 갱신은 동시 요청 race condition 방지를 위해 DB 트랜잭션(`SELECT FOR UPDATE`)으로 처리한다
- 이미 다른 요청이 갱신한 토큰이 있으면 재발급 없이 재사용한다

## GITHUB_APP_INSTALL (GitHub App 설치)

**rules**
- 설치는 `installation_id`, 설치 주체, 대상 저장소 목록, 설치 시각을 가진다
- 프로젝트 생성 전에 GitHub App이 대상 저장소에 설치되어 있어야 한다
- 미설치 상태에서 프로젝트 생성 시도 시 GitHub App 설치 페이지로 이동시킨다
- 설치 완료 후 콜백(`/api/github/install/callback`)으로 `installation_id`를 수신해 `app_installations`에 저장한다
- 설치 후 프로젝트 생성 다이얼로그로 복귀한다
