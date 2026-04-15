---
screenId: PROJECT-DETAIL
title: 화면_화면 상세
type: page
purpose: 프로젝트의 와이어프레임을 열람하고 핀 코멘트로 피드백하며 화면명세를 확인하는 핵심 작업 화면
viewport: [pc]
features: [WIREFRAME, COMMENT, VERSION]
group: project
---

# 화면 상세

> 라우트: `/projects/[id]/screens/[screenId]`
> 인증: 필수 (비로그인 시 `/login?next=원래경로`)
> 쿼리: `?version=v1.0` `?comment=123`
> 상세: [features/WIREFRAME.md](../../../features/WIREFRAME.md), [features/COMMENT.md](../../../features/COMMENT.md), [features/VERSION.md](../../../features/VERSION.md)

## 화면 구성

상단 헤더 + 좌우 2패널. 우측 기능 패널은 **플랫폼이 별도로 렌더하지 않는다** — 와이어프레임 HTML 자체에 `<bp-description>`으로 설명이 임베드되어 iframe 안의 우측 레일로 렌더됨 ([features/WIREFRAME.md](../../../features/WIREFRAME.md#embed-기능-설명-임베드)).

- **상단**: [탑바](./area_top-bar.md) — 브레드크럼·버전 드롭다운·발행 버튼·설정
- **좌측**: [파일 트리](./area_file-tree.md) — 240px 고정. 그룹/화면 트리 탐색
- **중앙**: [와이어프레임 뷰어](./area_viewer.md) — flex: 1. iframe + 핀 오버레이 + 화면명세 탭

### 코멘트 시트

뷰어 툴바의 💬 버튼으로 열리는 우측 슬라이드 시트.

- [코멘트 시트](./sheet_comments.md) — 코멘트 목록·라벨 필터·핀 하이라이트·상세·수정·해결·댓글 스레드

## 쿼리 파라미터

| 파라미터 | 기본값 | 동작 |
|----------|--------|------|
| `version` | `latest` (미발행) | 해당 버전의 Git ref로 와이어프레임 로딩 + 코멘트 필터 |
| `comment` | 없음 | 해당 코멘트의 핀 하이라이트 |

## 상태

| 상태 | 표시 |
|------|------|
| 정상 | 2패널 레이아웃 (좌 사이드바 + 중앙 뷰어) |
| 와이어프레임 로딩 중 | 중앙 뷰어 스켈레톤 |
| 와이어프레임 없음 | 빈 상태 안내 — "와이어프레임이 없습니다. 플러그인으로 생성해주세요" + 플러그인 사용법 링크 |
| 권한 없음 (collaborator 아님) | 권한 없음 화면으로 조건부 렌더링 (URL 유지) |
| 저장소 삭제됨 | "저장소를 찾을 수 없습니다" 에러 |
| 화면 ID 유효하지 않음 | 첫 번째 화면으로 리다이렉트 |
| 와이어프레임 업데이트 알림 | 토스트 "업데이트됐습니다 [새로고침]" |
| 과거 버전 열람 | 새 핀 생성 UI 비활성화 |

## 인수조건

- [ ] 좌측 사이드바(화면 목록) + 중앙(뷰어) 2패널이 표시된다
- [ ] 상단에 헤더 바(프로젝트명·버전·설정)가 표시된다
- [ ] 각 패널은 독립적으로 스크롤된다
- [ ] 중앙 뷰어의 iframe 안에는 와이어프레임 UI + 우측 설명 레일이 함께 렌더된다 (HTML 자체에 포함)
- [ ] 좌측 사이드바에서 화면 클릭 시 URL이 `/projects/[id]/screens/[newScreenId]`로 변경되고, 뷰어가 해당 화면으로 갱신된다
- [ ] 화면 전환 시 Realtime 채널이 이전 화면 해제 → 새 화면 구독으로 전환된다
- [ ] 헤더 버전 드롭다운에는 발행된 태그 + "최신(미발행)"이 표시된다
- [ ] 버전 선택 시 URL에 `?version=v1.0`이 추가되고, 해당 버전의 Git ref로 와이어프레임이 로딩된다
- [ ] 해당 버전에 귀속된 코멘트만 표시된다 (+ 이월된 미해결 코멘트)
- [ ] 과거 버전에서는 새 코멘트(핀) 생성이 불가하다
- [ ] `?comment=123`이 있으면 해당 코멘트의 핀이 하이라이트된다
- [ ] 쿼리 파라미터 조합이 가능하다 (`?version=v1.0&comment=123`)
- [ ] Supabase Realtime으로 `comments` INSERT/UPDATE가 실시간 반영된다 (목록 + 핀)
- [ ] 기획자 push 시 Broadcast로 "와이어프레임이 업데이트됐습니다 [새로고침]" 토스트가 표시된다 (자동 새로고침 안 함)
- [ ] 와이어프레임이 없으면 빈 상태 안내가 표시된다
- [ ] `/projects/[id]`로 접근 시 좌측 목록의 첫 번째 화면으로 자동 리다이렉트된다
- [ ] 데스크톱 전용 — 고정 레이아웃

## 비고

- Realtime 채널: `project:{projectId}:screen:{screenId}` — 화면 전환 시 채널 교체
- 헤더의 [←] 클릭 시 `/projects`로 이동
- 우측 기능 패널 폐기 이력: 2026-04-13 — 기능 설명을 와이어프레임 HTML에 `<bp-description>`으로 임베드하는 방식으로 전환. 상세: [features/WIREFRAME.md](../../../features/WIREFRAME.md#embed-기능-설명-임베드)
