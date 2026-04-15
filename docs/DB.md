---
title: DB 스키마
---

# DB 스키마

> Supabase (PostgreSQL). DB 접근은 모두 Server Action 경유 (클라이언트 직접 DB 접근 금지). 클라이언트는 Auth + Realtime 구독만 직접 사용.

## profiles

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | uuid (PK, FK → auth.users.id) | 사용자 ID |
| display_name | text | 표시 이름 (한글 등). 코멘트 멘션, UI 표시에 사용 |
| github_username | text | GitHub 사용자명 |
| github_token | text (encrypted) | GitHub OAuth 토큰. Vault 또는 pgsodium으로 암호화 |
| role | text | `planner`(기획자) / `developer`(개발자) / `designer`(디자이너) / `viewer`(게스트). UI 라벨 매핑은 [GLOSSARY.md](./GLOSSARY.md#사용자역할) |
| created_at | timestamptz | 생성일 |
| updated_at | timestamptz | 수정일 |

**RLS 정책:**
- `github_token` 컬럼은 본인 포함 모든 클라이언트에서 SELECT 불가
- Server Action에서 service role로만 접근
- 본인의 `display_name`, `github_username`, `role`은 읽기/수정 가능

## app_installations

| 컬럼 | 타입 | 설명 |
|------|------|------|
| installation_id | bigint (PK) | GitHub App 설치 ID |
| token | text | Installation access token |
| expires_at | timestamptz | 토큰 만료 시각 |

**RLS 정책:**
- 클라이언트 접근 전면 차단
- Server Action에서 service role로만 접근

## projects

- id, name (프로젝트 이름, 기본값: 저장소명), github_repo (owner/repo), last_published_sha (text, nullable — 마지막 발행 태그의 커밋 SHA), last_published_at (timestamptz, nullable — 마지막 발행 시각), created_by, created_at

## comments

- id
- project_id (FK > projects)
- github_issue_number
- title (이슈 제목, 핀 뱃지 + 목록 표시용)
- screen_id (그룹/폴더명, e.g. "상품/PRODUCT-LIST". 그룹 없으면 폴더명만, e.g. "STANDALONE")
- version (Git 태그, e.g. "v1.0" 또는 "latest")
- feature_id (e.g. "QNA__LIST__FILTER")
- page (와이어프레임 파일명, e.g. "product-list_wireframe-pc.html". PC/MO 분리 모드에서 뷰포트별 필터링에 사용)
- x, y (feature div 기준 상대 비율 좌표 0~1, div 축소 시 클램핑)
- commit_sha (text, nullable — 코멘트 생성 시점의 HEAD SHA. 와이어프레임 변경 후 좌표 불일치 감지용)
- status (open / closed)
- created_by
- created_at

## project_favorites

- user_id (FK → profiles.id)
- project_id (FK → projects)
- created_at
- PK: (user_id, project_id)

## github_cache

| 컬럼 | 타입 | 설명 |
|------|------|------|
| cache_key | text (PK) | `{owner}/{repo}:{path}@{ref}` |
| content | text | 응답 본문 |
| etag | text | GitHub API 응답의 ETag 헤더값 |
| cached_at | timestamptz | 캐시 시각 |

## invitations

프로젝트별 초대 링크. 하나의 링크를 여러 게스트에게 공유할 수 있다.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | uuid (PK) | 초대 링크 ID |
| project_id | uuid (FK → projects) | 초대 대상 프로젝트 |
| token | uuid (UNIQUE) | 초대 링크용 고유 토큰 |
| invited_by | uuid (FK → profiles) | 링크 생성한 사용자 (프로젝트 오너) |
| expires_at | timestamptz | 링크 만료 시각 (기본 7일) |
| created_at | timestamptz | 생성일 |

**RLS 정책:**
- 초대 링크 생성/삭제는 프로젝트 오너(created_by)만 가능
- 링크 조회는 프로젝트 오너만 가능
- token으로 invitation 조회 (토큰 검증): Server Action에서 service role로 처리 (RLS 우회)

## invitation_members

초대 링크를 통해 접속한 게스트 목록. 오너 승인 후 프로젝트 접근 가능.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| invitation_id | uuid (FK → invitations) | 사용한 초대 링크 |
| user_id | uuid (FK → profiles) | 접속한 게스트 |
| status | text default 'pending' | pending / approved |
| joined_at | timestamptz | 최초 접속 시각 |
| PK | (invitation_id, user_id) | |

**RLS 정책:**
- 뷰어는 본인 레코드만 조회 가능
- INSERT는 유효한 초대 링크로 접속 시 자동 생성 (Server Action)
- status 변경은 프로젝트 오너만 가능

---

## 멤버 관리

내부 사용자 멤버 정보는 저장하지 않음. 사용자 로그인 시 GitHub API로 접근 가능한 저장소 목록을 조회하고, DB의 프로젝트와 매칭하여 표시.
GitHub에서 collaborator 추가/제거하면 플랫폼에 자동 반영 (DB 업데이트 불필요).

### 게스트 프로젝트 조회 경로

게스트(viewer)는 GitHub collaborator가 아니므로 별도 경로로 프로젝트를 조회한다:

```
profiles.role = 'viewer'인 경우:
  invitation_members (user_id = 본인, status = 'approved')
  → invitations (invitation_id)
  → projects (project_id)
```

`getProjects` Server Action에서 role이 viewer이면 위 경로로 프로젝트 목록을 조회한다.

---

## 변경 이력

| 날짜 | 내용 |
|------|------|
| 2026-04-07 | SPEC.md, AUTH.md, INTEGRATION.md에서 분리. 전체 테이블 스키마 통합 |
| 2026-04-07 | projects에 last_published_sha/last_published_at, comments에 commit_sha 추가 (미발행 변경 감지) |
| 2026-04-08 | invitations + invitation_members 테이블 추가, profiles.role에 viewer 추가 (게스트 초대 기능). 링크 1개를 여러 명이 공유하는 구조 |
