# s01: The Agent Loop

`[ s01 ] s02 > s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"One loop & Bash is all you need"* -- 도구 하나 + 루프 하나 = 에이전트.
>
> **하네스(Harness) 계층**: 루프 -- 모델이 현실 세계와 만나는 첫 번째 접점.

## 문제

언어 모델은 코드에 대해 추론할 수 있지만, 현실 세계에 손을 뻗을 수는 없다. 파일을 읽지도, 테스트를 실행하지도, 에러를 확인하지도 못한다. 루프가 없으면 도구를 호출할 때마다 사용자가 직접 결과를 복사해서 붙여넣어야 한다. 사용자 자신이 루프가 되는 셈이다.

## 해결 방법

```
+--------+      +-------+      +---------+
|  User  | ---> |  LLM  | ---> |  Tool   |
| prompt |      |       |      | execute |
+--------+      +---+---+      +----+----+
                    ^                |
                    |   tool_result  |
                    +----------------+
                    (loop until stop_reason != "tool_use")
```

하나의 종료 조건이 전체 흐름을 제어한다. 모델이 도구 호출을 멈출 때까지 루프는 계속 돈다.

## 동작 방식

1. 사용자의 프롬프트가 첫 번째 메시지가 된다.

```python
messages.append({"role": "user", "content": query})
```

2. 메시지와 도구 정의를 LLM에 전송한다.

```python
response = client.messages.create(
    model=MODEL, system=SYSTEM, messages=messages,
    tools=TOOLS, max_tokens=8000,
)
```

3. 어시스턴트 응답을 추가하고 `stop_reason`을 확인한다. 모델이 도구를 호출하지 않았다면 종료.

```python
messages.append({"role": "assistant", "content": response.content})
if response.stop_reason != "tool_use":
    return
```

4. 각 도구 호출을 실행하고 결과를 수집하여 user 메시지로 추가한다. 2단계로 돌아간다.

```python
results = []
for block in response.content:
    if block.type == "tool_use":
        output = run_bash(block.input["command"])
        results.append({
            "type": "tool_result",
            "tool_use_id": block.id,
            "content": output,
        })
messages.append({"role": "user", "content": results})
```

하나의 함수로 조립하면:

```python
def agent_loop(query):
    messages = [{"role": "user", "content": query}]
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            return

        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = run_bash(block.input["command"])
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": output,
                })
        messages.append({"role": "user", "content": results})
```

이것이 30줄도 안 되는 에이전트의 전부다. 이 과정의 나머지는 모두 이 루프 위에 쌓아 올리는 것이며, 루프 자체는 변하지 않는다.

## 변경 사항

| Component     | Before     | After                          |
|---------------|------------|--------------------------------|
| Agent loop    | (none)     | `while True` + stop_reason     |
| Tools         | (none)     | `bash` (one tool)              |
| Messages      | (none)     | Accumulating list              |
| Control flow  | (none)     | `stop_reason != "tool_use"`    |

## 직접 실행해 보기

```sh
cd learn-claude-code
python agents/s01_agent_loop.py
```

1. `Create a file called hello.py that prints "Hello, World!"`
2. `List all Python files in this directory`
3. `What is the current git branch?`
4. `Create a directory called test_output and write 3 files in it`
