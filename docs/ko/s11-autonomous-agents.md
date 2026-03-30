# s11: Autonomous Agents

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > [ s11 ] s12`

> *"팀원이 보드를 스캔해서 직접 태스크를 가져간다"* -- 리드가 일일이 지시할 필요가 없다.
>
> **하네스(Harness) 레이어**: 자율성 -- 지시 없이 스스로 일을 찾는 모델.

## 문제

s09-s10에서 팀원들은 명시적으로 지시받아야만 일한다. 리드가 각 팀원을 특정 프롬프트로 생성해야 한다. 보드에 미할당 태스크가 10개? 리드가 하나씩 배정한다. 확장이 안 된다.

진정한 자율성: 팀원이 태스크 보드를 직접 스캔하고, 미할당 태스크를 선점하고, 작업을 수행한 뒤, 다음 일을 찾는다.

한 가지 주의점: 컨텍스트 압축(s06) 이후 에이전트가 자신이 누구인지 잊을 수 있다. 정체성 재주입으로 해결한다.

## 해결 방법

```
Teammate lifecycle with idle cycle:

+-------+
| spawn |
+---+---+
    |
    v
+-------+   tool_use     +-------+
| WORK  | <------------- |  LLM  |
+---+---+                +-------+
    |
    | stop_reason != tool_use (or idle tool called)
    v
+--------+
|  IDLE  |  poll every 5s for up to 60s
+---+----+
    |
    +---> check inbox --> message? ----------> WORK
    |
    +---> scan .tasks/ --> unclaimed? -------> claim -> WORK
    |
    +---> 60s timeout ----------------------> SHUTDOWN

Identity re-injection after compression:
  if len(messages) <= 3:
    messages.insert(0, identity_block)
```

## 작동 원리

1. 팀원 루프는 WORK와 IDLE 두 단계로 구성된다. LLM이 도구 호출을 중단하거나 `idle`을 호출하면, 팀원은 IDLE 상태에 진입한다.

```python
def _loop(self, name, role, prompt):
    while True:
        # -- WORK PHASE --
        messages = [{"role": "user", "content": prompt}]
        for _ in range(50):
            response = client.messages.create(...)
            if response.stop_reason != "tool_use":
                break
            # execute tools...
            if idle_requested:
                break

        # -- IDLE PHASE --
        self._set_status(name, "idle")
        resume = self._idle_poll(name, messages)
        if not resume:
            self._set_status(name, "shutdown")
            return
        self._set_status(name, "working")
```

2. IDLE 단계에서는 수신함과 태스크 보드를 반복적으로 폴링한다.

```python
def _idle_poll(self, name, messages):
    for _ in range(IDLE_TIMEOUT // POLL_INTERVAL):  # 60s / 5s = 12
        time.sleep(POLL_INTERVAL)
        inbox = BUS.read_inbox(name)
        if inbox:
            messages.append({"role": "user",
                "content": f"<inbox>{inbox}</inbox>"})
            return True
        unclaimed = scan_unclaimed_tasks()
        if unclaimed:
            claim_task(unclaimed[0]["id"], name)
            messages.append({"role": "user",
                "content": f"<auto-claimed>Task #{unclaimed[0]['id']}: "
                           f"{unclaimed[0]['subject']}</auto-claimed>"})
            return True
    return False  # 타임아웃 -> 종료
```

3. 태스크 보드 스캔: 대기 중이고, 소유자가 없고, 차단되지 않은 태스크를 찾는다.

```python
def scan_unclaimed_tasks() -> list:
    unclaimed = []
    for f in sorted(TASKS_DIR.glob("task_*.json")):
        task = json.loads(f.read_text())
        if (task.get("status") == "pending"
                and not task.get("owner")
                and not task.get("blockedBy")):
            unclaimed.append(task)
    return unclaimed
```

4. 정체성 재주입: 컨텍스트가 너무 짧으면(압축이 발생한 경우), 정체성 블록을 삽입한다.

```python
if len(messages) <= 3:
    messages.insert(0, {"role": "user",
        "content": f"<identity>You are '{name}', role: {role}, "
                   f"team: {team_name}. Continue your work.</identity>"})
    messages.insert(1, {"role": "assistant",
        "content": f"I am {name}. Continuing."})
```

## s10 대비 변경 사항

| 구성 요소      | 이전 (s10)       | 이후 (s11)                       |
|----------------|------------------|----------------------------------|
| 도구           | 12개             | 14개 (+idle, +claim_task)        |
| 자율성         | 리드 지시형      | 자기 조직화                      |
| IDLE 단계      | 없음             | 수신함 + 태스크 보드 폴링        |
| 태스크 선점    | 수동만 가능      | 미할당 태스크 자동 선점          |
| 정체성         | 시스템 프롬프트  | + 압축 후 재주입                 |
| 타임아웃       | 없음             | 60초 유휴 시 자동 종료           |

## 직접 해보기

```sh
cd learn-claude-code
python agents/s11_autonomous_agents.py
```

1. `Create 3 tasks on the board, then spawn alice and bob. Watch them auto-claim.`
2. `Spawn a coder teammate and let it find work from the task board itself`
3. `Create tasks with dependencies. Watch teammates respect the blocked order.`
4. `/tasks`를 입력하면 소유자가 표시된 태스크 보드를 확인할 수 있다
5. `/team`을 입력하면 누가 작업 중이고 누가 유휴 상태인지 확인할 수 있다
