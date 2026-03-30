# s05: Skills

`s01 > s02 > s03 > s04 > [ s05 ] s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"Load knowledge when you need it, not upfront"* -- 시스템 프롬프트가 아닌 tool_result로 주입한다.
>
> **하네스(Harness) 계층**: 온디맨드 지식 -- 모델이 요청할 때만 제공하는 도메인 전문성.

## 문제

에이전트가 도메인별 워크플로를 따르게 하고 싶다: Git 규칙, 테스트 패턴, 코드 리뷰 체크리스트. 이 모든 것을 시스템 프롬프트에 넣으면 사용하지 않는 스킬에 토큰을 낭비한다. 10개 스킬 x 2000 토큰 = 20,000 토큰, 대부분은 주어진 작업과 무관하다.

## 해결 방법

```
System prompt (Layer 1 -- always present):
+--------------------------------------+
| You are a coding agent.              |
| Skills available:                    |
|   - git: Git workflow helpers        |  ~100 tokens/skill
|   - test: Testing best practices     |
+--------------------------------------+

When model calls load_skill("git"):
+--------------------------------------+
| tool_result (Layer 2 -- on demand):  |
| <skill name="git">                   |
|   Full git workflow instructions...  |  ~2000 tokens
|   Step 1: ...                        |
| </skill>                             |
+--------------------------------------+
```

1계층: 스킬 *이름*을 시스템 프롬프트에 배치(저비용). 2계층: 스킬 *본문*을 tool_result로 전달(온디맨드).

## 동작 방식

1. 각 스킬은 YAML 프론트매터가 포함된 `SKILL.md` 파일을 가진 디렉터리로 구성된다.

```
skills/
  pdf/
    SKILL.md       # ---\n name: pdf\n description: Process PDF files\n ---\n ...
  code-review/
    SKILL.md       # ---\n name: code-review\n description: Review code\n ---\n ...
```

2. SkillLoader가 `SKILL.md` 파일을 재귀적으로 탐색하고, 디렉터리 이름을 스킬 식별자로 사용한다.

```python
class SkillLoader:
    def __init__(self, skills_dir: Path):
        self.skills = {}
        for f in sorted(skills_dir.rglob("SKILL.md")):
            text = f.read_text()
            meta, body = self._parse_frontmatter(text)
            name = meta.get("name", f.parent.name)
            self.skills[name] = {"meta": meta, "body": body}

    def get_descriptions(self) -> str:
        lines = []
        for name, skill in self.skills.items():
            desc = skill["meta"].get("description", "")
            lines.append(f"  - {name}: {desc}")
        return "\n".join(lines)

    def get_content(self, name: str) -> str:
        skill = self.skills.get(name)
        if not skill:
            return f"Error: Unknown skill '{name}'."
        return f"<skill name=\"{name}\">\n{skill['body']}\n</skill>"
```

3. 1계층은 시스템 프롬프트에 배치하고, 2계층은 일반적인 도구 핸들러로 처리한다.

```python
SYSTEM = f"""You are a coding agent at {WORKDIR}.
Skills available:
{SKILL_LOADER.get_descriptions()}"""

TOOL_HANDLERS = {
    # ...base tools...
    "load_skill": lambda **kw: SKILL_LOADER.get_content(kw["name"]),
}
```

모델은 어떤 스킬이 존재하는지 파악하고(저비용), 필요할 때만 불러온다(고비용).

## 변경 사항

| 구성 요소      | Before (s04)     | After (s05)                |
|----------------|------------------|----------------------------|
| Tools          | 5 (base + task)  | 5 (base + load_skill)      |
| System prompt  | Static string    | + skill descriptions       |
| Knowledge      | None             | skills/\*/SKILL.md files   |
| Injection      | None             | Two-layer (system + result)|

## 직접 실행해 보기

```sh
cd learn-claude-code
python agents/s05_skill_loading.py
```

1. `What skills are available?`
2. `Load the agent-builder skill and follow its instructions`
3. `I need to do a code review -- load the relevant skill first`
4. `Build an MCP server using the mcp-builder skill`
