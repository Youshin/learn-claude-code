# s03: TodoWrite

`s01 > s02 > [ s03 ] s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"An agent without a plan drifts"* -- 먼저 단계를 나열하고, 그다음 실행한다.
>
> **하네스(Harness) 계층**: 계획 수립 -- 경로를 정해주지 않으면서도 모델이 궤도를 이탈하지 않게 한다.

## 문제

여러 단계로 이루어진 작업에서 모델은 길을 잃는다. 같은 작업을 반복하거나, 단계를 건너뛰거나, 엉뚱한 곳으로 빠진다. 대화가 길어질수록 더 심해지는데, 도구 결과가 컨텍스트를 채우면서 시스템 프롬프트의 영향력이 희미해지기 때문이다. 10단계짜리 리팩토링에서 1~3단계를 끝낸 뒤, 4~10단계를 잊어버리고 즉흥적으로 행동하기 시작한다.

## 해결 방법

```
+--------+      +-------+      +---------+
|  User  | ---> |  LLM  | ---> | Tools   |
| prompt |      |       |      | + todo  |
+--------+      +---+---+      +----+----+
                    ^                |
                    |   tool_result  |
                    +----------------+
                          |
              +-----------+-----------+
              | TodoManager state     |
              | [ ] task A            |
              | [>] task B  <- doing  |
              | [x] task C            |
              +-----------------------+
                          |
              if rounds_since_todo >= 3:
                inject <reminder> into tool_result
```

## 동작 방식

1. TodoManager가 상태를 가진 항목 목록을 관리한다. `in_progress` 상태는 한 번에 하나만 허용된다.

```python
class TodoManager:
    def update(self, items: list) -> str:
        validated, in_progress_count = [], 0
        for item in items:
            status = item.get("status", "pending")
            if status == "in_progress":
                in_progress_count += 1
            validated.append({"id": item["id"], "text": item["text"],
                              "status": status})
        if in_progress_count > 1:
            raise ValueError("Only one task can be in_progress")
        self.items = validated
        return self.render()
```

2. `todo` 도구는 다른 도구와 마찬가지로 디스패치 맵에 추가된다.

```python
TOOL_HANDLERS = {
    # ...base tools...
    "todo": lambda **kw: TODO.update(kw["items"]),
}
```

3. nag 리마인더가 모델이 3라운드 이상 `todo`를 호출하지 않을 경우 알림을 주입한다.

```python
if rounds_since_todo >= 3 and messages:
    last = messages[-1]
    if last["role"] == "user" and isinstance(last.get("content"), list):
        last["content"].insert(0, {
            "type": "text",
            "text": "<reminder>Update your todos.</reminder>",
        })
```

"한 번에 in_progress는 하나만" 제약이 순차적 집중을 강제하고, nag 리마인더가 책임감을 부여한다.

## s02에서의 변경 사항

| Component      | Before (s02)     | After (s03)                |
|----------------|------------------|----------------------------|
| Tools          | 4                | 5 (+todo)                  |
| Planning       | None             | TodoManager with statuses  |
| Nag injection  | None             | `<reminder>` after 3 rounds|
| Agent loop     | Simple dispatch  | + rounds_since_todo counter|

## 직접 실행해 보기

```sh
cd learn-claude-code
python agents/s03_todo_write.py
```

1. `Refactor the file hello.py: add type hints, docstrings, and a main guard`
2. `Create a Python package with __init__.py, utils.py, and tests/test_utils.py`
3. `Review all Python files and fix any style issues`
