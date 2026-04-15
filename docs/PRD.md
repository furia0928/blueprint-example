---
title: PRD — Blueprint Platform
---

# PRD — Blueprint Platform

> 이 문서는 제품 전체를 조감하는 요약본이다. 각 항목의 상세 기능명세는 개별 SSOT 문서를 참조할 것.

## 제품 개요

와이어프레임 기반 협업 플랫폼.
기획자가 외부 플러그인(Claude Desktop 등)으로 생성한 와이어프레임(HTML)과 화면명세·기능명세(MD)를 GitHub에 push하면, 팀원(개발자·디자이너)이 열람하고 **핀 코멘트**로 피드백하는 서비스.

**핵심 가치:** 와이어프레임 위에 직접 핀을 찍어 맥락이 담긴 피드백을 남기고, 버전별로 이력을 추적하여 기획 ↔ 개발 간 이해도 격차를 줄인다.

## 사용자 및 역할

| 역할 | 작업 환경 | 주요 행동 |
|------|----------|----------|
| 기획자 | Claude Desktop + Git | 와이어프레임·화면명세·기능명세 작성, Git push, 코멘트 취합 후 수정·재업로드 |
| 개발자 | Claude Code + IDE | 와이어프레임 열람, 코멘트·피드백 |
| 디자이너 | — | 와이어프레임 열람, 코멘트·피드백 |

- GitHub 계정 필수 (GitHub 소셜 로그인만 지원)
- 권한은 GitHub 저장소 collaborator 기반 (플랫폼 자체 권한 모델 없음)
- 데스크톱 전용

## 핵심 기능

| 기능 | 요약 | 상세 |
|------|------|------|
| 와이어프레임 뷰어 | GitHub 저장소의 HTML을 iframe srcdoc로 렌더링. 파일명 기반 뷰포트 너비 자동 결정. 와이어프레임 HTML 자체에 기능 설명이 임베드되어 iframe 안의 우측 레일로 함께 렌더됨 (플랫폼 우측 패널 없음) | [features/WIREFRAME.md](./features/WIREFRAME.md) |
| 기능 설명 (임베드) | 플러그인이 `*screen*.md` + `docs/features/*.md`를 읽어 와이어프레임 HTML 생성 시 `<bp-description>` 컴포넌트로 섹션별 요소 설명을 임베드. 런타임에 우측 레일로 렌더 | [features/WIREFRAME.md](./features/WIREFRAME.md#embed-기능-설명-임베드) |
| 핀 코멘트 | 와이어프레임의 기능(feature) 영역을 클릭하여 GitHub Issue로 코멘트 생성. 좌표는 feature div 기준 비율(0~1) | [features/COMMENT.md](./features/COMMENT.md) |
| 버전 관리 | Git 태그 기반 메이저/마이너 버전 발행. 버전별 코멘트 분리, 미해결 코멘트 이월 | [features/VERSION.md](./features/VERSION.md) |
| 실시간 협업 | Supabase Realtime으로 코멘트 실시간 반영 + 와이어프레임 업데이트 토스트 알림 | [features/COMMENT.md#SYNC](./features/COMMENT.md#sync-실시간웹훅-동기화) |
| 프로젝트 관리 | GitHub App 설치로 저장소 연결 (1 저장소 = 1 프로젝트). 즐겨찾기, 설정, 삭제 | [ROUTING.md](./ROUTING.md) |

## 아키텍처

```
기획자 (Claude Desktop)
  │ git push
  ▼
GitHub 저장소 ◄─── GitHub App (파일 읽기, Webhook, 태그)
  │                         │
  │  Webhook push           │ Contents API
  ▼                         ▼
Vercel (Next.js 16) ──► Supabase
  │                     ├─ Auth (GitHub OAuth)
  │                     ├─ DB (프로젝트, 코멘트 메타)
  │                     └─ Realtime (실시간 동기화)
  ▼
사용자 브라우저
  ├─ 좌측 사이드바 (화면 목록)
  ├─ 중앙 뷰어 — 와이어프레임 iframe
  │    └─ iframe 내부: bp-page + 우측 설명 레일 (<aside class="bp-right-rail">)
  │       ※ 우측 레일은 HTML 자체에 임베드됨. 플랫폼은 별도 패널 안 그림
  └─ 핀 코멘트 오버레이 (iframe 위)
```

### 데이터 흐름 원칙

| 데이터 | 저장소 | 이유 |
|--------|--------|------|
| 와이어프레임 HTML, 화면명세·기능명세 MD | GitHub (Contents API) | 기획자가 Git으로 관리 |
| 코멘트 본문, 스레드 | GitHub Issues | 실제 작성자 기록, @멘션 알림 활용 |
| 코멘트 좌표, 상태, 메타데이터 | Supabase DB | 실시간 동기화, 빠른 조회 |
| 프로젝트 설정, 즐겨찾기 | Supabase DB | 플랫폼 고유 데이터 |
| 사용자 인증, 토큰 | Supabase Auth + DB | 세션 관리, 토큰 암호화 저장 |
| 멤버/권한 | GitHub (collaborator) | 별도 관리 불필요 |

### 기술 스택

| 분류 | 기술 | 비고 |
|------|------|------|
| 프레임워크 | Next.js 16 (App Router) | 풀스택 |
| 배포 | Vercel | 호스팅·CI/CD |
| 모니터링 | Vercel Analytics | 성능/사용량 |
| UI | shadcn/ui (base-ui 기반) + Tailwind v4 | 컴포넌트 + 스타일링 |
| 상태 관리 | Zustand | 클라이언트 상태 |
| 폼 | shadcn Field 컴포넌트 (네이티브 폼) | react-hook-form/zod 미사용 |
| 인증 | Supabase Auth (GitHub OAuth) | — |
| DB | Supabase (PostgreSQL) | 프로젝트·코멘트 메타 |
| 실시간 | Supabase Realtime | 코멘트 동기화 |
| Supabase SDK | @supabase/supabase-js + @supabase/ssr | Auth + DB + Realtime. DB 접근은 모두 Server Action 경유 (클라이언트 직접 DB 접근 금지). 클라이언트는 Auth + Realtime 구독만 |
| GitHub 연동 | GitHub App + OAuth App (이중 토큰) | 파일/이슈/Webhook |
| HTTP | fetch (래퍼 함수) | GitHub API 호출 |
| 마크다운 | react-markdown + remark-gfm | 화면명세·기능명세 MD 렌더링 |
| 와이어프레임 공통 리소스 | Vercel `public/` (버전별 경로) | `base.css`, `bp-components.js` — 플랫폼 배포 시 자동 반영. 플러그인이 생성 시점 버전을 참조하여 하위 호환 보장 |

### 주요 설계 결정

| 결정 | 이유 |
|------|------|
| GitHub Issues를 코멘트로 사용 | 작성자 귀속, @멘션, 스레드를 GitHub 인프라에 위임 |
| Supabase DB 병행 | 좌표/상태의 실시간 동기화, 빠른 목록 조회가 Issues API로는 불가 |
| GitHub App + OAuth App 이중 구조 | 사람의 행동(이슈 작성)은 사용자 토큰, 시스템의 행동(파일 읽기)은 App 토큰 |
| OAuth scope `repo` | private 저장소 이슈 접근에 필수. 이슈 전용 scope가 없음 |
| DB 접근은 Server Action 경유만 | 클라이언트에서 직접 DB 접근 금지. 클라이언트는 Auth + Realtime만 |
| feature div 기준 비율 좌표 | 화면 크기 무관 핀 위치 유지, div 이동 시 핀도 함께 이동 |
| iframe sandbox (allow-scripts only) | allow-same-origin 금지 — parent 세션/토큰 접근 차단 |
| Webhook + Broadcast 알림 | 자동 새로고침 안 함 — 사용자가 직접 결정 |

## 페이지 구성

| 라우트 | 설명 |
|--------|------|
| `/login` | GitHub 소셜 로그인 + 역할 선택 |
| `/` | 프로젝트 목록 (즐겨찾기 우선 → 생성일순) |
| `/projects/[id]/screens/[screenId]` | 와이어프레임 뷰어 + 코멘트 (핵심 화면) |

상세: [ROUTING.md](./ROUTING.md)

## 상세 문서 (SSOT)

| 문서 | 범위 |
|------|------|
| [features/WIREFRAME.md](./features/WIREFRAME.md) | 저장소 구조, 화면 인식, 렌더링, 브릿지, 뷰포트, 기능 설명 임베드, 상태 변형, 에러 대응 |
| [features/COMMENT.md](./features/COMMENT.md) | 코멘트 생성/수정/해결, 메타데이터, 라벨·담당자, 조회 전략, 딥링크 |
| [features/VERSION.md](./features/VERSION.md) | 버전 발행 흐름, 과거 버전 규칙, 이월 코멘트, 미발행 변경 감지 |
| [INTEGRATION.md](./INTEGRATION.md) | GitHub App/OAuth 역할, Webhook, 캐싱, API 에러 대응 |
| [AUTH.md](./features/AUTH.md) | 로그인 플로우, 토큰 관리, 보안 정책 |
| [DB.md](./DB.md) | 전체 테이블 스키마, RLS 정책 |
| [ROUTING.md](./ROUTING.md) | 페이지 구성, URL 라우팅, 딥링크, 다이얼로그 |

---

## 변경 이력

| 날짜 | 내용 |
|------|------|
| 2026-04-07 | 초안 작성. 기존 기능명세 8개 문서 기반 통합 요약 |
| 2026-04-13 | 플랫폼 우측 패널(panel-features) 폐기 반영. 기능 설명은 와이어프레임 HTML에 임베드되어 iframe 안의 우측 레일로 렌더됨. 플랫폼은 2패널(좌 사이드바 + 중앙 뷰어) 구조로 단순화. 상세: [features/WIREFRAME.md](./features/WIREFRAME.md#embed-기능-설명-임베드) |
