---
screenId: FORBIDDEN
title: 화면_권한 없음
type: page
purpose: collaborator가 아닌 사용자가 프로젝트 URL에 접근했을 때 거부 사유와 복귀 경로를 제공한다
viewport: [pc]
features: [AUTH]
group: auth
---

# 권한 없음

> 라우트: 없음 (원 페이지에서 조건부 렌더링)
> 트리거: collaborator가 아닌 로그인 사용자가 프로젝트 URL 접근 시 서버 컴포넌트에서 권한 체크 실패 → `<Forbidden />` 컴포넌트 반환
> URL은 원래 경로를 유지한다 (rewrite 아님)

## 화면 구성

중앙 카드 1개. 거부 사유 안내 + 복귀 액션.

### 접근 거부 안내

**elements**
- 헤더: "접근 권한이 없습니다"
- 안내 문구: "이 프로젝트의 collaborator가 아닙니다. 저장소 Admin에게 권한을 요청하세요."
- 액션 항목: "프로젝트 목록으로" 버튼 — `/projects`로 이동

## 상태

| 상태 | 표현 |
|------|------|
| 기본 | 안내 + 복귀 버튼 |

## 인수조건

- [ ] collaborator가 아닌 로그인 사용자가 `/projects/[id]/...` 접근 시 이 화면이 렌더링된다 (URL 유지)
- [ ] "프로젝트 목록으로" 클릭 시 `/projects`로 이동한다
- [ ] viewer 미승인(`pending`) 사용자는 이 화면이 아닌 [승인 대기](../invite-pending/screen.md)로 미들웨어가 rewrite한다 (분리된 처리)

## 비고

- 권한 체크는 GitHub collaborator 상태 기준 (자세한 권한 정책은 [features/PROJECT.md#ACCESS](../../../features/PROJECT.md#access-접근-권한))
- 별도 라우트가 없는 조건부 렌더링 전용 화면. 각 보호 페이지의 서버 컴포넌트에서 collaborator 검증 후 `<Forbidden />` 컴포넌트를 반환한다
- `/invite/pending`(미들웨어 rewrite)와 구현 방식이 다르다 — 이 화면은 원 URL을 유지하며 서버 컴포넌트에서 컴포넌트만 교체
