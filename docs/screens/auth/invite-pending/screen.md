---
screenId: INVITE-PENDING
title: 화면_승인 대기
type: page
purpose: 승인 대기 중인 게스트에게 본인 정보와 대기 안내를 표시한다
viewport: [pc]
features: [AUTH]
group: auth
---

# 승인 대기

> 라우트: `/invite/pending`
> 인증: 필수 (비로그인 시 `/login`으로 리다이렉트)
> 진입: 미들웨어 rewrite 대상 — pending viewer가 보호 경로 접근 시
> 상세: [features/AUTH.md#GUEST_APPROVAL](../../../features/AUTH.md#guest_approval-오너-승인-관리), [ROUTING.md](../../../ROUTING.md)

## 화면 구성

중앙 카드 1개. 시계 아이콘 + 상태 안내 + 본인 정보.

### [승인 대기 안내](../../../features/AUTH.md#guest_approval-오너-승인-관리)

**elements**
- 아이콘: 시계 (lucide `Clock`)
- 제목: "승인 대기 중입니다"
- 안내 문구: "프로젝트 오너가 접근을 승인하면 이용할 수 있습니다"
- AUTH 본인 정보 — 하단 박스
  - 텍스트 항목: 이름 (`display_name`) — 강조 표시. 값 없으면 줄 생략
  - 텍스트 항목: 이메일 — 보조 텍스트 색
- 보조 안내: "승인 후 다시 접속해주세요"

## 상태

| 상태 | 표시 |
|------|------|
| 기본 | 시계 아이콘 + 안내 + 본인 정보 박스 |
| `display_name` 없음 | 이름 줄 생략, 이메일만 표시 |
| 이름·이메일 모두 없음 | 본인 정보 박스 생략 (안내 문구만 표시) |
| 승인됨으로 전환 | 다음 보호 경로 진입 시 rewrite되지 않고 원래 경로 정상 렌더링 |

## 인수조건

- [ ] pending 상태의 viewer가 `/projects` 또는 `/onboarding` 접근 시 이 화면으로 rewrite된다 (URL은 유지됨)
- [ ] `display_name`이 있으면 강조 표시로 노출한다
- [ ] 이메일을 보조 텍스트 색으로 노출한다
- [ ] 이름·이메일이 모두 없으면 본인 정보 박스를 생략한다
- [ ] 조작 가능한 액션이 없는 읽기 전용 안내 화면이다
- [ ] `approved`로 전환된 뒤에는 더 이상 이 화면으로 rewrite되지 않는다

## 진입 경로

| 출발 | 트리거 |
|------|------|
| viewer(pending)가 `/projects` 접근 | 미들웨어 rewrite |
| viewer(pending)가 `/onboarding` 접근 | 미들웨어 rewrite |
| `/invite/pending` 직접 URL 입력 | 페이지 직접 렌더링 |

## 비고

- **[게스트 초대](../invite/screen.md) 화면의 승인 대기 상태와 다른 페이지**다. 초대 링크로 방금 등록한 게스트는 `/invite/[token]`에서 프로젝트명이 함께 표시된 안내를 본다. 이 화면(`/invite/pending`)은 재방문 시 rewrite 대상이며 프로젝트 컨텍스트가 없어 이름·이메일만 표시한다.
- rewrite 로직: [flows/AUTH.md#미들웨어-접근-제어-viewer](../../../flows/AUTH.md#미들웨어-접근-제어-viewer)
- 승인/거절은 오너 측 [프로젝트 설정 시트](../../project/detail/sheet_settings.md)에서 수행
