# harness-session

Harness Engineering 세션을 위한 Claude Code 플러그인.
수강생들이 실습하고 참고할 수 있는 스킬, 훅, 발표 자료를 포함한다.

## Project Structure

```
.claude-plugin/     # Plugin manifest (plugin.json)
skills/             # Skill implementations
  check-harness/    #   Harness 성숙도 진단 (5축 35개 체크리스트)
  scaffold/         #   Greenfield 프로젝트 하네스 스캐폴딩
  specify/          #   Goal → Implementation plan (spec.md)
  deep-interview/   #   Socratic 요구사항 인터뷰
hooks/              # Hook registration (hooks.json)
materials/          # 세션 발표 자료
  harness-checklist.md  # Harness Engineering 체크리스트
  slides/               # HTML 슬라이드 (slide-01 ~ slide-50)
.claude/            # Project-level Claude Code config
  settings.json
```

## Development Guidelines

- Skills: one directory per skill in `skills/{name}/SKILL.md`
- Hooks: register in `hooks/hooks.json`, implement in `scripts/`
- Hook scripts use `${CLAUDE_PLUGIN_ROOT}` for paths
- Version in `plugin.json` is the source of truth
- 출력물은 `.md` 파일로 생성 (spec.json, hoyeon-cli 사용하지 않음)
