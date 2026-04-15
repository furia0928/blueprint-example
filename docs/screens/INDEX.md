---
title: 화면 목록
---

# 화면 목록

## 페이지

| # | 화면 | 라우트 | 파일 | 비고 |
|---|------|--------|------|------|
| 1 | 랜딩 | `/` | [screen.md](./landing/screen.md) | 비로그인 진입점. 로그인 시 `/projects` 리다이렉트 |
| 2 | 로그인 | `/login` | [screen.md](./auth/login/screen.md) | GitHub 소셜 로그인 |
| 3 | 역할 선택 | `/onboarding` | [screen.md](./auth/onboarding/screen.md) | 로그인 후 역할 미선택 시 리다이렉트 |
| 4 | 비밀번호 재설정 | `/reset-password` | [screen.md](./auth/reset-password/screen.md) | 매직링크 진입 |
| 5 | 게스트 초대 | `/invite/[token]` | [screen.md](./auth/invite/screen.md) | 토큰 검증 → 매직링크 → 이름 입력 → 승인 대기 |
| 6 | 승인 대기 | `/invite/pending` | [screen.md](./auth/invite-pending/screen.md) | pending viewer가 보호 경로 접근 시 미들웨어 rewrite |
| 7 | 프로젝트 목록 | `/projects` | [screen.md](./project/list/screen.md) | 카드 목록, 즐겨찾기 — 하위 다이얼로그 1 (↓) |
| 8 | 화면 상세 | `/projects/[id]/screens/[screenId]` | [screen.md](./project/detail/screen.md) | 2패널 — 영역 3 / 시트 2 / 다이얼로그 2 (↓) |
| 9 | 권한 없음 | — | [screen.md](./auth/forbidden/screen.md) | collaborator 아닌 사용자가 프로젝트 접근 시 |
| 10 | 404 | — | [screen.md](./not-found/screen.md) | 존재하지 않는 경로 |

> `/projects/[id]`는 첫 화면으로 리다이렉트되므로 별도 페이지 없음.

## 페이지별 구성 (영역 / 시트 / 다이얼로그)

### 7. 프로젝트 목록 (`/projects`)

| 종류 | 파일 | 트리거·설명 |
|---|---|---|
| 다이얼로그 | [dialog_create.md](./project/list/dialog_create.md) | "새 프로젝트" 버튼 — 저장소 선택. GitHub App 미설치 시 설치 유도 |

### 8. 화면 상세 (`/projects/[id]/screens/[screenId]`)

| 종류 | 파일 | 트리거·설명 |
|---|---|---|
| 영역 | [area_top-bar.md](./project/detail/area_top-bar.md) | 브레드크럼, 버전 드롭다운, 발행, 설정 |
| 영역 | [area_file-tree.md](./project/detail/area_file-tree.md) | 좌측 사이드바 — 그룹/화면 트리 |
| 영역 | [area_viewer.md](./project/detail/area_viewer.md) | 중앙 뷰어 — HTML(iframe + 💬 + 핀) / MD(react-markdown) 자동 전환 |
| 시트 | [sheet_comments.md](./project/detail/sheet_comments.md) | 뷰어 툴바 💬 — 코멘트 목록·필터·핀 하이라이트·상세·수정·작성·close |
| 시트 | [sheet_settings.md](./project/detail/sheet_settings.md) | 탑바 톱니 — 이름 변경, 삭제 (Admin) |
| 다이얼로그 | [dialog_comment.md](./project/detail/dialog_comment.md) | 기능 영역 클릭(작성) / 코멘트 [수정] — Issue 생성·PATCH |
| 다이얼로그 | [dialog_version-publish.md](./project/detail/dialog_version-publish.md) | 발행 버튼 — 메이저/마이너 선택, 미해결 코멘트 경고 |

### 글로벌 (모든 로그인 페이지)

| 종류 | 파일 | 트리거·설명 |
|---|---|---|
| 시트 | [sheet_profile.md](./common/sheet_profile.md) | 헤더 프로필 아이콘 — 역할 변경, GitHub 사용자 정보 |

> **우측 기능 패널 폐기 (2026-04-13)**: 기능 설명은 와이어프레임 HTML에 `<bp-description>`으로 임베드되어 iframe 내부 우측 레일로 렌더된다. 상세: [features/WIREFRAME.md](../features/WIREFRAME.md#embed-기능-설명-임베드).
