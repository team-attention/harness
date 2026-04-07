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

- `materials/harness-checklist.md` — Harness Engineering 3단계 성숙도 체크리스트
- `materials/slides/` — 세션 발표 슬라이드 (HTML, 50장)

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
