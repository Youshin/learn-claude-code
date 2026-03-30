# s10: Team Protocols

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > s08 > s09 > [ s10 ] s11 > s12`

> *"팀원에게는 공유된 소통 규칙이 필요하다"* -- 하나의 요청-응답 패턴이 모든 협상을 이끈다.
>
> **하네스(Harness) 레이어**: 프로토콜 -- 모델 간 구조화된 핸드셰이크.

## 문제

s09에서 팀원들은 작업하고 소통하지만, 구조화된 조율은 빠져 있다.

**종료**: 스레드를 강제로 죽이면 파일이 반쯤 쓰인 채로 남고 config.json이 오래된 상태가 된다. 핸드셰이크가 필요하다. 리드가 요청하고, 팀원이 승인(마무리 후 종료)하거나 거부(계속 작업)한다.

**계획 승인**: 리드가 "인증 모듈을 리팩터링해"라고 말하면 팀원이 즉시 시작한다. 고위험 변경의 경우, 리드가 먼저 계획을 검토해야 한다.

두 경우 모두 구조는 같다. 한쪽이 고유 ID를 가진 요청을 보내고, 상대방이 해당 ID를 참조하여 응답한다.

## 해결 방법

```
Shutdown Protocol            Plan Approval Protocol
==================           ======================

Lead             Teammate    Teammate           Lead
  |                 |           |                 |
  |--shutdown_req-->|           |--plan_req------>|
  | {req_id:"abc"}  |           | {req_id:"xyz"}  |
  |                 |           |                 |
  |<--shutdown_resp-|           |<--plan_resp-----|
  | {req_id:"abc",  |           | {req_id:"xyz",  |
  |  approve:true}  |           |  approve:true}  |

Shared FSM:
  [pending] --approve--> [approved]
  [pending] --reject---> [rejected]

Trackers:
  shutdown_requests = {req_id: {target, status}}
  plan_requests     = {req_id: {from, plan, status}}
```

## 작동 원리

1. 리드가 request_id를 생성하고 수신함을 통해 종료를 요청한다.

```python
shutdown_requests = {}

def handle_shutdown_request(teammate: str) -> str:
    req_id = str(uuid.uuid4())[:8]
    shutdown_requests[req_id] = {"target": teammate, "status": "pending"}
    BUS.send("lead", teammate, "Please shut down gracefully.",
             "shutdown_request", {"request_id": req_id})
    return f"Shutdown request {req_id} sent (status: pending)"
```

2. 팀원이 요청을 받고 승인 또는 거부로 응답한다.

```python
if tool_name == "shutdown_response":
    req_id = args["request_id"]
    approve = args["approve"]
    shutdown_requests[req_id]["status"] = "approved" if approve else "rejected"
    BUS.send(sender, "lead", args.get("reason", ""),
             "shutdown_response",
             {"request_id": req_id, "approve": approve})
```

3. 계획 승인도 동일한 패턴을 따른다. 팀원이 계획을 제출(request_id 생성)하고, 리드가 같은 request_id를 참조하여 검토한다.

```python
plan_requests = {}

def handle_plan_review(request_id, approve, feedback=""):
    req = plan_requests[request_id]
    req["status"] = "approved" if approve else "rejected"
    BUS.send("lead", req["from"], feedback,
             "plan_approval_response",
             {"request_id": request_id, "approve": approve})
```

하나의 FSM, 두 가지 적용. 동일한 `pending -> approved | rejected` 상태 머신이 어떤 요청-응답 프로토콜이든 처리한다.

## s09 대비 변경 사항

| 구성 요소      | 이전 (s09)       | 이후 (s10)                          |
|----------------|------------------|-------------------------------------|
| 도구           | 9개              | 12개 (+shutdown_req/resp +plan)     |
| 종료           | 자연 종료만      | 요청-응답 핸드셰이크                |
| 계획 게이팅    | 없음             | 제출/검토 및 승인                   |
| 상관 관계      | 없음             | 요청별 request_id                   |
| FSM            | 없음             | pending -> approved/rejected        |

## 직접 해보기

```sh
cd learn-claude-code
python agents/s10_team_protocols.py
```

1. `Spawn alice as a coder. Then request her shutdown.`
2. `List teammates to see alice's status after shutdown approval`
3. `Spawn bob with a risky refactoring task. Review and reject his plan.`
4. `Spawn charlie, have him submit a plan, then approve it.`
5. `/team`을 입력하면 상태를 모니터링할 수 있다
