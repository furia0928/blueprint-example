---
domain: AUTH
title: 플로우_인증
---

# 인증 플로우

화면과 기능을 가로지르는 인증 journey. 규칙은 [features/AUTH.md](../features/AUTH.md), 인프라 설정은 [AUTH-SETUP.md](../AUTH-SETUP.md) 참조.

## 내부 사용자 로그인 (GitHub OAuth)

```mermaid
flowchart TD
    A["로그인 페이지"] --> B["GitHub로 로그인"]
    B --> C["GitHub OAuth 동의<br/>scope: repo"]
    C --> D["auth/callback"]
    D --> E{"code↔session<br/>교환 성공?"}
    E -- 실패 --> F["login 에러"]
    E -- 성공 --> G{"provider =<br/>github?"}
    G -- github --> H{"provider_token<br/>존재?"}
    H -- 있음 --> I["github_token 저장<br/>3회 재시도"]
    I --> J{"저장 성공?"}
    J -- 실패 --> K["login 에러<br/>token_save_failed"]
    J -- 성공 --> L{"profiles.role<br/>존재?"}
    H -- 없음 --> M{"DB에 기존<br/>github_token?"}
    M -- 있음 --> L
    M -- 없음 --> N["login 에러<br/>prompt: consent 재인증"]
    G -- email --> L
    L -- 없음 --> O{"next가<br/>/invite로 시작?"}
    O -- 아님 --> P["onboarding<br/>역할 선택"]
    P --> Q["projects"]
    O -- 맞음 --> R["next 경로로<br/>리다이렉트"]
    L -- 있음 --> Q
```

- `signInWithOAuth` → GitHub 동의 화면 (scope: `repo`)
- 콜백에서 Supabase 세션 생성 + `provider_token` 수집
- email provider는 GitHub 토큰 체크를 스킵 (게스트 매직링크 경로)
- `/invite/`로 시작하는 next가 있으면 온보딩 스킵

## provider_token 누락 대응

```mermaid
flowchart TD
    A["onAuthStateChange"] --> B{"provider =<br/>email?"}
    B -- email --> C["GitHub 토큰 체크 스킵"]
    B -- github --> D{"provider_token<br/>존재?"}
    D -- 있음 --> E["Server Action으로<br/>profiles에 저장"]
    D -- 없음 --> F{"profiles에<br/>기존 토큰 있음?"}
    F -- 있음 --> G["기존 토큰 사용"]
    F -- 없음 --> H["prompt: consent<br/>강제 재인증"]
```

Supabase Auth는 `provider_token`을 최초 로그인 시에만 반환한다. 세션 refresh 시에는 반환하지 않으므로, 기존 토큰이 없으면 `prompt: 'consent'`로 강제 재인증한다.

## provider_token 저장 실패 대응

```mermaid
flowchart TD
    A["콜백에서 토큰 저장 시도"] --> B{"저장 성공?"}
    B -- 성공 --> C["완료"]
    B -- 실패 --> D{"재시도 횟수<br/>3회 미만?"}
    D -- 미만 --> A
    D -- 3회 도달 --> E["prompt: consent<br/>강제 재인증"]
```

Supabase는 `provider_token`을 재발급하지 않으므로 즉시 3회 재시도하고, 실패 시 강제 재인증으로 복구한다.

## 게스트 초대 — 오너 측

```mermaid
flowchart TD
    A["프로젝트 설정 시트"] --> B["초대 링크 생성 클릭"]
    B --> C{"미만료 기존<br/>링크 있음?"}
    C -- 있음 --> D["기존 링크 표시"]
    C -- 없음 --> E["새 링크 생성<br/>7일 만료"]
    E --> D
    D --> F["링크 복사<br/>카카오톡, 슬랙, 이메일로 전달"]

    G["승인 대기 목록"] --> H{"오너 액션"}
    H -- 수락 --> I["status: approved<br/>프로젝트 접근 가능"]
    H -- 거절 --> J["invitation_members 삭제<br/>재등록 가능"]
    H -- 승인해제 --> K["approved → pending<br/>대기 목록 복귀"]
```

## 게스트 초대 — 게스트 측

```mermaid
flowchart TD
    A["초대 링크 클릭<br/>invite/token"] --> B{"토큰 유효?"}
    B -- 무효 --> C["404"]
    B -- 만료 --> D["만료 안내"]
    B -- 유효 --> E{"로그인 상태?"}
    E -- 미로그인 --> F["이메일 입력 + [다음]"]
    F --> G{"checkGuestEmail<br/>기존 계정?"}
    G -- 기존 --> H["비밀번호 입력<br/>(이메일 disabled)"]
    H --> I["signInWithPassword"]
    I --> J["로그인 성공<br/>페이지 리로드"]
    J --> E
    G -- 신규 --> K["signInWithOtp<br/>매직링크 발송"]
    K --> L["이메일 확인 안내"]
    L --> M["매직링크 클릭"]
    M --> N["auth/callback<br/>email provider, GitHub 토큰 스킵"]
    N --> O["invite/token 리다이렉트"]
    O --> E
    E -- 로그인됨 --> P{"display_name<br/>있음?"}
    P -- 없음 --> Q["이름 + 비밀번호<br/>+ 비밀번호 확인 입력"]
    Q --> R["registerGuest<br/>role: viewer, 비밀번호 설정,<br/>status: pending"]
    R --> S["승인 대기 화면"]
    P -- 있음 --> T{"invitation_members<br/>상태?"}
    T -- pending --> S
    T -- approved --> U["프로젝트 리다이렉트"]
    T -- 미등록 --> Q
```

## 게스트 재방문 — 이메일+비밀번호 로그인

```mermaid
flowchart TD
    A["/login 페이지"] --> B["게스트 로그인 폼 펼침"]
    B --> C["이메일 + 비밀번호 입력"]
    C --> D["signInWithPassword"]
    D --> E{"로그인 성공?"}
    E -- 성공 --> F{"role = viewer<br/>approved?"}
    F -- approved --> G["프로젝트 접근"]
    F -- 미승인 --> H["미들웨어에서<br/>/invite/pending rewrite"]
    E -- 실패 --> I["에러 표시"]
```

게스트 로그인 폼은 기본 접힘 상태. 내부 사용자(GitHub 로그인)가 혼동하지 않도록 토글로 분리.

## 비밀번호 재설정

```mermaid
flowchart TD
    A["로그인 페이지<br/>게스트 폼 열림"] --> B["비밀번호 찾기 클릭"]
    B --> C["재설정 모드 전환<br/>이메일 필드만 표시"]
    C --> D["이메일 입력 +<br/>재설정 링크 발송"]
    D --> E["resetPasswordForEmail"]
    E --> F["이메일 발송 완료 안내"]
    F --> G["게스트가 이메일 링크 클릭"]
    G --> H["/reset-password<br/>#access_token=..."]
    H --> I["onAuthStateChange<br/>PASSWORD_RECOVERY 감지"]
    I --> J["새 비밀번호 + 확인 입력"]
    J --> K["updateUser({ password })"]
    K --> L{"변경 성공?"}
    L -- 성공 --> M["완료 안내 +<br/>로그인하러 가기"]
    M --> N["/login"]
    L -- 실패 --> O["에러 표시"]
```

recovery 세션 제약으로 변경 후 자동 로그인하지 않고 `/login`으로 안내한다.

## 토큰 무효화 대응 (OAuth 토큰)

```mermaid
flowchart TD
    A["GitHub API 호출"] --> B["401 응답"]
    B --> C["profiles.github_token = null"]
    C --> D["GitHub 연결 해제 안내"]
    D --> E["재로그인"]
    E --> F["provider_token 재수집"]
    F --> G["profiles 업데이트"]
```

GitHub OAuth 토큰은 만료되지 않지만 사용자가 연동 해제·비밀번호 변경·직접 revoke 시 무효화된다.

## Installation 토큰 갱신 (GitHub App)

```mermaid
flowchart TD
    A["클라이언트"] --> B["Server Action"]
    B --> C["app_installations에서<br/>token 조회"]
    C --> D{"expires_at<br/>지남?"}
    D -- 지남 --> E["SELECT FOR UPDATE<br/>DB 트랜잭션"]
    E --> F{"이미 갱신된<br/>토큰 존재?"}
    F -- 있음 --> G["갱신된 토큰 재사용"]
    F -- 없음 --> H["GitHub App private key로<br/>새 토큰 발급"]
    H --> I["DB 갱신 + 커밋"]
    I --> G
    D -- 안 지남 --> J["캐시 사용"]
    G --> K["GitHub API 호출"]
    J --> K
```

Installation 토큰은 1시간 만료. 동시 요청 시 race condition 방지를 위해 트랜잭션으로 갱신한다.

## GitHub App 설치

```mermaid
flowchart TD
    A["새 프로젝트 클릭"] --> B{"GitHub App<br/>설치됨?"}
    B -- 설치됨 --> C["저장소 선택 가능"]
    B -- 미설치 --> D["GitHub App<br/>설치 페이지로 이동"]
    D --> E["저장소 선택 후<br/>설치 완료"]
    E --> F["설치 콜백 URL로 redirect<br/>/api/github/install/callback"]
    F --> G["installation_id 수신<br/>app_installations에 저장"]
    G --> H["프로젝트 생성<br/>다이얼로그로 복귀"]
    H --> C
```

## 미들웨어 접근 제어 (viewer)

```mermaid
flowchart TD
    A["요청 진입"] --> B{"보호 경로?<br/>projects, onboarding"}
    B -- 아님 --> Z["통과"]
    B -- 맞음 --> C{"로그인?"}
    C -- 미로그인 --> D["login 리다이렉트<br/>next=원래경로"]
    C -- 로그인 --> E{"role = viewer?"}
    E -- 아님 --> Z
    E -- viewer --> F{"approved 멤버십<br/>있음?"}
    F -- 있음 --> Z
    F -- 없음 --> G["invite/pending<br/>rewrite"]
```

viewer(pending) 상태의 사용자가 `/projects` 또는 `/onboarding`에 접근하면 `/invite/pending`으로 rewrite된다. approved된 viewer는 정상 통과.

## 게스트 코멘트 대리 생성

```mermaid
flowchart TD
    A["게스트가 코멘트 작성"] --> B["Server Action"]
    B --> C["GitHub App<br/>Installation 토큰"]
    C --> D["Issue 생성"]
    D --> E["라벨: guest 자동 부착"]
    D --> F["본문에 실제 작성자 표기"]
```

게스트(viewer)는 GitHub 토큰이 없으므로 GitHub App(봇)이 Installation 토큰으로 Issue를 대리 생성한다. 이슈 본문에 실제 작성자 이메일을 메타데이터로 포함하여 플랫폼 UI에서는 실제 작성자로 표시.
