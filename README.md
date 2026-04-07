# harness-session

**Harness Engineering** 세션을 위한 Claude Code 플러그인.

AI 에이전트가 잘 일하는 환경을 설계하는 기술 — Harness Engineering의 핵심 개념을 실습하고, 바로 써볼 수 있는 스킬과 자료를 제공한다.

## Skills

| Skill | 설명 | 사용법 |
|-------|------|--------|
| **check-harness** | 현재 프로젝트의 Harness 성숙도를 5축 35개 체크리스트로 진단 | `/check-harness` |
| **scaffold** | Greenfield 프로젝트에 AI-optimized 하네스 구조를 스캐폴딩 | `/scaffold` |
| **specify** | 목표를 구조화된 구현 계획(spec.md)으로 변환 | `/specify "목표"` |
| **deep-interview** | Socratic 방식의 요구사항 인터뷰 (Ambiguity Score 기반) | `/deep-interview "주제"` |

## Materials

### Harness Engineering 체크리스트

`materials/harness-checklist.md`

AI가 잘 일하는 환경을 설계하기 위한 자가진단 체크리스트. 3단계 성숙도(L1 시작하기 → L2 내 것으로 만들기 → L3 자율 운영)로 나뉘며, 5개 축에 걸쳐 35개 항목을 점검한다.

- **준비 (Scaffolding)** — AI가 프로젝트를 스스로 파악할 수 있는가
- **맥락 (Context)** — CLAUDE.md, 규칙, 점진적 노출
- **실행 설계 (Execution)** — 계획, 위임, 오케스트레이션
- **검증 (Verification)** — 테스트, 리뷰, 품질 관리
- **개선 (Improvement)** — 학습, 피드백 루프

### 발표 슬라이드

`materials/slides/`

"Harness Engineering — AI가 잘 일하는 환경을 설계하는 기술" 세션 발표 자료.
HTML 슬라이드 50장 + `viewer.html`로 로컬에서 바로 열어볼 수 있다.

```bash
# 슬라이드 뷰어 열기
open materials/slides/viewer.html
```

## Quick Start

```bash
# 1. 이 플러그인이 있는 디렉토리에서 Claude Code 실행
cd harness-session
claude

# 2. 현재 프로젝트의 하네스 성숙도 진단
/check-harness

# 3. 새 프로젝트에 하네스 스캐폴딩
/scaffold

# 4. 요구사항이 불명확할 때 인터뷰
/deep-interview 뭘 만들어야 할지 모르겠어

# 5. 목표를 구현 계획으로 변환
/specify "사용자 인증 시스템 구현"
```

## Project Structure

```
.claude-plugin/plugin.json    # Plugin manifest
skills/
  check-harness/SKILL.md      # Harness 성숙도 진단
  scaffold/SKILL.md            # 프로젝트 스캐폴딩
  specify/SKILL.md             # Goal → spec.md
  deep-interview/SKILL.md     # Socratic 인터뷰
hooks/hooks.json               # Hook 등록 (빈 템플릿)
materials/                     # 세션 발표 자료
.claude/settings.json          # Claude Code 프로젝트 설정
```

## License

Internal use only.
