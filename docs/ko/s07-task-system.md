# s07: Task System

`s01 > s02 > s03 > s04 > s05 > s06 | [ s07 ] s08 > s09 > s10 > s11 > s12`

> *"Break big goals into small tasks, order them, persist to disk"* -- 의존 관계가 있는 파일 기반 태스크 그래프로, 멀티 에이전트 협업의 토대를 마련한다.
>
> **하네스(Harness) 계층**: 영속 태스크 -- 단일 대화의 수명을 넘어 유지되는 목표.

## 문제

s03의 TodoManager는 메모리 위의 단순한 체크리스트에 불과하다: 순서도 없고, 의존 관계도 없고, 상태는 완료 아니면 미완료뿐이다. 실제 목표에는 구조가 있다 -- 태스크 B는 태스크 A에 의존하고, 태스크 C와 D는 병렬 실행이 가능하며, 태스크 E는 C와 D 모두가 끝나기를 기다린다.

명시적인 관계 없이는 에이전트가 무엇이 실행 가능하고, 무엇이 블로킹되어 있고, 무엇을 동시에 돌릴 수 있는지 판단할 수 없다. 게다가 리스트가 메모리에만 존재하므로 컨텍스트 압축(s06)이 일어나면 통째로 사라진다.

## 해결 방법

단순한 체크리스트를 디스크에 영속화되는 **태스크 그래프**로 승격시킨다. 각 태스크는 상태와 의존 관계(`blockedBy`)를 담은 하나의 JSON 파일이다. 태스크 그래프는 항상 세 가지 질문에 답한다:

- **실행 가능한 것은?** -- `pending` 상태이면서 `blockedBy`가 비어 있는 태스크.
- **블로킹된 것은?** -- 미완료 의존을 기다리는 태스크.
- **완료된 것은?** -- `completed` 태스크. 완료 시 후속 태스크를 자동으로 언블로킹한다.

```
.tasks/
  task_1.json  {"id":1, "status":"completed"}
  task_2.json  {"id":2, "blockedBy":[1], "status":"pending"}
  task_3.json  {"id":3, "blockedBy":[1], "status":"pending"}
  task_4.json  {"id":4, "blockedBy":[2,3], "status":"pending"}

태스크 그래프 (DAG):
                 +----------+
            +--> | task 2   | --+
            |    | pending  |   |
+----------+     +----------+    +--> +----------+
| task 1   |                          | task 4   |
| completed| --> +----------+    +--> | blocked  |
+----------+     | task 3   | --+     +----------+
                 | pending  |
                 +----------+

순서:       task 1이 2와 3보다 먼저 완료되어야 한다
병렬:       task 2와 3은 동시에 실행할 수 있다
의존:       task 4는 2와 3 모두를 기다린다
상태:       pending -> in_progress -> completed
```

이 태스크 그래프는 s07 이후 모든 메커니즘의 협업 백본이 된다: 백그라운드 실행(s08), 멀티 에이전트 팀(s09+), worktree 격리(s12) 모두 이 동일한 구조를 읽고 쓴다.

## 동작 방식

1. **TaskManager**: 태스크당 하나의 JSON 파일, 의존 그래프를 포함한 CRUD.

```python
class TaskManager:
    def __init__(self, tasks_dir: Path):
        self.dir = tasks_dir
        self.dir.mkdir(exist_ok=True)
        self._next_id = self._max_id() + 1

    def create(self, subject, description=""):
        task = {"id": self._next_id, "subject": subject,
                "status": "pending", "blockedBy": [],
                "owner": ""}
        self._save(task)
        self._next_id += 1
        return json.dumps(task, indent=2)
```

2. **의존 해제**: 태스크가 완료되면 다른 모든 태스크의 `blockedBy` 리스트에서 해당 ID를 제거하여 후속 태스크를 자동으로 언블로킹한다.

```python
def _clear_dependency(self, completed_id):
    for f in self.dir.glob("task_*.json"):
        task = json.loads(f.read_text())
        if completed_id in task.get("blockedBy", []):
            task["blockedBy"].remove(completed_id)
            self._save(task)
```

3. **상태 전이 + 의존 관계 연결**: `update`가 상태 변경과 의존 에지를 처리한다.

```python
def update(self, task_id, status=None,
           add_blocked_by=None, remove_blocked_by=None):
    task = self._load(task_id)
    if status:
        task["status"] = status
        if status == "completed":
            self._clear_dependency(task_id)
    if add_blocked_by:
        task["blockedBy"] = list(set(task["blockedBy"] + add_blocked_by))
    if remove_blocked_by:
        task["blockedBy"] = [x for x in task["blockedBy"] if x not in remove_blocked_by]
    self._save(task)
```

4. 네 개의 태스크 도구를 디스패치 맵에 등록한다.

```python
TOOL_HANDLERS = {
    # ...base tools...
    "task_create": lambda **kw: TASKS.create(kw["subject"]),
    "task_update": lambda **kw: TASKS.update(kw["task_id"], kw.get("status")),
    "task_list":   lambda **kw: TASKS.list_all(),
    "task_get":    lambda **kw: TASKS.get(kw["task_id"]),
}
```

s07부터 태스크 그래프가 멀티스텝 작업의 기본이 된다. s03의 Todo는 가벼운 단일 세션용 체크리스트로 남는다.

## 변경 사항

| 구성 요소 | Before (s06) | After (s07) |
|---|---|---|
| Tools | 5 | 8 (`task_create/update/list/get`) |
| 계획 모델 | 플랫 체크리스트 (메모리) | 의존 관계 태스크 그래프 (디스크) |
| 관계 | 없음 | `blockedBy` 에지 |
| 상태 추적 | 완료 또는 미완료 | `pending` -> `in_progress` -> `completed` |
| 영속성 | 압축 시 소멸 | 압축 및 재시작 후에도 유지 |

## 직접 실행해 보기

```sh
cd learn-claude-code
python agents/s07_task_system.py
```

1. `Create 3 tasks: "Setup project", "Write code", "Write tests". Make them depend on each other in order.`
2. `List all tasks and show the dependency graph`
3. `Complete task 1 and then list tasks to see task 2 unblocked`
4. `Create a task board for refactoring: parse -> transform -> emit -> test, where transform and emit can run in parallel after parse`
