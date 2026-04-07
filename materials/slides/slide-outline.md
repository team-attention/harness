---
title: "Harness Engineering"
subtitle: "AI가 잘 일하는 환경을 설계하는 기술"
author: "Hoyeon Lee"
style: executive-minimal
slides: 43
---

# Slide Outline

## Slide 01 — 타이틀
- **Harness Engineering**
- "AI가 잘 일하는 환경을 설계하는 기술"

## Slide 02 — 자기소개

## Slide 03 — 실제 사례: Harness의 힘 ★
- Harness 셋업을 해드렸더니, 날아다녔다
- 이민경님(디자이너) — AI Native Dashboard 2위 / 355명
- 영상 + 랭킹 스크린샷

## Slide 04 — 오늘 다룰 내용
- 01 왜 Harness Engineering인가
- 02 내 Harness 시연
- 03 준비 — 구조, 도구, 경계
- 04 맥락 — CLAUDE.md, 규칙, 점진적 노출
- 05 실행 설계 — 계획, 위임, 오케스트레이션
- 06 검증 & 개선

## Slide 05 — Hook
- "같은 Claude Code인데 왜 누구는 10분, 누구는 2시간이 걸리나?"

---

### 섹션 1: 왜 Harness Engineering인가

## Slide 06 — 프롬프트만으로는 부족하다
## Slide 07 — 진화 흐름
## Slide 08 — Harness Engineering 정의
## Slide 09 — Harness의 정의는 생각보다 넓다 ★
## Slide 10 — 성장 곡선

---

### 섹션 2: 큰 그림

## Slide 11 — Harness의 5개 축 — 순환 구조
- 준비 / 맥락 / 실행 / 검증 / 개선

---

### 섹션 3: 내 Harness 시연

## Slide 12 — 내 Harness 라이브 데모
- 프로젝트 구조 → 설정 파일 → 실제 작업 흐름
- [LIVE DEMO]

---

### 섹션 4: 준비 — 환경 세팅

## Slide 13 — 섹션 디바이더: 준비

## Slide 14 — 프로젝트 구조 설계 ★
- 1. Monorepo로 묶기 (context를 가깝게)
- 2. 역할별 폴더링 (docs/, tests/, .dev/, out/)
- 3. 아키텍처가 퀄리티를 결정

## Slide 15 — AI 도구 배치하기
- Skills, Hooks, Agents, MCP, Plugins

## Slide 16 — 경계 설정하기
- 뭘 알려줄까 / 어디까지 허용할까 / 뭘 막을까

---

### 섹션 5: 맥락 — AI에게 알려주기

## Slide 17 — 섹션 디바이더: 맥락

## Slide 18 — 설정 파일 한눈에 보기
- CLAUDE.md 계층: User → Project → Folder
- .claude/rules/ glob 조건부 로드

## Slide 19 — CLAUDE.md 실전 가이드
- User / Project / Folder 레벨별 뭘 쓰나

## Slide 20 — 맥락 관리의 핵심 원칙
- Progressive Disclosure / .claude/rules/ / Scope 계층

## Slide 21 — Progressive Disclosure
- 다 넣지 말고, 필요할 때 참조시키기

## Slide 22 — .claude/rules/ 가이드
- glob 패턴으로 조건부 규칙 로드

---

### 섹션 6: 실행 설계

## Slide 23 — 섹션 디바이더: 실행 설계

## Slide 24 — 핵심 흐름: 계획 → 실행 → 검증
- Plan → Execute → Verify 루프

## Slide 25 — 계획 세우게 하기 ★
- "해줘" 안티패턴 vs Plan → Execute

## Slide 26 — 계획의 진화 — 커스텀 Plan 스킬 ★
- Plan Mode → 커스텀 Plan 스킬 (/specify)

## Slide 27 — 실행 패턴: 혼자 vs 부하 파견 vs 팀
## Slide 28 — 상황별 오케스트레이션 패턴
## Slide 29 — Ralph Loop ★
## Slide 30 — Auto Research ★

---

### 섹션 7: 검증

## Slide 31 — 섹션 디바이더: 검증

## Slide 32 — 기준이 있어야 검증이 가능하다 ★
## Slide 33 — 컨텍스트를 나누고, 관점을 분리한다 ★
## Slide 34 — 모델도 나누고, 역할도 나눈다 ★
## Slide 35 — 안전장치
## Slide 36 — 에이전트에게 눈을 달아주기

---

### 섹션 8: 개선

## Slide 37 — 섹션 디바이더: 개선

## Slide 38 — 관측하고 개선하기
## Slide 39 — 단순화하기
## Slide 40 — 좋은 Harness의 신호

---

### 마무리

## Slide 41 — 오늘 다 알고 가야 하는 건 아닙니다
## Slide 42 — 지금 할 수 있는 것 ★
## Slide 43 — 감사 + Q&A
