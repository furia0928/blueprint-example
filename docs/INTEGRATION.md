---
title: GitHub 연동
---

# GitHub 연동

## GitHub 통합 주체

플랫폼은 세 가지 인증/연동 주체를 사용하며, 모두 `blueprint-emotion` org 소유이다.

| 통합 | 용도 | 토큰 수명 | 관리 위치 |
|------|------|----------|----------|
| OAuth App (Supabase Auth) | 내부 사용자 로그인, 이슈 생성/댓글 등 사용자 행동 | 만료 없음 (revoke 전까지) | `github.com/organizations/blueprint-emotion/settings/applications` |
| GitHub App (설치형) | 파일 읽기, Webhook, 태그 생성 등 시스템 행동 | 1시간 (자동 재발급) | `github.com/organizations/blueprint-emotion/settings/apps` |
| 이메일 매직링크 (Supabase Auth) | 게스트(viewer) 로그인, GitHub 계정 불필요 | Supabase 세션 기본 수명 | — |

> 인증 플로우·토큰 저장·무효화 정책 등 상세 설계는 [features/AUTH.md](./features/AUTH.md) 참조. Supabase 이메일 설정·OAuth scope 선택 사유·구현 스니펫은 [AUTH-SETUP.md](./AUTH-SETUP.md) 참조.

## App vs OAuth 역할 분담

원칙: **사람의 행동은 사용자 OAuth 토큰, 시스템의 행동은 GitHub App 토큰**

| 용도 | 토큰 | 이유 |
|------|------|------|
| 파일 읽기 (Contents API) | App | 저장소에 설치된 앱 권한 |
| Webhook 수신 | App | 앱 단위 동작 |
| 태그 생성 (버전 발행) | App | 저장소 쓰기 |
| collaborator 목록 조회 | App | 사용자 권한과 무관 |
| 이슈 생성/댓글 | 사용자 OAuth | 실제 작성자로 기록, @멘션 알림 |
| 이슈 close (해결) | 사용자 OAuth | 누가 해결했는지 기록 |
| 접근 가능 저장소 조회 | 사용자 OAuth | 본인 기준 목록 |

## GitHub App 권한

| 권한 | 수준 | 용도 |
|------|------|------|
| Contents | Read | 와이어프레임 HTML, 화면명세·기능명세 MD 파일 읽기 |
| Issues | Read & Write | 코멘트 생성/조회, 라벨 붙이기 |
| Metadata | Read | 저장소 기본 정보 (필수) |
| Webhooks | Read & Write | push 이벤트 수신 → 와이어프레임 변경 감지 |
| Email addresses | Read | 사용자 식별, @멘션 알림 |

## Webhook

- 기획자가 push 시 GitHub Webhook → 플랫폼 API 수신
- 해당 화면을 보고 있는 사용자에게 토스트 알림: "와이어프레임이 업데이트됐습니다 [새로고침]"
- 사용자가 직접 새로고침 결정 (자동 새로고침 안 함)
- Webhook push 이벤트는 파일 캐시 무효화 트리거로도 사용 (아래 캐싱 전략 참조)
- Webhook `issues` 이벤트 수신: 이슈 close/reopen/delete 시 `comments.status` 동기화
- **서명 검증 필수**: `X-Hub-Signature-256` 헤더로 HMAC 검증. 실패 시 요청 무시 (403). Webhook secret은 환경변수로 관리

## GitHub API 캐싱 전략

GitHub API rate limit: App 토큰 5,000/hr, OAuth 토큰 5,000/hr.
다수 사용자가 같은 와이어프레임을 열면 빠르게 소진될 수 있으므로 캐싱 필수.

**캐시 저장소:** Supabase DB `github_cache` 테이블 (스키마는 [DB.md](./DB.md) 참조)

**API별 전략:**

| API | 캐싱 | 무효화 |
|-----|------|--------|
| Contents (HTML/MD) | DB 캐시 + ETag | Webhook push 시 해당 저장소 캐시 삭제 |
| Issues (코멘트 목록) | 캐시 없음 | Supabase Realtime이 실시간 동기화 담당 |
| Collaborators | 5분 TTL | TTL 만료 시 재조회 |
| Labels | 15분 TTL | TTL 만료 시 재조회 |
| Tags (버전 목록) | DB 캐시 | 버전 발행 시 무효화 |

**조회 흐름 (Contents API):**
1. `github_cache`에서 `cache_key`로 조회
2. 캐시 있음 → ETag로 conditional request (`If-None-Match` 헤더)
   - 304 → 캐시 본문 반환 (API 응답 작아서 대역폭 절약)
   - 200 → 캐시 갱신 후 반환
3. 캐시 없음 → 일반 요청 → 응답 + ETag 캐시 저장

**캐시 정리:**
- 30일 이상 된 캐시 자동 삭제 (Cron Job, 1일 1회)
- 프로젝트 삭제 시 해당 저장소 캐시도 삭제

## 에러 대응

**GitHub API 에러:**

| 상태 코드 | 원인 | UI 동작 |
|-----------|------|---------|
| 401 | OAuth 토큰 무효화 | 토큰 삭제 + "GitHub 재연결 필요" 안내 → 재로그인 |
| 403 | 권한 없음 / rate limit 초과 | rate limit이면 "잠시 후 다시 시도" + 재시도 가능 시각 표시. 권한이면 "접근 권한이 없습니다" |
| 404 | 저장소 삭제됨 / 파일 없음 | 저장소 삭제: "저장소를 찾을 수 없습니다" 안내. 파일 없음: 해당 화면 목록에서 제외 |
| 5xx | GitHub 서버 에러 | "GitHub 서버에 문제가 있습니다. 잠시 후 다시 시도해주세요" |
| 네트워크 에러 | 연결 실패 | "네트워크 연결을 확인해주세요" + 재시도 버튼 |

**GitHub App 에러:**

| 상황 | 감지 방법 | UI 동작 |
|------|----------|---------|
| App 미설치 상태에서 프로젝트 생성 | Installation API 조회 시 404 | GitHub App 설치 페이지로 안내 |
| App이 저장소에서 uninstall됨 | Contents API 호출 시 403 + `installation_id` 조회 실패 | "GitHub App이 제거되었습니다. 다시 설치해주세요" + 설치 링크 |
| App 권한 변경 (Contents Read 제거 등) | API 호출 시 403 | "GitHub App 권한을 확인해주세요" |

---

## 변경 이력

| 날짜 | 내용 |
|------|------|
| 2026-04-07 | SPEC.md에서 분리. App vs OAuth 역할, App 권한, Webhook, 캐싱 전략 포함 |
| 2026-04-07 | SPEC.md에서 에러 대응(GitHub API/App) 이동. github_cache 스키마 DB.md로 이동 |
