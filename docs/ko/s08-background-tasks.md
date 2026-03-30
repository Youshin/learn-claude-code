# s08: Background Tasks

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > [ s08 ] s09 > s10 > s11 > s12`

> *"Run slow operations in the background; the agent keeps thinking"* -- 데몬 스레드가 명령을 실행하고, 완료 시 알림을 주입한다.
>
> **하네스(Harness) 계층**: 백그라운드 실행 -- 모델이 사고를 이어가는 동안 하네스(Harness)가 대기한다.

## 문제

일부 명령은 수 분이 걸린다: `npm install`, `pytest`, `docker build`. 블로킹 루프에서는 모델이 서브프로세스 완료를 기다리며 가만히 멈춰 있다. 사용자가 "의존성을 설치하고, 그동안 설정 파일을 만들어줘"라고 해도, 에이전트는 병렬이 아닌 순차적으로 처리한다.

## 해결 방법

```
Main thread                Background thread
+-----------------+        +-----------------+
| agent loop      |        | subprocess runs |
| ...             |        | ...             |
| [LLM call] <---+------- | enqueue(result) |
|  ^drain queue   |        +-----------------+
+-----------------+

Timeline:
Agent --[spawn A]--[spawn B]--[other work]----
             |          |
             v          v
          [A runs]   [B runs]      (parallel)
             |          |
             +-- results injected before next LLM call --+
```

## 동작 방식

1. BackgroundManager가 스레드 안전한 알림 큐로 태스크를 추적한다.

```python
class BackgroundManager:
    def __init__(self):
        self.tasks = {}
        self._notification_queue = []
        self._lock = threading.Lock()
```

2. `run()`이 데몬 스레드를 시작하고 즉시 반환한다.

```python
def run(self, command: str) -> str:
    task_id = str(uuid.uuid4())[:8]
    self.tasks[task_id] = {"status": "running", "command": command}
    thread = threading.Thread(
        target=self._execute, args=(task_id, command), daemon=True)
    thread.start()
    return f"Background task {task_id} started"
```

3. 서브프로세스가 완료되면 결과가 알림 큐에 들어간다.

```python
def _execute(self, task_id, command):
    try:
        r = subprocess.run(command, shell=True, cwd=WORKDIR,
            capture_output=True, text=True, timeout=300)
        output = (r.stdout + r.stderr).strip()[:50000]
    except subprocess.TimeoutExpired:
        output = "Error: Timeout (300s)"
    with self._lock:
        self._notification_queue.append({
            "task_id": task_id, "result": output[:500]})
```

4. 에이전트 루프가 매 LLM 호출 전에 알림을 소비한다.

```python
def agent_loop(messages: list):
    while True:
        notifs = BG.drain_notifications()
        if notifs:
            notif_text = "\n".join(
                f"[bg:{n['task_id']}] {n['result']}" for n in notifs)
            messages.append({"role": "user",
                "content": f"<background-results>\n{notif_text}\n"
                           f"</background-results>"})
        response = client.messages.create(...)
```

루프는 싱글 스레드를 유지한다. 서브프로세스 I/O만 병렬화된다.

## 변경 사항

| 구성 요소      | Before (s07)     | After (s08)                |
|----------------|------------------|----------------------------|
| Tools          | 8                | 6 (base + background_run + check)|
| Execution      | Blocking only    | Blocking + background threads|
| Notification   | None             | Queue drained per loop     |
| Concurrency    | None             | Daemon threads             |

## 직접 실행해 보기

```sh
cd learn-claude-code
python agents/s08_background_tasks.py
```

1. `Run "sleep 5 && echo done" in the background, then create a file while it runs`
2. `Start 3 background tasks: "sleep 2", "sleep 4", "sleep 6". Check their status.`
3. `Run pytest in the background and keep working on other things`
