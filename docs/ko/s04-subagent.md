# s04: Subagents

`s01 > s02 > s03 > [ s04 ] s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"Break big tasks down; each subtask gets a clean context"* -- 서브에이전트는 독립된 messages[]를 사용하여 메인 대화를 깔끔하게 유지한다.
>
> **하네스(Harness) 계층**: 컨텍스트 격리 -- 모델의 사고 명확성을 지킨다.

## 문제

에이전트가 작업을 진행할수록 messages 배열은 계속 커진다. 파일을 읽을 때마다, bash 출력을 받을 때마다 컨텍스트에 영구적으로 남는다. "이 프로젝트는 어떤 테스트 프레임워크를 쓰고 있나요?"라는 질문에 파일 5개를 읽어야 할 수도 있지만, 부모 에이전트에게 필요한 건 "pytest"라는 답변뿐이다.

## 해결 방법

```
Parent agent                     Subagent
+------------------+             +------------------+
| messages=[...]   |             | messages=[]      | <-- fresh
|                  |  dispatch   |                  |
| tool: task       | ----------> | while tool_use:  |
|   prompt="..."   |             |   call tools     |
|                  |  summary    |   append results |
|   result = "..." | <---------- | return last text |
+------------------+             +------------------+

Parent context stays clean. Subagent context is discarded.
```

## 동작 방식

1. 부모에게 `task` 도구를 추가한다. 자식은 `task`를 제외한 모든 기본 도구를 갖는다(재귀적 생성 불가).

```python
PARENT_TOOLS = CHILD_TOOLS + [
    {"name": "task",
     "description": "Spawn a subagent with fresh context.",
     "input_schema": {
         "type": "object",
         "properties": {"prompt": {"type": "string"}},
         "required": ["prompt"],
     }},
]
```

2. 서브에이전트는 `messages=[]`로 시작하여 자체 루프를 실행한다. 최종 텍스트만 부모에게 반환된다.

```python
def run_subagent(prompt: str) -> str:
    sub_messages = [{"role": "user", "content": prompt}]
    for _ in range(30):  # 안전 제한
        response = client.messages.create(
            model=MODEL, system=SUBAGENT_SYSTEM,
            messages=sub_messages,
            tools=CHILD_TOOLS, max_tokens=8000,
        )
        sub_messages.append({"role": "assistant",
                             "content": response.content})
        if response.stop_reason != "tool_use":
            break
        results = []
        for block in response.content:
            if block.type == "tool_use":
                handler = TOOL_HANDLERS.get(block.name)
                output = handler(**block.input)
                results.append({"type": "tool_result",
                    "tool_use_id": block.id,
                    "content": str(output)[:50000]})
        sub_messages.append({"role": "user", "content": results})
    return "".join(
        b.text for b in response.content if hasattr(b, "text")
    ) or "(no summary)"
```

자식의 전체 메시지 히스토리(30회 이상의 도구 호출 포함)는 폐기된다. 부모는 한 단락 분량의 요약을 일반 `tool_result`로 받는다.

## s03에서의 변경 사항

| Component      | Before (s03)     | After (s04)               |
|----------------|------------------|---------------------------|
| Tools          | 5                | 5 (base) + task (parent)  |
| Context        | Single shared    | Parent + child isolation  |
| Subagent       | None             | `run_subagent()` function |
| Return value   | N/A              | Summary text only         |

## 직접 실행해 보기

```sh
cd learn-claude-code
python agents/s04_subagent.py
```

1. `Use a subtask to find what testing framework this project uses`
2. `Delegate: read all .py files and summarize what each one does`
3. `Use a task to create a new module, then verify it from here`
