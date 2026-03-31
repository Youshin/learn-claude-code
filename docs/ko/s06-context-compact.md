# s06: Context Compact

`s01 > s02 > s03 > s04 > s05 > [ s06 ] | s07 > s08 > s09 > s10 > s11 > s12`

> *"Context will fill up; you need a way to make room"* -- 3계층 압축 전략으로 무한 세션을 구현한다.
>
> **하네스(Harness) 계층**: 압축 -- 깨끗한 메모리로 무한 세션을 유지한다.

## 문제

컨텍스트 윈도우는 유한하다. 1000줄짜리 파일에 `read_file` 한 번이면 약 4000 토큰을 소비한다. 파일 30개를 읽고 bash 명령 20번을 실행하면 100,000 토큰을 넘긴다. 압축 없이는 에이전트가 대규모 코드베이스에서 작업할 수 없다.

## 해결 방법

적극성을 단계적으로 높이는 3계층 구조:

```
Every turn:
+------------------+
| Tool call result |
+------------------+
        |
        v
[Layer 1: micro_compact]        (silent, every turn)
  Replace tool_result > 3 turns old
  with "[Previous: used {tool_name}]"
        |
        v
[Check: tokens > 50000?]
   |               |
   no              yes
   |               |
   v               v
continue    [Layer 2: auto_compact]
              Save transcript to .transcripts/
              LLM summarizes conversation.
              Replace all messages with [summary].
                    |
                    v
            [Layer 3: compact tool]
              Model calls compact explicitly.
              Same summarization as auto_compact.
```

## 동작 방식

1. **1계층 -- micro_compact**: 매 LLM 호출 전에 오래된 도구 결과를 플레이스홀더로 교체한다.

```python
def micro_compact(messages: list) -> list:
    tool_results = []
    for i, msg in enumerate(messages):
        if msg["role"] == "user" and isinstance(msg.get("content"), list):
            for j, part in enumerate(msg["content"]):
                if isinstance(part, dict) and part.get("type") == "tool_result":
                    tool_results.append((i, j, part))
    if len(tool_results) <= KEEP_RECENT:
        return messages
    for _, _, part in tool_results[:-KEEP_RECENT]:
        if len(part.get("content", "")) > 100:
            part["content"] = f"[Previous: used {tool_name}]"
    return messages
```

2. **2계층 -- auto_compact**: 토큰이 임계값을 초과하면 전체 트랜스크립트를 디스크에 저장한 뒤 LLM에 요약을 요청한다.

```python
def auto_compact(messages: list) -> list:
    # 복구를 위해 트랜스크립트 저장
    transcript_path = TRANSCRIPT_DIR / f"transcript_{int(time.time())}.jsonl"
    with open(transcript_path, "w") as f:
        for msg in messages:
            f.write(json.dumps(msg, default=str) + "\n")
    # LLM이 요약
    response = client.messages.create(
        model=MODEL,
        messages=[{"role": "user", "content":
            "Summarize this conversation for continuity..."
            + json.dumps(messages, default=str)[:80000]}],
        max_tokens=2000,
    )
    return [
        {"role": "user", "content": f"[Compressed]\n\n{response.content[0].text}"},
    ]
```

3. **3계층 -- 수동 compact**: `compact` 도구가 동일한 요약 처리를 온디맨드로 실행한다.

4. 에이전트 루프가 세 계층을 모두 통합한다:

```python
def agent_loop(messages: list):
    while True:
        micro_compact(messages)                        # Layer 1
        if estimate_tokens(messages) > THRESHOLD:
            messages[:] = auto_compact(messages)       # Layer 2
        response = client.messages.create(...)
        # ... 도구 실행 ...
        if manual_compact:
            messages[:] = auto_compact(messages)       # Layer 3
```

트랜스크립트가 디스크에 전체 이력을 보존한다. 진정으로 사라지는 것은 없다 -- 활성 컨텍스트 밖으로 옮겨질 뿐이다.

## 변경 사항

| 구성 요소      | Before (s05)     | After (s06)                |
|----------------|------------------|----------------------------|
| Tools          | 5                | 5 (base + compact)         |
| Context mgmt   | None             | Three-layer compression    |
| Micro-compact  | None             | Old results -> placeholders|
| Auto-compact   | None             | Token threshold trigger    |
| Transcripts    | None             | Saved to .transcripts/     |

## 직접 실행해 보기

```sh
cd learn-claude-code
python agents/s06_context_compact.py
```

1. `Read every Python file in the agents/ directory one by one` (micro-compact가 오래된 결과를 교체하는 과정을 관찰한다)
2. `Keep reading files until compression triggers automatically`
3. `Use the compact tool to manually compress the conversation`
