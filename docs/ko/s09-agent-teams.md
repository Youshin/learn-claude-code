# s09: Agent Teams

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > s08 > [ s09 ] s10 > s11 > s12`

> *"혼자 감당하기엔 너무 크면, 팀원에게 위임하라"* -- 영속적인 팀원 + 비동기 메일박스.
>
> **하네스(Harness) 레이어**: 팀 메일박스 -- 여러 모델을 파일로 조율한다.

## 문제

서브에이전트(s04)는 일회용이다. 생성하고, 작업하고, 요약을 돌려주고, 사라진다. 정체성도 없고, 호출 사이의 기억도 없다. 백그라운드 태스크(s08)는 셸 명령을 실행하지만, LLM 기반의 판단은 내릴 수 없다.

진정한 팀워크에는 세 가지가 필요하다: (1) 단일 프롬프트를 넘어 지속되는 영속 에이전트, (2) 정체성과 생명주기 관리, (3) 에이전트 간 통신 채널.

## 해결 방법

```
Teammate lifecycle:
  spawn -> WORKING -> IDLE -> WORKING -> ... -> SHUTDOWN

Communication:
  .team/
    config.json           <- team roster + statuses
    inbox/
      alice.jsonl         <- append-only, drain-on-read
      bob.jsonl
      lead.jsonl

              +--------+    send("alice","bob","...")    +--------+
              | alice  | -----------------------------> |  bob   |
              | loop   |    bob.jsonl << {json_line}    |  loop  |
              +--------+                                +--------+
                   ^                                         |
                   |        BUS.read_inbox("alice")          |
                   +---- alice.jsonl -> read + drain ---------+
```

## 작동 원리

1. TeammateManager는 config.json으로 팀 명단을 관리한다.

```python
class TeammateManager:
    def __init__(self, team_dir: Path):
        self.dir = team_dir
        self.dir.mkdir(exist_ok=True)
        self.config_path = self.dir / "config.json"
        self.config = self._load_config()
        self.threads = {}
```

2. `spawn()`은 팀원을 생성하고, 에이전트 루프를 스레드로 시작한다.

```python
def spawn(self, name: str, role: str, prompt: str) -> str:
    member = {"name": name, "role": role, "status": "working"}
    self.config["members"].append(member)
    self._save_config()
    thread = threading.Thread(
        target=self._teammate_loop,
        args=(name, role, prompt), daemon=True)
    thread.start()
    return f"Spawned teammate '{name}' (role: {role})"
```

3. MessageBus: append-only JSONL 수신함. `send()`는 JSON 줄을 추가하고, `read_inbox()`는 전부 읽은 뒤 비운다.

```python
class MessageBus:
    def send(self, sender, to, content, msg_type="message", extra=None):
        msg = {"type": msg_type, "from": sender,
               "content": content, "timestamp": time.time()}
        if extra:
            msg.update(extra)
        with open(self.dir / f"{to}.jsonl", "a") as f:
            f.write(json.dumps(msg) + "\n")

    def read_inbox(self, name):
        path = self.dir / f"{name}.jsonl"
        if not path.exists(): return "[]"
        msgs = [json.loads(l) for l in path.read_text().strip().splitlines() if l]
        path.write_text("")  # drain
        return json.dumps(msgs, indent=2)
```

4. 각 팀원은 LLM 호출 전에 수신함을 확인하고, 받은 메시지를 컨텍스트에 주입한다.

```python
def _teammate_loop(self, name, role, prompt):
    messages = [{"role": "user", "content": prompt}]
    for _ in range(50):
        inbox = BUS.read_inbox(name)
        if inbox != "[]":
            messages.append({"role": "user",
                "content": f"<inbox>{inbox}</inbox>"})
        response = client.messages.create(...)
        if response.stop_reason != "tool_use":
            break
        # 도구 실행, 결과 추가...
    self._find_member(name)["status"] = "idle"
```

## s08 대비 변경 사항

| 구성 요소      | 이전 (s08)       | 이후 (s09)                   |
|----------------|------------------|------------------------------|
| 도구           | 6개              | 9개 (+spawn/send/read_inbox) |
| 에이전트       | 단일             | 리드 + N명의 팀원            |
| 영속성         | 없음             | config.json + JSONL 수신함   |
| 스레드         | 백그라운드 명령   | 스레드별 완전한 에이전트 루프 |
| 생명주기       | 실행 후 잊기     | idle -> working -> idle      |
| 통신           | 없음             | 메시지 + 브로드캐스트        |

## 직접 해보기

```sh
cd learn-claude-code
python agents/s09_agent_teams.py
```

1. `Spawn alice (coder) and bob (tester). Have alice send bob a message.`
2. `Broadcast "status update: phase 1 complete" to all teammates`
3. `Check the lead inbox for any messages`
4. `/team`을 입력하면 팀 명단과 상태를 확인할 수 있다
5. `/inbox`을 입력하면 리드의 수신함을 직접 확인할 수 있다
