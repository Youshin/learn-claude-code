# s12: Worktree + Task Isolation

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > [ s12 ]`

> *"각자 자기 디렉토리에서 작업하고, 서로 간섭하지 않는다"* -- 태스크는 목표를 관리하고, 워크트리는 디렉토리를 관리하며, 태스크 ID로 연결한다.
>
> **하네스(Harness) 레이어**: 디렉토리 격리 -- 절대 충돌하지 않는 병렬 실행 레인.

## 문제

s11까지 에이전트는 태스크를 자율적으로 선점하고 완료할 수 있게 되었다. 하지만 모든 태스크가 하나의 공유 디렉토리에서 실행된다. 두 에이전트가 서로 다른 모듈을 동시에 리팩터링하면 충돌이 발생한다. 에이전트 A가 `config.py`를 수정하고, 에이전트 B도 `config.py`를 수정하면, 스테이징되지 않은 변경 사항이 뒤섞이고 어느 쪽도 깔끔하게 롤백할 수 없다.

태스크 보드는 *무엇을 할지*를 추적하지만, *어디에서 할지*에는 관여하지 않는다. 해결책: 각 태스크에 고유한 Git 워크트리 디렉토리를 부여한다. 태스크가 목표를 관리하고, 워크트리가 실행 컨텍스트를 관리한다. 태스크 ID로 둘을 연결한다.

## 해결 방법

```
Control plane (.tasks/)             Execution plane (.worktrees/)
+------------------+                +------------------------+
| task_1.json      |                | auth-refactor/         |
|   status: in_progress  <------>   branch: wt/auth-refactor
|   worktree: "auth-refactor"   |   task_id: 1             |
+------------------+                +------------------------+
| task_2.json      |                | ui-login/              |
|   status: pending    <------>     branch: wt/ui-login
|   worktree: "ui-login"       |   task_id: 2             |
+------------------+                +------------------------+
                                    |
                          index.json (worktree registry)
                          events.jsonl (lifecycle log)

State machines:
  Task:     pending -> in_progress -> completed
  Worktree: absent  -> active      -> removed | kept
```

## 작동 원리

1. **태스크를 생성한다.** 먼저 목표를 영속화한다.

```python
TASKS.create("Implement auth refactor")
# -> .tasks/task_1.json  status=pending  worktree=""
```

2. **워크트리를 생성하고 태스크에 바인딩한다.** `task_id`를 전달하면 태스크가 자동으로 `in_progress`로 전환된다.

```python
WORKTREES.create("auth-refactor", task_id=1)
# -> git worktree add -b wt/auth-refactor .worktrees/auth-refactor HEAD
# -> index.json에 새 항목 추가, task_1.json에 worktree="auth-refactor" 설정
```

바인딩은 양쪽에 상태를 기록한다:

```python
def bind_worktree(self, task_id, worktree):
    task = self._load(task_id)
    task["worktree"] = worktree
    if task["status"] == "pending":
        task["status"] = "in_progress"
    self._save(task)
```

3. **워크트리 안에서 명령을 실행한다.** `cwd`가 격리된 디렉토리를 가리킨다.

```python
subprocess.run(command, shell=True, cwd=worktree_path,
               capture_output=True, text=True, timeout=300)
```

4. **마무리한다.** 두 가지 선택지가 있다:
   - `worktree_keep(name)` -- 나중을 위해 디렉토리를 보존한다.
   - `worktree_remove(name, complete_task=True)` -- 디렉토리를 제거하고, 바인딩된 태스크를 완료하고, 이벤트를 발행한다. 한 번의 호출로 정리와 완료를 모두 처리한다.

```python
def remove(self, name, force=False, complete_task=False):
    self._run_git(["worktree", "remove", wt["path"]])
    if complete_task and wt.get("task_id") is not None:
        self.tasks.update(wt["task_id"], status="completed")
        self.tasks.unbind_worktree(wt["task_id"])
        self.events.emit("task.completed", ...)
```

5. **이벤트 스트림.** 모든 생명주기 단계가 `.worktrees/events.jsonl`에 기록된다:

```json
{
  "event": "worktree.remove.after",
  "task": {"id": 1, "status": "completed"},
  "worktree": {"name": "auth-refactor", "status": "removed"},
  "ts": 1730000000
}
```

발행되는 이벤트: `worktree.create.before/after/failed`, `worktree.remove.before/after/failed`, `worktree.keep`, `task.completed`.

장애 발생 후에는 디스크의 `.tasks/` + `.worktrees/index.json`으로부터 상태를 재구성한다. 대화 메모리는 휘발성이지만, 파일 상태는 영속적이다.

## s11 대비 변경 사항

| 구성 요소            | 이전 (s11)                 | 이후 (s12)                                    |
|----------------------|----------------------------|-----------------------------------------------|
| 조율                 | 태스크 보드 (소유자/상태)  | 태스크 보드 + 명시적 워크트리 바인딩          |
| 실행 범위            | 공유 디렉토리              | 태스크별 격리된 디렉토리                      |
| 복구 가능성          | 태스크 상태만              | 태스크 상태 + 워크트리 인덱스                 |
| 정리                 | 태스크 완료                | 태스크 완료 + 명시적 keep/remove              |
| 생명주기 가시성      | 로그에 암묵적으로 포함     | `.worktrees/events.jsonl`에 명시적 이벤트     |

## 직접 해보기

```sh
cd learn-claude-code
python agents/s12_worktree_task_isolation.py
```

1. `Create tasks for backend auth and frontend login page, then list tasks.`
2. `Create worktree "auth-refactor" for task 1, then bind task 2 to a new worktree "ui-login".`
3. `Run "git status --short" in worktree "auth-refactor".`
4. `Keep worktree "ui-login", then list worktrees and inspect events.`
5. `Remove worktree "auth-refactor" with complete_task=true, then list tasks/worktrees/events.`
