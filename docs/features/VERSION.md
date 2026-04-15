---
domain: VERSION
title: 기능_버전
toc:
  - id: TAG
    label: 버전 정의
  - id: PUBLISH
    label: 버전 발행
  - id: FILE_SCOPE
    label: 파일 타입별 버전 적용
  - id: PAST_ACCESS
    label: 과거 버전 열람 규칙
  - id: INHERIT
    label: 이월된 미해결 코멘트
  - id: UNPUBLISHED
    label: 미발행 변경 감지
  - id: COORD_TRUST
    label: 코멘트 좌표 신뢰성
---

# VERSION (버전)

Git 태그 기반의 와이어프레임·화면명세 버전 체계. 기획자가 피드백 수렴 시점에 수동으로 발행하며, 코멘트는 발행 시점에 해당 버전으로 귀속된다.

## TAG (버전 정의)

**rules**
- 버전은 Git 태그와 1:1로 대응된다
- 메이저(`v1.x → v2.0`)는 화면 구조 변경 등 큰 수정에 사용한다
- 마이너(`v1.x → v1.y`)는 피드백 반영·소소한 수정에 사용한다
- "최신(미발행)"은 기본 브랜치(main)의 HEAD를 가리키는 가상 버전이다
- 브랜치 전환은 지원하지 않는다 (기본 브랜치 HEAD만 사용)
- 버전 드롭다운에는 "최신(미발행)" + 발행된 태그 목록이 표시된다
- 첫 발행은 `v1.0`으로 고정되며 유형 선택이 불필요하다

## PUBLISH (버전 발행)

**rules**
- 발행은 GitHub API로 기본 브랜치 HEAD 커밋에 태그를 생성하는 동작이다
- 발행은 App 토큰으로 수행한다 (시스템 행동)
- 발행 유형(메이저/마이너)을 기획자가 선택한다
- 미해결 코멘트(`status = open`)가 있어도 발행은 차단하지 않으며 경고만 표시한다
- 해결된 코멘트(`status = closed`)는 발행 시 해당 버전에 귀속된다 (`comments.version`을 태그로 UPDATE)
- 미해결 코멘트는 `version = 'latest'`에 잔류하여 다음 발행까지 이월된다
- 동시 발행 경합은 GitHub API가 중복 태그를 거부하므로 별도 락이 불필요하다
- 발행 직후 태그 목록 캐시를 무효화하여 드롭다운에 즉시 반영한다
- 발행 시 `projects.last_published_sha`와 `last_published_at`을 갱신한다

## FILE_SCOPE (파일 타입별 버전 적용)

**rules**
- 태그 시점의 저장소 스냅샷을 기준으로 모든 파일이 함께 고정된다
- 와이어프레임(`.html`)은 해당 태그 시점의 HTML을 iframe에 렌더링한다
- 화면명세(`.md`)는 해당 태그 시점의 MD를 마크다운으로 렌더링한다
- 버전 드롭다운은 파일 타입에 관계없이 공유되며, 현재 열람 중인 파일 타입에 맞는 콘텐츠를 로딩한다
- `[발행]` 버튼은 "최신(미발행)" 선택 시에만 활성화 후보가 된다 (활성화 조건은 UNPUBLISHED 참조)

## PAST_ACCESS (과거 버전 열람 규칙)

**rules**
- 과거 버전 열람 중에는 새 코멘트(핀) 생성이 불가하다
- 과거 버전에서도 댓글 추가(GitHub Issue comment)는 가능하다
- 과거 버전에서도 해결 처리(Issue close)는 가능하다
- 과거 버전 열람 중에는 `[발행]` 버튼이 비활성화된다
- 뷰어의 기능 영역 하이라이트·클릭도 과거 버전에서는 비활성화된다

## INHERIT (이월된 미해결 코멘트)

**rules**
- 이월 코멘트는 발행되지 않은 `version = 'latest'` 상태로 잔류한 미해결 코멘트를 의미한다
- 이월 코멘트의 featureId가 현재 와이어프레임에 존재하면 핀을 표시한다
- 이월 코멘트의 featureId가 현재 와이어프레임에 없으면 핀은 표시하지 않고, 목록에서 "삭제된 기능" 뱃지로 표시한다
- 좌표는 feature div 기준 상대 비율(0~1)이므로 div 이동에 영향받지 않는다
- feature div 크기가 줄어 좌표가 범위 밖이면 최대값으로 클램핑한다 (`min(좌표, div크기)`)

## UNPUBLISHED (미발행 변경 감지)

**rules**
- 기획자가 Git push 후 발행을 잊는 상황을 방지하는 안전장치이다
- 자동 발행은 하지 않는다 — 유형 선택과 코멘트 정리 타이밍은 기획자가 결정해야 한다
- GitHub Webhook `push` 이벤트 수신 시 HEAD SHA와 `projects.last_published_sha`를 비교한다
- HEAD SHA ≠ `last_published_sha`이면 미발행 상태로 간주한다
- `last_published_sha = NULL`(한 번도 발행하지 않은 프로젝트)인 경우도 미발행 상태로 간주한다
- 미발행 상태에서만 `[발행]` 버튼이 활성화된다 (별도 뱃지 없이 버튼 상태로만 표현)
- 발행 시 `last_published_sha`를 태그 대상 커밋 SHA로 갱신한다

## COORD_TRUST (코멘트 좌표 신뢰성)

**rules**
- 각 코멘트는 생성 시점의 HEAD 커밋 SHA를 `comments.commit_sha`에 기록한다
- 현재 HEAD SHA ≠ 코멘트의 `commit_sha`이면 핀에 ⚠ 표시와 함께 "이전 와이어프레임 기준 코멘트" 안내를 노출한다
- 와이어프레임이 변경된 뒤 발행 전에 달린 코멘트의 좌표 불일치를 팀원에게 알리는 목적이다
- 발행 후에는 해당 버전의 `commit_sha`와 태그 커밋 SHA가 일치하므로 경고가 해제된다
