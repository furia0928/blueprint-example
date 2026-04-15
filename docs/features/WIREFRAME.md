---
domain: WIREFRAME
title: 기능_와이어프레임
toc:
  - id: REPO_STRUCTURE
    label: 저장소 구조
  - id: RECOGNITION
    label: 화면 인식 규칙
  - id: RENDERING
    label: iframe 렌더링
  - id: BRIDGE
    label: 브릿지 스크립트
  - id: VIEWPORT
    label: 뷰포트 너비
  - id: EMBED
    label: 기능 설명 임베드
  - id: STATE_VARIANT
    label: 상태 변형
  - id: ERROR
    label: 에러 대응
---

# WIREFRAME (와이어프레임)

기획자가 플러그인(`blueprint-plugin`)으로 생성한 HTML 와이어프레임을 플랫폼이 GitHub에서 가져와 iframe으로 렌더링한다. 우측 기능 설명 패널은 플랫폼이 아닌 HTML 자체에 임베드되어 iframe 안에서 함께 렌더된다.

## REPO_STRUCTURE (저장소 구조)

**rules**
- 기능 명세는 `docs/features/*.md`에 도메인 단위로 작성한다
- 화면은 `docs/screens/` 아래 폴더 구조로 작성한다
- 1-depth 폴더명은 그룹명(한글 허용)이다
- 2-depth 폴더명은 `screenId`로 사용되며, 폴더 안의 `.md`·`.html` 파일이 해당 화면의 콘텐츠가 된다
- 그룹 폴더 없이 바로 화면 폴더를 두면 "미분류" 그룹으로 표시된다
- 그룹 폴더 자체가 화면을 겸할 수 있다 (그룹 폴더에도 `.md`·`.html` 파일을 둘 수 있음)
- 저장소는 플러그인이 생성·갱신하는 파생 산출물이며, SSOT는 명세 마크다운이다

## RECOGNITION (화면 인식 규칙)

**rules**
- `docs/screens/` 아래 폴더를 재귀 탐색한다
- 폴더 안에 `.md` 또는 `.html` 파일이 하나라도 있으면 유효한 화면으로 인식한다
- `.md`는 마크다운 렌더러, `.html`은 iframe 와이어프레임 렌더러로 표시한다
- 파일명은 자유이며, 파일 수 제한도 없다
- 같은 폴더가 화면과 그룹 역할을 동시에 할 수 있다 (`.md`·`.html` 보유 + 하위 화면 폴더 보유)
- screen_id는 그룹 없으면 폴더명(`STANDALONE`), 그룹 아래면 `그룹/폴더명`(`상품/PRODUCT-LIST`), 그룹 자체가 화면이면 그룹명(`상품`)이다

## RENDERING (iframe 렌더링)

**rules**
- Server Action에서 GitHub Contents API로 HTML을 fetch한 뒤 브릿지 스크립트를 주입하고 `<iframe srcdoc="...">`로 렌더링한다
- 1MB 초과 파일은 Contents API의 `raw` 미디어 타입(`Accept: application/vnd.github.raw`)으로 요청하며 최대 100MB까지 지원한다
- 샌드박스는 `sandbox="allow-scripts"`만 사용한다
- `allow-same-origin`은 사용하지 않는다 — `allow-scripts`와 함께 쓰면 iframe 내 스크립트가 parent 세션/토큰에 접근 가능해 sandbox가 무력화된다
- 공통 리소스(`base.css`, `bp-components.js`)는 버전별 경로(`/blueprint/v1/...`)로 관리한다
- 플러그인은 HTML 생성 시점의 리소스 버전 경로를 HTML에 삽입한다
- 과거 와이어프레임은 생성 시점 버전의 리소스를 계속 참조하므로 하위 호환을 보장한다
- iframe은 고유 origin이 부여되므로 parent 접근은 불가하고, 외부 CDN 리소스는 정상 로딩된다

## BRIDGE (브릿지 스크립트)

**rules**
- 브릿지 스크립트는 srcdoc 조립 시 `</body>` 앞에 주입된다
- iframe → parent 단방향 `postMessage` 통신을 사용한다 (parent에서 iframe 내부 DOM 접근은 sandbox로 차단됨)
- `{ type: 'bp:resize', height }` 메시지로 `document.body.scrollHeight`를 전달하여 iframe 높이를 동기화한다
- `{ type: 'bp:features', features: [{ id, rect }] }` 메시지로 기능 div들의 위치·크기를 전달하여 오버레이 하이라이트 영역 계산에 사용한다
- DOMContentLoaded·resize·scroll·이미지/폰트 로딩 완료 시 rect를 재계산한다
- parent는 `window.addEventListener('message', ...)` + origin 검증(`event.origin === 'null'`)으로만 메시지를 수신한다 (sandbox iframe은 null origin)

## VIEWPORT (뷰포트 너비)

**rules**
- 뷰포트 너비는 파일명으로 자동 결정된다 (프로젝트/화면 단위 설정 없음)
- 파일명이 `*-mobile.html` 또는 `*_mobile.html`이면 뷰어 너비는 370px 고정·중앙 정렬이다
- 그 외(`*wireframe.html`, `*-pc.html` 등)는 100% 너비·최소 1200px이다
- PC/MO 분리 화면은 파일 두 개(`*-pc.html`, `*-mobile.html`)가 파일 트리에 동시 표시되며 클릭으로 전환한다
- 뷰포트 전환 토글은 제공하지 않는다 — 파일 선택이 곧 뷰포트 선택이다

## EMBED (기능 설명 임베드)

**rules**
- 플랫폼 우측 기능 패널은 2026-04-13 자로 폐기되었다
- 기능 설명은 와이어프레임 HTML 자체에 `bp-*` 컴포넌트로 임베드되며, 플랫폼은 별도 패널 없이 iframe에 그대로 렌더한다
- 화면 전체 목적·비즈니스 규칙은 `<bp-screen-description>`(body 자식, `bp-page` 형제)에 작성한다
- 화면 목적은 `<bp-description-purpose>`, 화면 간 비즈니스 규칙은 `<bp-description-rules>`(`<li>` 목록)에 작성한다
- 섹션(기능 영역)의 설명은 `<bp-section>` 자식으로 `<bp-description>`을 두고, 개별 요소는 `<bp-description-element name="...">`로 감싼다
- 런타임에 `right-rail.ts`가 모든 `bp-description` / `bp-screen-description`을 body 직속 `<aside class="bp-right-rail">`로 이동시켜 우측 320px sticky 레일로 렌더한다
- body는 grid 2-컬럼으로 전환된다 (좌: `bp-page`, 우: 레일)
- 레일의 `bp-description-element[name="X"]`와 메인 UI의 `[data-element="X"]`는 mouseenter/leave로 양방향 하이라이트(outline)된다
- HTML의 데이터 소스는 SSOT인 `*screen*.md` + `docs/features/*.md`이며, 플러그인이 이를 읽어 HTML을 생성한다
- 플랫폼은 사이드바 파일 열람 용도 외에는 `*screen*.md` / `docs/features/*.md`를 직접 로딩하지 않는다

## STATE_VARIANT (상태 변형)

**rules**
- 하나의 화면은 여러 상태(정상·빈 목록·로딩·에러 등)를 가질 수 있다
- 와이어프레임 HTML 내에서 `data-show="상태명"` / `data-hide="상태명"` 속성으로 상태별 콘텐츠를 인라인 정의한다
- 속성이 없는 요소는 항상 표시된다
- 쉼표(`,`)로 복수 상태를 지정할 수 있다 (`data-show="에러, 네트워크 에러"`)
- 기본 상태(`기본`)는 암묵적이며, `data-show`·`data-hide`가 없는 요소만 표시되는 초기 상태이다
- 상태 이름에 `/`를 사용하여 계층을 표현한다 (`답변 작성 중/입력 검증 에러`)
- 하위 상태 선택 시 모든 조상 상태의 콘텐츠가 함께 표시된다
- `data-hide`에 등장하는 최상위 경로는 기본과 같은 레벨의 대체 상태로 간주한다 (기본 콘텐츠를 숨기고 대체)
- `data-show`에만 등장하는 상태는 기본의 하위 상태로 간주한다 (기본 콘텐츠에 추가 표시)
- 상태 변형이 있는 `bp-section`만 토글 그룹이 자동 생성된다
- 토글 UI는 트리 구조로 표시되며 노드 클릭으로 상태를 전환한다
- 대체 상태 선택 시 기본 하위 트리의 버튼은 비활성화된다
- 토글 UI는 와이어프레임 콘텐츠 바깥(페이지 상단)에 배치되어 콘텐츠를 오염시키지 않는다
- `state-variants.ts`가 각 `bp-section`의 상태 변형을 스캔해 레일의 해당 `bp-description` 상단에 토글 트리를 주입하며, 탭 클릭 시 메인 UI와 description 요소가 동시에 상태 전환된다

## ERROR (에러 대응)

**rules**
- `docs/screens/` 디렉토리가 없거나 유효한 화면이 하나도 없으면 파일 트리에 빈 목록과 함께 "와이어프레임이 없습니다. 플러그인으로 생성해주세요" 안내가 표시된다
- 화면 인식 규칙에 맞지 않는 폴더는 에러 없이 목록에서 제외된다
- HTML 로딩 실패 시 뷰어에 "와이어프레임을 불러올 수 없습니다" 메시지와 재시도 버튼을 표시한다
- 화면 폴더명이 변경되면(`LOGIN/` → `SIGN-IN/`) 새 화면으로 인식되며, 기존 폴더에 달린 코멘트는 연결이 끊긴다
- 연결 끊긴 코멘트는 프로젝트 설정에서 "연결 끊긴 코멘트 N개" 로 표시되며, 자동 추적하지 않는다
- 폴더명 변경 금지 가이드라인은 플러그인 측에서 제공한다
