---
title: 라우팅 및 페이지 구성
---

# 라우팅 및 페이지 구성

## 플랫폼 UI

- 데스크톱 전용
- 프로젝트 없을 때: shadcn empty 컴포넌트
- 와이어프레임 없을 때: 플러그인 사용법 안내 화면

**좌측 사이드바 (화면 목록):**
- 그룹 = 저장소 `docs/screens/` 아래 1-depth 폴더명 (한글 가능). 플러그인이 생성
- 그룹 폴더 없이 바로 화면 폴더가 있으면 "미분류" 그룹에 표시
- 정렬: 그룹/화면 모두 알파벳순 고정
- DB 테이블 없음 — GitHub 디렉토리 구조에서 실시간 파싱

## URL 라우팅

| 라우트 | 페이지 | 설명 |
|--------|--------|------|
| `/` | 랜딩 | 서비스 소개, 로그인 CTA. 로그인 상태면 `/projects`로 리다이렉트 |
| `/login` | 로그인 | GitHub 로그인 (기본) + 게스트 로그인 토글 (이메일+비밀번호) + 비밀번호 찾기 |
| `/onboarding` | 역할 선택 | 로그인 후 역할 미선택 시 리다이렉트. 기획자/개발자/디자이너 선택 |
| `/projects` | 프로젝트 목록 | 연결된 저장소 목록 + 프로젝트 생성 버튼. 정렬: 즐겨찾기 우선 → 생성일 내림차순 |
| `/projects/[id]` | 프로젝트 상세 | 첫 번째 화면 자동 선택 (리다이렉트) |
| `/projects/[id]/screens/[screenId]` | 화면 상세 | 와이어프레임 뷰어 + 코멘트 |
| `/projects/[id]/features` | 기능 명세 (추후) | features/ MD를 react-flow로 트리구조 시각화. **추후 구현** |
| `/invite/[token]` | 게스트 초대 | 토큰 검증 → 이메일 확인 → 기존 계정이면 비밀번호 로그인, 신규면 매직링크 → 이름+비밀번호 등록 → 승인 대기. 미인증 접근 허용 |
| `/invite/pending` | 승인 대기 | viewer(pending)가 보호 경로 접근 시 미들웨어가 rewrite. 이름+이메일 표시 |
| `/reset-password` | 비밀번호 재설정 | 재설정 이메일의 링크로 진입. Supabase가 URL fragment에 세션 토큰 포함. 새 비밀번호 입력 → updateUser. 미인증 접근 허용 |

**쿼리 파라미터:**
- `?version=v1.0` — 특정 버전으로 이동 (생략 시 "최신(미발행)")
- `?comment=123` — 특정 코멘트 하이라이트 + 핀 하이라이트
- 조합 가능: `?version=v1.0&comment=123`

**딥링크 접근 제어:**
- 비로그인 → 로그인 페이지로 이동 → 로그인 후 원래 URL로 복귀
- 로그인했지만 collaborator 아님 → 원 페이지의 서버 컴포넌트에서 `<Forbidden />` 컴포넌트 반환 ([권한 없음 화면](./screens/auth/forbidden/screen.md)). URL은 원래 경로 유지 (rewrite 아님)
- 게스트(viewer)는 `invitation_members.status = approved`인 프로젝트만 접근 가능
- 게스트(viewer, pending)가 `/projects` 또는 `/onboarding`에 접근 시 → 미들웨어에서 `/invite/pending`으로 rewrite (리다이렉트가 아닌 rewrite — URL은 유지됨)
- 승인된 게스트(viewer, approved)는 보호 경로에 정상 접근 가능

**미인증 접근 허용 라우트:**
- `/`, `/login`, `/invite/[token]`, `/invite/pending`, `/reset-password`

**콜백 라우트:**
- `/api/auth/callback` — Supabase Auth OAuth/매직링크 콜백. `provider` 필드로 GitHub/email 분기: GitHub이면 provider_token 저장, email이면 토큰 체크 스킵. `next`가 `/invite`로 시작하면 온보딩 리다이렉트 스킵
- `/api/github/install/callback` — GitHub App 설치 콜백
- `/api/github/webhook` — GitHub Webhook 수신

## 다이얼로그 / 시트

- 프로젝트 생성 → 다이얼로그 (저장소 선택). GitHub App 미설치 시 설치 안내 → 설치 후 저장소 선택
- 프로젝트 설정 → 시트 (톱니바퀴 아이콘): 프로젝트 이름 변경, 프로젝트 삭제 (Admin만)

## 프로젝트 삭제 시 정리 범위

- `projects` row 삭제 → `comments` cascade 삭제
- `github_cache`에서 해당 저장소 캐시 삭제
- GitHub 저장소/이슈는 유지 (플랫폼 데이터만 제거)
- Realtime 채널: 별도 정리 불필요 (구독자 없으면 자연 소멸)
- Webhook: 별도 해제 불필요 (수신 시 project가 DB에 없으면 무시)
- GitHub App 설치: 유지 (다른 프로젝트에서 같은 저장소를 다시 연결할 수 있으므로)

## 추후 결정

- 기능 명세(features/) → react-flow로 트리구조 시각화 예정. 배치 위치는 화면 보면서 결정

---

## 변경 이력

| 날짜 | 내용 |
|------|------|
| 2026-04-07 | SPEC.md에서 분리. 플랫폼 UI, URL 라우팅, 쿼리 파라미터, 딥링크, 콜백, 다이얼로그/시트, 프로젝트 삭제 정리 포함 |
| 2026-04-07 | 랜딩 페이지(`/`) 추가, 프로젝트 목록 라우트 `/projects`로 변경, 역할 선택(`/onboarding`) 페이지 추가 |
| 2026-04-08 | `/invite/[token]` 라우트 추가, 게스트 접근 제어 정책, 미인증 허용 라우트 정의 |
| 2026-04-08 | 구현 반영: 로그인 페이지에 이메일+비밀번호 폼 추가, `/invite/pending` 라우트 추가, 미들웨어 viewer rewrite 동작 명시, auth callback의 provider 분기 및 /invite 온보딩 스킵 |
| 2026-04-08 | `/reset-password` 라우트 추가, 미인증 허용 라우트에 포함 |
