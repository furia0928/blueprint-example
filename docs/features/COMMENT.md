---
domain: COMMENT
title: 기능_코멘트
toc:
  - id: CREATE
    label: 코멘트 생성
  - id: EDIT
    label: 코멘트 수정
  - id: RESOLVE
    label: 코멘트 해결
  - id: THREAD
    label: 댓글 스레드
  - id: PIN
    label: 핀 좌표 관리
  - id: LABEL
    label: 라벨·담당자
  - id: SYNC
    label: 실시간·웹훅 동기화
  - id: DEEP_LINK
    label: 딥링크 하이라이트
---

# COMMENT (코멘트)

와이어프레임 위에서 남기는 피드백. GitHub Issue로 저장되고 Supabase `comments` 테이블이 좌표·버전·화면 귀속 등 메타데이터를 보관한다. 게스트 코멘트 대리 작성 규칙은 [AUTH.md#GUEST_COMMENT](./AUTH.md#guest_comment-게스트-코멘트-대리-작성)에서 정의한다.

## CREATE (코멘트 생성)

**rules**
- 코멘트는 제목(필수), 본문(선택), 라벨, 담당자, 화면(screenId), 버전(version), 좌표(x, y), featureId, 작성자를 가진다
- 제목은 비어 있을 수 없다
- 생성 시 GitHub Issue를 먼저 생성하고, 본문에 `<!-- blueprint-meta: {"page","x","y","version"} -->` 주석을 포함한다
- Issue 생성 성공 후 반환된 `issue_number`를 기준으로 `comments` 테이블에 row를 저장한다
- Supabase 저장이 실패하면 최대 3회 재시도하고, 최종 실패 시 "GitHub에는 생성됐지만 플랫폼에 반영되지 않았습니다" 안내를 표시한다
- 누락된 코멘트는 GitHub Webhook `issues` 이벤트로 자동 보정된다
- 과거 버전(발행된 태그) 열람 중에는 새 코멘트 생성이 불가하다
- 기능 영역(feature div)이 없는 빈 영역에는 코멘트를 생성할 수 없다
- 게스트 코멘트는 GitHub App(봇)이 Installation 토큰으로 Issue를 대리 생성한다 ([AUTH.md#GUEST_COMMENT](./AUTH.md#guest_comment-게스트-코멘트-대리-작성))

## EDIT (코멘트 수정)

**rules**
- 본인이 작성한 코멘트의 제목·본문·라벨·담당자만 수정할 수 있다
- 타인 코멘트에는 수정 UI가 노출되지 않는다
- 수정 시 GitHub Issue PATCH + `comments` 테이블 UPDATE를 함께 수행한다
- 핀 위치(featureId, 좌표)는 수정할 수 없다. 위치를 바꾸려면 삭제 후 재작성한다
- 과거 버전의 코멘트라도 본인이 작성했다면 수정할 수 있다
- GitHub에서 직접 Issue를 수정한 경우 Webhook으로 플랫폼에 동기화된다

## RESOLVE (코멘트 해결)

**rules**
- 해결은 GitHub Issue를 close하고 `comments` 테이블에서 row를 삭제하는 동작이다
- GitHub에는 closed Issue로 남아 이력이 유지된다
- 확인 절차(확인 다이얼로그)를 통과해야 실행된다
- 해결된 코멘트는 목록·뷰어 핀에서 즉시 제거된다
- GitHub에서 Issue를 close/reopen한 경우도 Webhook으로 동기화된다
- 과거 버전에서도 해결은 가능하다

## THREAD (댓글 스레드)

**rules**
- 댓글은 GitHub Issue comments를 그대로 사용한다 (플랫폼 별도 저장 없음)
- 댓글 전송은 사용자 OAuth 토큰으로 GitHub API를 호출한다
- 게스트 댓글은 GitHub App(봇)이 Installation 토큰으로 대리 작성한다
- `@멘션` 입력 시 저장소 collaborator 목록에서 자동완성된다
- `@멘션` 알림은 GitHub 자체 알림에 위임하며 플랫폼이 별도 알림을 보내지 않는다
- 과거 버전의 코멘트에도 댓글을 추가할 수 있다
- 시간순(오름차순)으로 표시된다

## PIN (핀 좌표 관리)

**rules**
- 좌표는 feature div 기준 상대 비율(0~1)로 저장한다
- feature div가 이동하면 핀이 함께 이동한다
- feature div 크기가 줄어 좌표가 범위 밖이면 최대값으로 클램핑한다
- 브릿지 스크립트의 `bp:features` 메시지로 feature rect를 수신해 뷰어 좌표로 변환한다
- 이미지·폰트 로딩 완료 시 rect를 재계산해 핀 위치를 갱신한다
- featureId가 현재 와이어프레임에 없는 이월 코멘트는 핀을 표시하지 않으며, 목록에서는 "삭제된 기능" 뱃지로 표시된다
- 핀 레이어는 기본 `pointer-events: none`이며, 코멘트 시트가 열려 기능 영역 하이라이트가 활성화된 경우에만 클릭 가능해진다

## LABEL (라벨·담당자)

**rules**
- 라벨은 저장소의 GitHub Labels를 그대로 사용한다 (플랫폼 별도 관리 없음)
- 담당자(Assignee)는 저장소 collaborator에서 다중 선택한다
- 라벨·담당자는 작성·수정 모두에서 변경할 수 있다
- 코멘트 시트의 라벨 필터는 다중 선택이며, 선택 시 코멘트 목록과 뷰어 핀이 함께 필터링된다
- 필터 해제 시 전체 코멘트를 표시한다
- 게스트 코멘트 라벨 자동 부착 규칙은 [AUTH.md#GUEST_COMMENT](./AUTH.md#guest_comment-게스트-코멘트-대리-작성) 참조

## SYNC (실시간·웹훅 동기화)

**rules**
- `comments` 테이블 INSERT/UPDATE는 Supabase Realtime Postgres Changes로 구독되어 코멘트 목록과 뷰어 핀에 즉시 반영된다
- Realtime 채널 이름은 `project:{projectId}:screen:{screenId}`이며, 화면 단위로 구독한다
- Postgres Changes 필터는 `project_id` + `screen_id`를 사용해 현재 화면의 코멘트만 수신한다
- 화면 전환 시 이전 채널을 해제하고 새 채널을 구독한다
- 와이어프레임 업데이트 알림은 Broadcast 이벤트(`wireframe:updated`)를 사용하며, Webhook API가 GitHub push를 수신했을 때 해당 채널로 전송한다
- 업데이트 Broadcast 수신 시 토스트로 안내하되 자동 새로고침은 하지 않는다 (사용자가 직접 새로고침 결정)
- GitHub Webhook `issues` 이벤트로 수정/close/reopen을 플랫폼에 동기화한다 (`comments.status` 업데이트, 누락 row 보정)
- 코멘트 목록은 페이지네이션 없이 현재 화면+버전의 전체를 한 번에 조회한다 (버전별 코멘트 수가 적다는 가정)
- 상세 본문·댓글 스레드는 카드 클릭 시 GitHub Issues API로 on-demand fetch한다

## DEEP_LINK (딥링크 하이라이트)

**rules**
- URL에 `?comment={issueNumber}`가 있으면 해당 코멘트의 핀이 뷰어에서 강조되고 코멘트 시트가 자동으로 상세 뷰로 열린다
- `?version`, `?viewport` 파라미터와 조합 가능하다
- 해당 코멘트가 현재 버전·뷰포트에 존재하지 않으면 일반 목록 뷰만 표시한다
