# s02: Tool Use

`s01 > [ s02 ] s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"Adding a tool means adding one handler"* -- 루프는 그대로, 새 도구는 디스패치 맵에 등록하기만 하면 된다.
>
> **하네스(Harness) 계층**: 도구 디스패치 -- 모델이 닿을 수 있는 범위를 넓힌다.

## 문제

`bash` 하나만으로는 에이전트가 모든 것을 셸에 의존하게 된다. `cat`은 예측 불가능하게 출력을 잘라내고, `sed`는 특수 문자에서 실패하며, 모든 bash 호출은 제약 없는 보안 표면이 된다. `read_file`이나 `write_file` 같은 전용 도구를 쓰면 도구 수준에서 경로 샌드박싱을 강제할 수 있다.

핵심 포인트: 도구를 추가해도 루프를 바꿀 필요가 없다.

## 해결 방법

```
+--------+      +-------+      +------------------+
|  User  | ---> |  LLM  | ---> | Tool Dispatch    |
| prompt |      |       |      | {                |
+--------+      +---+---+      |   bash: run_bash |
                    ^           |   read: run_read |
                    |           |   write: run_wr  |
                    +-----------+   edit: run_edit |
                    tool_result | }                |
                                +------------------+

The dispatch map is a dict: {tool_name: handler_function}.
One lookup replaces any if/elif chain.
```

## 동작 방식

1. 각 도구에 핸들러 함수를 정의한다. 경로 샌드박싱으로 워크스페이스 밖으로의 탈출을 방지한다.

```python
def safe_path(p: str) -> Path:
    path = (WORKDIR / p).resolve()
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"Path escapes workspace: {p}")
    return path

def run_read(path: str, limit: int = None) -> str:
    text = safe_path(path).read_text()
    lines = text.splitlines()
    if limit and limit < len(lines):
        lines = lines[:limit]
    return "\n".join(lines)[:50000]
```

2. 디스패치 맵이 도구 이름과 핸들러를 연결한다.

```python
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"],
                                        kw["new_text"]),
}
```

3. 루프 안에서 이름으로 핸들러를 조회한다. 루프 본체는 s01에서 달라진 것이 없다.

```python
for block in response.content:
    if block.type == "tool_use":
        handler = TOOL_HANDLERS.get(block.name)
        output = handler(**block.input) if handler \
            else f"Unknown tool: {block.name}"
        results.append({
            "type": "tool_result",
            "tool_use_id": block.id,
            "content": output,
        })
```

도구 추가 = 핸들러 추가 + 스키마 항목 추가. 루프는 절대 바뀌지 않는다.

## s01에서의 변경 사항

| Component      | Before (s01)       | After (s02)                |
|----------------|--------------------|----------------------------|
| Tools          | 1 (bash only)      | 4 (bash, read, write, edit)|
| Dispatch       | Hardcoded bash call | `TOOL_HANDLERS` dict       |
| Path safety    | None               | `safe_path()` sandbox      |
| Agent loop     | Unchanged          | Unchanged                  |

## 직접 실행해 보기

```sh
cd learn-claude-code
python agents/s02_tool_use.py
```

1. `Read the file requirements.txt`
2. `Create a file called greet.py with a greet(name) function`
3. `Edit greet.py to add a docstring to the function`
4. `Read greet.py to verify the edit worked`
