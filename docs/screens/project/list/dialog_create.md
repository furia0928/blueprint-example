---
screenId: PROJECT-CREATE
title: 다이얼로그_프로젝트 생성
type: dialog
purpose: GitHub 저장소를 선택해 새 프로젝트를 생성한다. 미설치 저장소는 GitHub App 설치를 유도한다
viewport: [pc]
features:
  - PROJECT
  - AUTH
group: project
---

# 프로젝트 생성

> 트리거: [프로젝트 목록](./screen.md)에서 "새 프로젝트" 버튼

## 화면 구성

2스텝 다이얼로그. Step 1에서 GitHub App 설치 여부에 따라 **저장소 선택** 또는 **App 설치 유도**로 분기. Step 2에서 프로젝트 이름 확정.

### [Step 1 — 저장소 선택](../../../features/PROJECT.md#create-프로젝트-생성)

GitHub App이 설치된 경우의 영역.

**elements**
- 다이얼로그 헤더: "새 프로젝트" + 닫기 버튼
- 섹션 제목: "저장소 선택"
- 텍스트 항목: 저장소 검색 입력 — 목록 실시간 필터
- PROJECT__ACCESS collaborator 저장소를 라디오 리스트로 표시
  - 1개만 선택 가능
  - 이미 다른 프로젝트에 연결된 저장소는 비활성 + "이미 연결됨" 배지
- 액션 항목: "다음" 버튼 — Step 2로 이동 (저장소 미선택 시 비활성)

### [Step 1 — GitHub App 설치 유도](../../../features/AUTH.md#github_app_install-github-app-설치)

GitHub App이 설치되지 않은 경우의 영역.

**elements**
- 안내 문구: "Blueprint App을 GitHub에 설치해야 저장소에 접근할 수 있습니다"
- 액션 항목: "GitHub App 설치하기" 버튼 — GitHub App 설치 페이지로 이동 (새 탭)
- 보조 안내: "설치 완료 후 이 화면으로 돌아옵니다"

### [Step 2 — 프로젝트 이름](../../../features/PROJECT.md#info-프로젝트-기본-정보)

저장소 선택 후 이름 확정.

**elements**
- 상단 정보: 선택된 `owner/repo` 표시 (읽기 전용)
- 텍스트 항목: 프로젝트 이름 입력 — 기본값은 저장소명(repo)
- 액션 항목: "이전" 버튼 — Step 1로 복귀 (선택 유지)
- 액션 항목: "생성" 버튼 — 프로젝트 생성 실행

## 상태

| 상태 | 표현 |
|------|------|
| 다이얼로그 열림 (로딩) | Installation API 조회 중 — 스피너 |
| App 설치됨 | Step 1 저장소 선택 영역 |
| App 미설치 | Step 1 설치 유도 영역 |
| 설치 후 복귀 | Step 1 저장소 선택 영역으로 자동 전환 |
| 저장소 선택 완료 → Step 2 | 프로젝트 이름 영역 |
| 생성 중 | "생성" 버튼 비활성 + 스피너 |
| 생성 성공 | 다이얼로그 자동 닫힘 → `/projects/[id]`로 이동 |
| 저장소 목록 로딩 실패 | "저장소 목록을 불러올 수 없습니다" + 재시도 |
| 이름 빈값 | "프로젝트 이름을 입력해주세요" |
| 생성 실패 | "프로젝트 생성에 실패했습니다" + 재시도 |

## 인수조건

- [ ] 다이얼로그 열릴 때 Installation API로 설치 여부를 조회해 영역을 분기한다
- [ ] GitHub App이 설치된 저장소만 목록에 노출된다
- [ ] 이미 프로젝트에 연결된 저장소는 비활성 표시되며 선택할 수 없다
- [ ] 저장소는 1개만 선택 가능하다 (라디오)
- [ ] "GitHub App 설치하기" 클릭 시 설치 페이지로 이동하고, 설치 콜백(`/api/github/install/callback`)을 거쳐 다이얼로그로 복귀한다
- [ ] 복귀 시 다이얼로그 상태가 복원된다 (URL 파라미터 또는 sessionStorage 활용)
- [ ] 프로젝트 이름 기본값은 저장소명이며 수정 가능하다
- [ ] 빈 이름으로 "생성"을 시도하면 유효성 메시지를 표시한다
- [ ] "생성" 성공 시 `projects` row가 생성되고 `/projects/[id]`로 이동하며 다이얼로그가 자동으로 닫힌다
- [ ] 생성 중에는 버튼이 비활성된다
- [ ] 다이얼로그를 [X] 또는 오버레이 클릭으로 닫으면 입력 내용이 소멸된다

## 진입 경로

| 출발 | 트리거 |
|------|------|
| 프로젝트 목록 | 우상단 "새 프로젝트" 버튼 |
| 프로젝트 목록 (빈 상태) | 빈 상태 카드의 "새 프로젝트" 버튼 |
| GitHub App 설치 콜백 | 설치 완료 후 다이얼로그 복귀 |

## 비고

- 1 GitHub 저장소 = 1 프로젝트 (중복 연결 불가)
- 저장소·Admin 권한·collaborator 개념은 [features/PROJECT.md#access-접근-권한](../../../features/PROJECT.md#access-접근-권한) 참조
- GitHub App 설치 플로우 상세: [flows/AUTH.md#github-app-설치](../../../flows/AUTH.md#github-app-설치)
