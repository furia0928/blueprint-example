---
title: 영역_뷰어
type: panel
features:
  - id: WIREFRAME__RENDERING
    ref: ../../../features/WIREFRAME.md#rendering-iframe-렌더링
---

# 뷰어

> 소속: [screen.md](./screen.md)

중앙 `flex: 1` 영역. 파일 트리에서 선택한 파일의 확장자에 따라 HTML 뷰어 또는 MD 뷰어로 전환된다.

## 유저스토리

- 사용자로서, 기획자가 만든 와이어프레임을 원본 그대로 보고 싶다.
- 사용자로서, 와이어프레임 위에 기존 코멘트의 핀을 보고 싶다.
- 사용자로서, 코멘트를 작성하려면 어느 영역을 클릭해야 하는지 시각적으로 구분되어야 한다.
- 사용자로서, 마크다운 문서(화면명세/기능명세)를 같은 뷰어에서 열람하고 싶다.

## 화면 구성

### HTML 뷰어 (`*.html` 선택 시)

**elements**
- iframe 컨테이너: `<iframe sandbox="allow-scripts" srcdoc="...">` — `allow-same-origin` 사용 금지
- 브릿지 스크립트: 빌드 시 `</body>` 앞에 주입되어 `bp:resize` / `bp:features` postMessage 전송
- 핀 오버레이 레이어 (iframe 위 absolute positioned div)
  - 핀 뱃지: 코멘트 제목 요약, 클릭 가능
  - 기본 `pointer-events: none` (iframe 내부 인터랙션 허용)
- 기능 영역 하이라이트: 코멘트 시트 열림 시 반투명 오버레이. 하이라이트된 영역만 `pointer-events` 활성
- 💬 코멘트 버튼: 클릭 시 [코멘트 시트](./sheet_comments.md) 토글. 시각적으로 탑바 우측 버전 드롭다운 앞에 배치

> 💬 버튼은 **DOM은 이 섹션(WIREFRAME__RENDERING) 안**에 두고 `position: fixed`로 탑바 좌표에 투영한다. 상태 종속성은 뷰어, 시각 위치는 탑바가 되어야 하는데 `state-variants.ts`가 섹션별 독립 상태 트리라 DOM 중첩이 상태 경계를 결정하기 때문. 실 플랫폼(`viewer-toolbar.tsx`)은 `{isHtml && …}` 런타임 조건부로 동일 위치에 렌더한다.

### MD 뷰어 (`*.md` 선택 시)

**elements**
- 렌더링 컨테이너: react-markdown + remark-gfm
- 테이블·코드 블록 등 GFM 문법 지원
- 핀 오버레이 없음 (코멘트는 `.html`에만 가능 — 💬 버튼도 숨김)

## 상태

| 상태 | 비고 | 표현 |
|------|------|------|
| 기본 (파일 선택 안내) | 좌측 트리 미선택 — 실 서비스에선 자동 리다이렉트로 거의 노출 안됨 | "파일을 선택하세요" placeholder |
| HTML 뷰어 (alt) | `*.html` 선택 시 | iframe + 핀 오버레이 |
| MD 뷰어 (alt) | `*.md` 선택 시 | react-markdown 렌더링. 핀 없음 |
| 로딩 중 (alt) | 파일 fetch 중 | 뷰어 영역 스켈레톤 |
| 로딩 실패 (alt) | 네트워크/권한/삭제 등 | "와이어프레임을 불러올 수 없습니다" + 재시도 |
| 빈 프로젝트 (alt) | `docs/screens/` 빈 상태 | "와이어프레임이 없습니다" + 플러그인 사용법 |
| HTML — feature div 없음 | 서브 상태 | 핀 없이 와이어프레임만. 시트에서 "클릭 가능한 영역이 없습니다" |
| 과거 버전 (HTML) | `?version=v1.2` 등 | 기능 영역 하이라이트/클릭 비활성 |

## 인수조건

### HTML 뷰어

- [ ] GitHub Contents API로 HTML을 fetch하여 `<iframe srcdoc>`으로 렌더링한다
- [ ] `sandbox="allow-scripts"`만 사용하고 `allow-same-origin`은 사용하지 않는다
- [ ] 브릿지 스크립트가 `</body>` 앞에 주입된다
- [ ] iframe 높이가 `bp:resize` postMessage로 자동 동기화된다
- [ ] 공통 리소스(`base.css`, `bp-components.js`)가 정상 로딩된다
- [ ] 1MB 초과 파일은 Contents API `raw` 미디어 타입으로 요청된다
- [ ] iframe 외부에 absolute positioned 레이어로 핀이 렌더링된다
- [ ] 핀 위치는 feature div 기준 비율 좌표(0~1)로 계산된다
- [ ] `bp:features` 메시지로 받은 feature rect를 뷰어 좌표로 변환한다
- [ ] 핀에 코멘트 제목 뱃지가 표시된다
- [ ] 핀 클릭 시 코멘트 시트에서 해당 코멘트가 하이라이트된다
- [ ] `?comment=123` 딥링크 시 해당 핀이 강조된다
- [ ] 코멘트 시트가 열리면 기능 영역이 하이라이트 표시되고, 하이라이트 영역만 클릭 가능하다
- [ ] 기능이 없는 빈 영역에는 코멘트를 생성할 수 없다
- [ ] 기능 영역 클릭 시 클릭 좌표로 [코멘트 작성 다이얼로그](./dialog_comment.md)가 열린다
- [ ] 과거 버전에서는 하이라이트/클릭이 비활성화된다
- [ ] 이미지·폰트 로딩 완료 시 브릿지가 rect를 재계산하여 오버레이가 갱신된다

### MD 뷰어

- [ ] `.md` 파일을 react-markdown + remark-gfm으로 렌더링한다
- [ ] 테이블·코드 블록 등 GFM 문법이 지원된다
- [ ] 핀 오버레이가 표시되지 않는다
- [ ] 파일이 없거나 로딩 실패 시 안내 메시지가 표시된다

### 공통

- [ ] 파일 트리에서 `.html` 선택 → HTML 뷰어, `.md` 선택 → MD 뷰어로 전환된다
- [ ] 화면 전환 시 이전 iframe/렌더러가 해제되고 새 파일이 fetch된다

## 비고

- origin 검증: `event.origin === 'null'` (sandbox iframe은 null origin)
- feature div 크기가 줄어서 핀 좌표가 범위 밖이면 최대값으로 클램핑
- featureId가 현재 와이어프레임에 없는 이월 코멘트: 핀 미표시 (코멘트 시트에서만 "삭제된 기능" 표시)
- MD 뷰어는 현재 화면명세(`screen.md`, `area_*.md`)·기능명세(`features/*.md`)·플로우(`flows/*.md`) 모두 공통 렌더러 사용
