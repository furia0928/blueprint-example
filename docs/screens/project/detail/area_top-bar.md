---
title: 영역_탑바
type: panel
features:
  - id: VERSION__TAG
    ref: ../../../features/VERSION.md#tag-버전-정의
  - id: VERSION__PUBLISH
    ref: ../../../features/VERSION.md#publish-버전-발행
---

# 탑바

> 소속: [screen.md](./screen.md)

## 유저스토리

- 사용자로서, 지금 보고 있는 파일이 어디에 있는지 알고 싶다.
- 사용자로서, 버전을 전환하여 과거/현재 와이어프레임을 비교하고 싶다.
- 사용자로서, 현재 상태를 버전으로 발행할 수 있어야 한다.
- 사용자로서, 사이드바를 토글하여 뷰어 공간을 넓히고 싶다.

**elements**
- 좌측
  - `[☰]` 사이드바 토글 버튼 — 좌측 사이드바 collapse/expand
  - 브레드크럼 — 현재 파일 경로를 세그먼트별 표시 (`상품 / PRODUCT-LIST / wireframe.html`)
- 우측
  - 버전 드롭다운 (shadcn Select)
    - "최신 (미발행)"이 분리선 위에 단독 배치
    - 발행된 버전은 "발행된 버전" 라벨 아래 그룹으로 표시
    - 현재 선택된 버전 하이라이트
  - 버전 발행 버튼 (TagIcon + "발행") — 드롭다운 옆에 배치
  - 설정 아이콘 (⚙) — 프로젝트 설정 시트 열기

> **참고:**
> - `[←]` 뒤로가기 + 프로젝트명은 좌측 사이드바 헤더에 배치 ([area_file-tree](./area_file-tree.md) 참조)
> - 💬 코멘트 버튼은 버전 드롭다운 앞에 배치되며, 가시성은 [area_viewer](./area_viewer.md)의 `WIREFRAME__RENDERING` 상태(`HTML 뷰어`)에 위임된다

## 상태

| 요소 | 상태 | 표현 |
|------|------|------|
| 발행 버튼 | "최신(미발행)" 선택 + HEAD SHA ≠ last_published_sha | enabled (primary) |
| 발행 버튼 | 과거 버전 열람 중 | disabled (muted) |
| 발행 버튼 | HEAD SHA = last_published_sha | disabled (muted) |

## 인수조건

- [ ] `[☰]` 사이드바 토글 버튼이 헤더 좌측에 표시되고, 클릭 시 좌측 사이드바가 collapse/expand 된다
- [ ] 브레드크럼에 현재 파일 경로가 세그먼트별로 표시된다
- [ ] 버전 드롭다운은 "최신(미발행)" + "발행된 버전" 그룹으로 구성된다
- [ ] 현재 선택된 버전이 하이라이트된다
- [ ] 버전 선택 시 URL의 `?version` 파라미터가 변경되고, 뷰어·코멘트가 해당 버전으로 갱신된다
- [ ] 버전 목록은 실제 Git Tags API로 조회한다 (캐싱 적용)
- [ ] 발행 버튼은 미발행 변경이 있을 때(`HEAD SHA ≠ last_published_sha`)만 활성화된다
- [ ] 과거 버전 열람 중에는 발행 버튼이 비활성화된다
- [ ] 발행 버튼 클릭 시 [버전 발행 다이얼로그](./dialog_version-publish.md)가 열린다
- [ ] 설정 아이콘(⚙) 클릭 시 [프로젝트 설정 시트](./sheet_settings.md)가 열린다

## 비고

- 버전 목록은 GitHub Tags API로 조회 (캐싱 적용)
- 발행 버튼은 별도 뱃지 없이 활성화 상태로만 변경 신호를 전달. 상세 규칙은 [features/VERSION.md](../../../features/VERSION.md)
