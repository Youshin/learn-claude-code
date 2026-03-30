[English](./README.md) | [中文](./README-zh.md) | [日本語](./README-ja.md) | [한국어](./README-ko.md)

# Learn Claude Code -- 진짜 Agent를 위한 Harness Engineering

## 모델이 곧 Agent입니다

코드 이야기를 시작하기 전에, 한 가지를 분명히 하고 넘어가겠습니다.

**Agent란 모델입니다. 프레임워크(Framework)가 아닙니다. 프롬프트 체인(Prompt Chain)이 아닙니다. 드래그 앤 드롭(Drag-and-Drop) 워크플로도 아닙니다.**

### Agent란 무엇인가

Agent는 신경망(Neural Network)입니다. Transformer, RNN, 혹은 학습된 함수처럼, 수십억 번의 gradient update를 거치며 행동 시퀀스(action-sequence) 데이터 위에서 환경을 지각하고, 목표를 추론하고, 행동하는 법을 학습한 존재입니다. AI에서 "Agent"라는 말은 처음부터 이 의미였습니다. 늘 그랬습니다.

인간도 Agent입니다. 수백만 년의 진화적 훈련을 통해 형성된 생물학적 신경망입니다. 감각으로 세계를 지각하고, 뇌로 추론하며, 몸으로 행동합니다. DeepMind, OpenAI, Anthropic이 "Agent"라고 말할 때도, 그 의미는 이 분야가 처음부터 써온 의미와 같습니다. **행동하는 법을 학습한 모델**입니다.

역사가 그 증거를 보여줍니다.

- **2013 -- DeepMind DQN plays Atari.** 단일 신경망이 raw pixel과 game score만 받아 7개의 Atari 2600 게임을 학습했고, 이전의 모든 알고리즘을 뛰어넘었으며 그중 3개에서는 인간 전문가를 이겼습니다. 2015년에는 같은 아키텍처가 [49개 게임으로 확장되어 professional human tester 수준에 도달](https://www.nature.com/articles/nature14236)했고, *Nature*에 실렸습니다. 게임별 규칙도 없었습니다. decision tree도 없었습니다. 하나의 모델이 경험으로부터 학습했습니다. 그 모델이 곧 Agent였습니다.

- **2019 -- OpenAI Five conquers Dota 2.** 5개의 신경망이 10개월 동안 [45,000년 분량의 Dota 2](https://openai.com/index/openai-five-defeats-dota-2-world-champions/)를 self-play했고, 샌프란시스코 라이브 스트리밍에서 **OG** -- TI8 월드 챔피언 -- 를 2-0으로 꺾었습니다. 이후 공개 아레나에서는 42,729경기 중 99.4%를 승리했습니다. 스크립트된 전략은 없었습니다. 메타 프로그래밍된 팀 협업도 없었습니다. 모델이 온전히 self-play를 통해 팀워크, 전술, 실시간 적응을 학습했습니다.

- **2019 -- DeepMind AlphaStar masters StarCraft II.** AlphaStar는 비공개 경기에서 [프로 선수들을 10-1로 이겼고](https://deepmind.google/blog/alphastar-mastering-the-real-time-strategy-game-starcraft-ii/), 이후 유럽 서버에서 [Grandmaster 등급에 도달](https://www.nature.com/articles/d41586-019-03298-6)했습니다. 이는 90,000명 중 상위 0.15%였습니다. 불완전 정보, 실시간 판단, 체스와 바둑을 훨씬 뛰어넘는 조합적 action space를 가진 게임입니다. Agent는 무엇이었습니까? 모델이었습니다. 학습된 모델이었습니다. 스크립트가 아니었습니다.

- **2019 -- Tencent Jueyu dominates Honor of Kings.** Tencent AI Lab의 "Jueyu"는 2019년 8월 2일 World Champion Cup에서 [KPL 프로 선수를 상대로 5v5 경기에서 승리](https://www.jiemian.com/article/3371171.html)했습니다. 1v1 모드에서는 프로 선수가 [15전 중 1승만 거두었고, 8분 이상 버티지 못했습니다](https://developer.aliyun.com/article/851058). 훈련 강도는 1일이 인간 440년에 해당했습니다. 2021년까지는 전체 hero pool에서 KPL 프로를 전면적으로 앞질렀습니다. 손으로 만든 hero matchup table도 없었습니다. 스크립트된 조합도 없었습니다. self-play로 게임 전체를 처음부터 학습한 모델이었습니다.

- **2024-2025 -- LLM Agent reshapes software engineering.** Claude, GPT, Gemini -- 인류의 코드와 추론 전반을 학습한 대규모 언어 모델 -- 이 coding Agent로 배치되고 있습니다. 코드베이스를 읽고, 구현을 작성하고, 장애를 디버깅하고, 팀 단위로 협업합니다. 아키텍처는 앞선 모든 Agent와 동일합니다. 학습된 모델이 환경에 놓이고, 지각하고 행동할 수 있는 도구를 부여받습니다. 차이는 단 하나입니다. 학습 규모와 해결 가능한 작업의 범용성입니다.

이 모든 이정표가 같은 진실을 말하고 있습니다. **"Agent"는 결코 주변의 코드가 아닙니다. Agent는 언제나 모델 그 자체입니다.**

### Agent가 아닌 것

"Agent"라는 단어는 프롬프트 배관(prompt plumbing) 산업 전체에 의해 잘못 소비되고 있습니다.

드래그 앤 드롭 워크플로 빌더입니다. 노코드 "AI Agent" 플랫폼입니다. 프롬프트 체인 orchestration 라이브러리입니다. 이들은 모두 같은 환상을 공유합니다. LLM API 호출을 if-else 분기, 노드 그래프, 하드코딩된 라우팅 로직으로 엮으면 그것이 곧 "Agent를 만드는 일"이라고 믿습니다.

아닙니다. 그들이 만든 것은 루브 골드버그 머신입니다. 과도하게 설계된, 취약한 절차적 규칙 파이프라인입니다. LLM은 그 안에 포장된 텍스트 완성 노드처럼 끼워 넣어져 있을 뿐입니다. 그것은 Agent가 아닙니다. 거대한 망상을 가진 shell script에 가깝습니다.

**프롬프트 배관식 "Agent"는 모델을 훈련하지 않는 프로그래머의 환상입니다.** 절차적 로직을 계속 쌓아 올려 지능을 힘으로 구현하려고 합니다. 거대한 rule tree, node graph, chain-of-prompt waterfall을 만들고, 언젠가 충분한 glue code가 자율적 행동을 창발시켜 주기를 바랍니다. 하지만 그렇지 않습니다. Engineering만으로 agency를 코딩할 수는 없습니다. Agency는 학습되는 것이지, 프로그래밍되는 것이 아닙니다.

그런 시스템은 태어나는 순간부터 한계가 정해져 있습니다. 취약하고, 확장되지 않으며, 근본적으로 일반화가 불가능합니다. 이것은 GOFAI(Good Old-Fashioned AI), 즉 고전적 기호 AI의 현대판입니다. 수십 년 전에 학계가 버린 기호 규칙 시스템에 LLM이라는 페인트만 다시 칠한 것입니다. 포장은 다르지만, 결국 같은 막다른 길입니다.

### 마인드셋의 전환: "Agent를 개발한다"에서 Harness를 개발한다로

"Agent를 개발하고 있다"고 말할 때, 실제로 의미할 수 있는 것은 두 가지뿐입니다.

**1. 모델을 훈련시키는 것입니다.** 강화학습, fine-tuning, RLHF, 혹은 그 밖의 gradient-based 방법으로 weight를 조정하는 일입니다. task-process data -- 실제 도메인에서의 지각, 추론, 행동의 시퀀스 -- 를 수집하고, 그것으로 모델의 행동을 형성합니다. DeepMind, OpenAI, Tencent AI Lab, Anthropic이 하는 일이 바로 이것입니다. 이것이 가장 본질적인 의미의 Agent 개발입니다.

**2. Harness를 구축하는 것입니다.** 모델이 작동할 수 있는 환경을 제공하는 코드를 작성하는 일입니다. 우리 대부분이 하는 일이 이것이며, 이 저장소의 핵심도 여기에 있습니다.

Harness는 Agent가 특정 도메인에서 기능하기 위해 필요한 모든 것입니다.

```text
Harness = Tools + Knowledge + Observation + Action Interfaces + Permissions

    Tools:        파일 입출력, 쉘, 네트워크, 데이터베이스, 브라우저
    Knowledge:    제품 문서, 도메인 지식, API 명세, 스타일 가이드  
    Observation:  git diff, 에러 로그, 브라우저 상태, 센서 데이터  
    Action:       CLI 명령, API 호출, UI 상호작용  
    Permissions:  샌드박스, 승인 워크플로, 권한 경계
```

모델이 결정합니다. Harness가 실행합니다. 모델이 추론합니다. Harness가 컨텍스트를 제공합니다. 모델은 운전자입니다. Harness는 차량입니다.

**Coding Agent의 Harness는 IDE, 터미널, 파일 시스템입니다.** 농업 Agent의 Harness는 센서 시스템, 관개 제어, 기상 데이터입니다. 호텔 Agent의 Harness는 예약 시스템, 고객 커뮤니케이션 채널, 시설 관리 API입니다. Agent — 지능이자 의사결정 주체 — 는 언제나 모델입니다. Harness는 도메인마다 달라집니다. 하지만 Agent는 어떤 도메인에서도 동일한 방식으로 작동합니다.

이 저장소는 차량을 만드는 방법을 가르칩니다. 코딩을 위한 차량입니다. 하지만 이 설계 패턴은 특정 분야에만 국한되지 않습니다. 농장 관리, 호텔 운영, 제조, 물류, 의료, 교육, 과학 연구까지 모두 동일하게 적용됩니다.

어떤 작업이든, 지각하고 → 추론하고 → 행동해야 한다면 그곳에는 Agent가 작동할 Harness가 필요합니다.

### Harness 엔지니어의 역할

이 저장소를 읽고 있다면, 아마 당신은 Harness 엔지니어일 것입니다. 그리고 그것은 강력한 정체성입니다. 다음이 바로 당신의 실제 역할입니다.

- **도구를 구현합니다.** Agent에게 ‘손’을 제공합니다. 파일 읽기/쓰기, 셸 실행, API 호출, 브라우저 제어, 데이터베이스 쿼리 등입니다. 각 도구는 Agent가 환경에서 수행할 수 있는 행동입니다. 작고 명확하며, 서로 조합 가능하도록 설계해야 합니다.

- **지식을 큐레이션합니다.** Agent에게 도메인 전문성을 제공합니다. 제품 문서, 아키텍처 결정 기록, 스타일 가이드, 규제 요구사항 등이 여기에 포함됩니다. 모든 정보를 미리 넣지 말고, 필요할 때 불러오도록 해야 합니다(s05). Agent는 어떤 정보가 있는지 알고, 필요할 때 스스로 가져와야 합니다.

- **컨텍스트를 관리합니다.** Agent에게 깨끗한 작업 공간을 제공합니다. subagent isolation(s04)은 불필요한 정보가 섞이는 것을 막고, context compression(s06)은 대화가 과도하게 길어지는 것을 방지합니다. task system(s07)은 목표를 한 번의 대화에 머물지 않고 지속시킵니다.

- **권한을 제어합니다.** Agent의 행동 범위를 설정합니다. 파일 접근을 제한하고, 위험한 작업에는 승인을 요구하며, 외부 시스템과의 신뢰 경계를 명확히 합니다. 이 영역은 안전 공학과 Harness 엔지니어링이 만나는 지점입니다.

- **task-process data를 수집합니다.** Agent가 수행하는 모든 행동 과정은 곧 학습 데이터입니다. 실제 환경에서의 지각 → 추론 → 행동 기록은 다음 세대 Agent 모델을 개선하는 데 사용됩니다. 즉, Harness는 단순히 Agent를 돕는 것을 넘어 Agent 자체를 더 발전시키는 역할도 합니다.

당신은 지능을 만드는 사람이 아닙니다. 지능이 살아갈 환경을 만드는 사람입니다. -- Agent가 얼마나 잘 지각할 수 있는지, 얼마나 정확하게 행동할 수 있는지, 얼마나 풍부한 지식을 활용할 수 있는지 —- 이 모든 것은 Harness의 품질에 달려 있습니다.
**좋은 Harness를 만드십시오. 나머지는 Agent가 해냅니다.**

### 왜 Claude Code인가 -- Harness Engineering의 교과서

왜 이 저장소는 특별히 Claude Code를 해부합니까?

Claude Code는 우리가 본 것 중 가장 우아하고 완성도가 높은 Agent Harness이기 때문입니다. 무언가 영리한 트릭 하나 때문이 아닙니다. 오히려 그것이 **하지 않는 것** 때문에 중요합니다. Agent 자체가 되려 하지 않습니다. 경직된 워크플로를 강요하지 않습니다. 정교한 decision tree로 모델을 계속 의심하지 않습니다. 도구, 지식, 컨텍스트 관리, 권한 경계를 제공하고, 그다음 물러섭니다.

Claude Code를 본질만 남겨 쓰면 다음과 같습니다.

```text
Claude Code = one agent loop
            + tools (bash, read, write, edit, glob, grep, browser...)
            + on-demand skill loading
            + context compression
            + subagent spawning
            + task system with dependency graph
            + team coordination with async mailboxes
            + worktree isolation for parallel execution
            + permission governance
```

이것이 전부입니다. 이것이 전체 아키텍처입니다. 모든 구성 요소는 Harness 메커니즘입니다. Agent가 살아가는 세계의 일부입니다. Agent 그 자체는 무엇입니까? Claude입니다. 모델입니다. Anthropic이 인류의 추론과 코드 전반 위에서 훈련한 모델입니다. Harness가 Claude를 똑똑하게 만든 것이 아닙니다. Claude는 이미 똑똑합니다. Harness가 Claude에게 손과 눈, 그리고 작업 공간을 준 것입니다.

이것이 Claude Code가 이상적인 교육 대상인 이유입니다. **모델을 신뢰하고, 엔지니어링의 초점을 Harness에 둘 때 무엇이 가능한지를 보여주기 때문입니다.** 이 저장소의 각 세션(s01-s12)은 Claude Code 아키텍처에서 Harness 메커니즘 하나를 reverse engineering합니다. 끝까지 오면, Claude Code의 동작 방식뿐 아니라, 어떤 도메인의 어떤 Agent에도 적용되는 Harness 공학의 보편 원리를 이해하게 됩니다.

교훈은 "Claude Code를 복제하라"가 아닙니다. 교훈은 이것입니다. **최고의 Agent 제품은, 자신의 일이 Intelligence가 아니라 Harness라는 사실을 이해한 엔지니어가 만듭니다.**

---

## 비전: 우주를 진짜 Agent로 채우기

이것은 coding Agent만의 이야기가 아닙니다.

인간이 복잡하고, 다단계이며, 판단 집약적인 일을 수행하는 모든 도메인은 Agent가 작동할 수 있는 도메인입니다. 올바른 Harness만 있다면 가능합니다. 이 저장소의 패턴은 보편적입니다.

```text
Estate management Agent   = model + property sensors + maintenance tools + tenant comms
Agricultural Agent       = model + soil/weather data + irrigation controls + crop knowledge
Hotel operations Agent   = model + booking system + guest channels + facility APIs
Medical research Agent   = model + literature search + lab instruments + protocol docs
Manufacturing Agent      = model + production line sensors + quality controls + logistics
Education Agent          = model + curriculum knowledge + student progress + assessment tools
```

루프는 언제나 같습니다. 바뀌는 것은 도구입니다. 지식이 바뀝니다. 권한이 바뀝니다. Agent -- 모델 -- 가 모든 것을 일반화합니다.

이 저장소를 읽는 모든 Harness 엔지니어는 소프트웨어 엔지니어링을 훨씬 넘어서는 패턴을 배우고 있습니다. 당신은 지능적이고 자동화된 미래를 위한 인프라를 구축하는 법을 배우고 있습니다. 실제 도메인에 배포된 훌륭한 Harness 하나하나가, Agent가 지각하고, 추론하고, 행동할 수 있는 새로운 거점이 됩니다.

먼저 작업장을 채웁니다. 그다음 농장과 병원과 공장입니다. 그다음 도시입니다. 그다음 행성입니다.

**Bash is all you need. Real agents are all the universe needs.**

---

```text
                    THE AGENT PATTERN
                    =================

    User --> messages[] --> LLM --> response
                                      |
                            stop_reason == "tool_use"?
                           /                          \
                         yes                           no
                          |                             |
                    execute tools                    return text
                    append results
                    loop back -----------------> messages[]


    이것이 최소 루프입니다.
    모든 AI Agent에는 이 루프가 필요합니다.
    모델이 tool 호출과 중단 시점을 결정합니다.
    코드는 모델의 요청을 실행할 뿐입니다.
    이 저장소는 이 루프를 둘러싼 모든 것,
    곧 특정 도메인에서 Agent를 효과적으로 만드는 Harness를
    어떻게 구축하는지 가르칩니다.
```

**12개의 점진적 세션이 있습니다. 단순한 loop에서 분리된 autonomous execution까지 이어집니다.**  
**각 세션은 하나의 Harness 메커니즘을 추가합니다. 각 메커니즘에는 하나의 motto가 있습니다.**

> **s01** &nbsp; *"One loop & Bash is all you need"* &mdash; 하나의 tool + 하나의 loop = Agent
>
> **s02** &nbsp; *"Adding a tool means adding one handler"* &mdash; loop는 바뀌지 않습니다. 새 tool은 dispatch map에 등록하기만 하면 됩니다
>
> **s03** &nbsp; *"An agent without a plan drifts"* &mdash; 먼저 단계를 적고, 그다음 실행합니다
>
> **s04** &nbsp; *"Break big tasks down; each subtask gets a clean context"* &mdash; subagent는 독립된 messages[]를 사용하여 메인 대화를 오염시키지 않습니다
>
> **s05** &nbsp; *"Load knowledge when you need it, not upfront"* &mdash; system prompt가 아니라 tool_result로 주입합니다
>
> **s06** &nbsp; *"Context will fill up; you need a way to make room"* &mdash; 3계층 압축으로 무한 세션을 구현합니다
>
> **s07** &nbsp; *"Break big goals into small tasks, order them, persist to disk"* &mdash; file-based task graph가 multi-agent collaboration의 기반이 됩니다
>
> **s08** &nbsp; *"Run slow operations in the background; the agent keeps thinking"* &mdash; daemon thread가 command를 실행하고, 완료 후 notification을 주입합니다
>
> **s09** &nbsp; *"When the task is too big for one, delegate to teammates"* &mdash; persistent teammate + async mailbox
>
> **s10** &nbsp; *"Teammates need shared communication rules"* &mdash; 하나의 request-response 패턴이 모든 협상을 구동합니다
>
> **s11** &nbsp; *"Teammates scan the board and claim tasks themselves"* &mdash; 리더가 일일이 task를 배정할 필요가 없습니다
>
> **s12** &nbsp; *"Each works in its own directory, no interference"* &mdash; task는 목표를 관리하고, worktree는 디렉터리를 관리하며, 둘은 ID로 연결됩니다

---

## 코어 패턴

```python
def agent_loop(messages):
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM,
            messages=messages, tools=TOOLS,
        )
        messages.append({"role": "assistant",
                         "content": response.content})

        if response.stop_reason != "tool_use":
            return

        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = TOOL_HANDLERS[block.name](**block.input)
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": output,
                })
        messages.append({"role": "user", "content": results})
```

각 세션은 이 루프 위에 Harness 메커니즘 하나를 겹쳐 올립니다. 루프 자체는 바뀌지 않습니다. 루프는 Agent의 것입니다. 메커니즘은 Harness의 것입니다.

## 범위 (중요)

이 저장소는 Harness Engineering의 0->1 학습 프로젝트입니다. Agent 모델을 둘러싼 환경을 구축하는 법을 배우기 위한 것입니다. 학습을 우선하기 위해, 다음과 같은 production 메커니즘은 의도적으로 단순화하거나 생략했습니다.

- 완전한 event / hook bus입니다. 예를 들면 PreToolUse, SessionStart/End, ConfigChange가 있습니다.  
  s12에는 교육용으로 최소한의 append-only lifecycle event stream만 구현되어 있습니다.
- rule-based permission governance와 trust workflow입니다.
- session lifecycle control(resume/fork)과 고급 worktree lifecycle control입니다.
- MCP runtime의 세부사항입니다. 예를 들면 transport, OAuth, resource subscribe, polling이 있습니다.

이 저장소의 team JSONL mailbox 방식은 교육용 구현입니다. 특정 production 내부 구현을 주장하는 것이 아닙니다.

## Quick Start

```sh
git clone https://github.com/shareAI-lab/learn-claude-code
cd learn-claude-code
pip install -r requirements.txt
cp .env.example .env   # .env를 열고 ANTHROPIC_API_KEY를 입력합니다

python agents/s01_agent_loop.py       # 여기서 시작합니다
python agents/s12_worktree_task_isolation.py  # 전체 세션의 도착점입니다
python agents/s_full.py               # 총정리: 모든 메커니즘 통합 버전입니다
```

### Web Platform

interactive visualization, step-through animation, source viewer, 각 세션 문서를 제공합니다.

```sh
cd web && npm install && npm run dev   # http://localhost:3000
```

## 학습 경로

```text
Phase 1: THE LOOP                    Phase 2: PLANNING & KNOWLEDGE
==================                   ==============================
s01  The Agent Loop          [1]     s03  TodoWrite               [5]
     while + stop_reason                  TodoManager + nag reminder
     |                                    |
     +-> s02  Tool Use            [4]     s04  Subagents            [5]
              dispatch map: name->handler     fresh messages[] per child
                                              |
                                         s05  Skills               [5]
                                              SKILL.md via tool_result
                                              |
                                         s06  Context Compact      [5]
                                              3-layer compression

Phase 3: PERSISTENCE                 Phase 4: TEAMS
==================                   =====================
s07  Tasks                   [8]     s09  Agent Teams             [9]
     file-based CRUD + deps graph         teammates + JSONL mailboxes
     |                                    |
s08  Background Tasks        [6]     s10  Team Protocols          [12]
     daemon threads + notify queue        shutdown + plan approval FSM
                                          |
                                     s11  Autonomous Agents       [14]
                                          idle cycle + auto-claim
                                     |
                                     s12  Worktree Isolation      [16]
                                          task coordination + optional isolated execution lanes

                                     [N] = number of tools
```

## 프로젝트 구조

```text
learn-claude-code/
|
|-- agents/                        # Python reference implementation (s01-s12 + s_full capstone)
|-- docs/{en,zh,ja}/               # mental-model-first documentation (3 languages)
|-- web/                           # interactive learning platform (Next.js)
|-- skills/                        # Skill files for s05
+-- .github/workflows/ci.yml       # CI: typecheck + build
```

## 문서

mental-model-first 방식입니다. 문제, 해결책, ASCII 다이어그램, 최소 코드 중심입니다.  
[English](./docs/en/) | [中文](./docs/zh/) | [日本語](./docs/ja/)

| 세션 | 주제 | 모토 |
|------|------|------|
| [s01](./docs/en/s01-the-agent-loop.md) | The Agent Loop | *One loop & Bash is all you need* |
| [s02](./docs/en/s02-tool-use.md) | Tool Use | *Adding a tool means adding one handler* |
| [s03](./docs/en/s03-todo-write.md) | TodoWrite | *An agent without a plan drifts* |
| [s04](./docs/en/s04-subagent.md) | Subagents | *Break big tasks down; each subtask gets a clean context* |
| [s05](./docs/en/s05-skill-loading.md) | Skills | *Load knowledge when you need it, not upfront* |
| [s06](./docs/en/s06-context-compact.md) | Context Compact | *Context will fill up; you need a way to make room* |
| [s07](./docs/en/s07-task-system.md) | Tasks | *Break big goals into small tasks, order them, persist to disk* |
| [s08](./docs/en/s08-background-tasks.md) | Background Tasks | *Run slow operations in the background; the agent keeps thinking* |
| [s09](./docs/en/s09-agent-teams.md) | Agent Teams | *When the task is too big for one, delegate to teammates* |
| [s10](./docs/en/s10-team-protocols.md) | Team Protocols | *Teammates need shared communication rules* |
| [s11](./docs/en/s11-autonomous-agents.md) | Autonomous Agents | *Teammates scan the board and claim tasks themselves* |
| [s12](./docs/en/s12-worktree-task-isolation.md) | Worktree + Task Isolation | *Each works in its own directory, no interference* |

## 다음 단계 -- 이해에서 배포까지

12개 세션을 마치면 Harness Engineering의 내부 구조를 깊이 이해하게 됩니다. 그 지식을 실제로 활용하는 방법은 두 가지입니다.

### Kode Agent CLI -- Open-Source Coding Agent CLI

> `npm i -g @shareai-lab/kode`

Skill & LSP를 지원하고, Windows-ready이며, GLM / MiniMax / DeepSeek 등 open model에 연결할 수 있습니다. 설치 후 바로 사용할 수 있습니다.

GitHub: **[shareAI-lab/Kode-cli](https://github.com/shareAI-lab/Kode-cli)**

### Kode Agent SDK -- 앱에 Agent 기능을 내장하기

공식 Claude Code Agent SDK는 내부적으로 완전한 CLI process와 통신합니다. 즉, concurrent user마다 별도의 terminal process가 필요합니다. Kode SDK는 독립 라이브러리이며 사용자별 process overhead가 없습니다. backend, browser extension, embedded device 등에도 내장할 수 있습니다.

GitHub: **[shareAI-lab/Kode-agent-sdk](https://github.com/shareAI-lab/Kode-agent-sdk)**

---

## Sister Repo: *on-demand session*에서 *always-on assistant*로

이 저장소가 가르치는 Harness는 **use-and-discard** 방식입니다. terminal을 열고, Agent에게 task를 주고, 끝나면 닫습니다. 다음 session은 빈 상태에서 다시 시작합니다. 이것이 Claude Code 모델입니다.

[OpenClaw](https://github.com/openclaw/openclaw)는 다른 가능성을 보여주었습니다. 같은 agent core 위에 두 가지 Harness 메커니즘만 더하면, Agent는 "불러야 움직이는 도구"에서 "30초마다 스스로 깨어나 일을 찾는 존재"로 바뀝니다.

- **Heartbeat** -- 30초마다 Harness가 Agent에게 메시지를 보내, 할 일이 있는지 확인합니다. 없으면 다시 잠들고, 있으면 즉시 행동합니다.
- **Cron** -- Agent가 스스로 미래 작업을 예약하고, 시간이 되면 자동으로 실행합니다.

여기에 multi-channel IM routing(WhatsApp / Telegram / Slack / Discord 등 13+ 플랫폼), persistent context memory, Soul personality system을 더하면, Agent는 일회성 도구에서 항상 켜져 있는 personal AI assistant로 변합니다.

**[claw0](https://github.com/shareAI-lab/claw0)** 는 이러한 Harness 메커니즘을 처음부터 분해해 설명하는 companion teaching repo입니다.

```text
claw agent = agent core + heartbeat + cron + IM chat + memory + soul
```

```text
learn-claude-code                   claw0
(agent harness core:                (proactive always-on harness:
 loop, tools, planning,              heartbeat, cron, IM channels,
 teams, worktree isolation)          memory, soul personality)
```

## 라이선스

MIT

---

**모델이 곧 Agent입니다. 코드는 Harness입니다. 좋은 Harness를 만드십시오. 나머지는 Agent가 해냅니다.**

**Bash is all you need. Real agents are all the universe needs.**
