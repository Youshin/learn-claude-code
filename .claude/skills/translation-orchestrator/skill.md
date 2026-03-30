---
name: translation-orchestrator
description: "번역 에이전트 팀을 조율하는 오케스트레이터. 문서 번역 프로젝트를 실행할 때 이 스킬을 사용한다. '이 문서들을 번역해줘', '한국어 번역 만들어줘', '다국어 번역', 'translate these docs' 등 여러 파일의 번역 작업을 요청하면 반드시 이 스킬을 사용할 것. 단일 파일의 간단한 번역에는 translate-doc 스킬을 직접 사용."
---

# Translation Orchestrator

번역 에이전트 팀을 구성하여 문서 번역 프로젝트를 실행하는 오케스트레이터.

## 실행 모드: 에이전트 팀

## 에이전트 구성

| 팀원 | 에이전트 타입 | 역할 | 스킬 | 출력 |
|------|-------------|------|------|------|
| translator-1 | translator (커스텀) | 문서 번역 (배치 A) | translate-doc | 번역된 파일들 |
| translator-2 | translator (커스텀) | 문서 번역 (배치 B) | translate-doc | 번역된 파일들 |
| reviewer | translation-reviewer (커스텀) | 번역 품질 검수 | review-translation | 리뷰 리포트 |

> translator 수는 번역 대상 파일 수에 따라 조정한다 (5개 이하: 1명, 6~12개: 2명, 13개 이상: 3명).

## 워크플로우

### Phase 1: 준비

1. 사용자 입력 분석:
   - 번역 대상 파일/디렉토리 확인
   - 대상 언어 확인
   - 기존 번역 패턴 파악 (파일명 컨벤션, 디렉토리 구조)

2. 작업 디렉토리에 `_workspace/` 생성

3. 파일 목록과 배치 분배 계획을 `_workspace/00_plan.md`에 저장:
   ```markdown
   ## 번역 계획
   - 대상 언어: {lang}
   - 총 파일 수: {N}
   - translator 수: {M}

   ### 배치 분배
   - translator-1: file1.md, file2.md, ...
   - translator-2: file3.md, file4.md, ...

   ### 출력 경로
   - 패턴: {기존 프로젝트 패턴 또는 기본 패턴}
   ```

4. 용어집 준비:
   - 프로젝트에 기존 번역이 있으면 용어 패턴을 추출하여 `_workspace/00_terminology.md`에 저장
   - `translate-doc/references/terminology-guide.md`가 있으면 참조 지시에 포함

### Phase 2: 팀 구성

1. 팀 생성:
   ```
   TeamCreate(
     team_name: "translation-team",
     members: [
       {
         name: "translator-1",
         agent_type: "translator",
         model: "opus",
         prompt: "당신은 translator 에이전트입니다. 다음 파일들을 {대상 언어}로 번역하세요.
           배치: {파일 목록}
           출력 경로: {경로 패턴}
           용어집: {용어집 경로 또는 인라인}
           번역 완료 후 reviewer에게 SendMessage로 알려주세요.
           Skill 도구로 /translate-doc을 호출하여 번역 가이드를 참조하세요."
       },
       {
         name: "translator-2",
         agent_type: "translator",
         model: "opus",
         prompt: "(위와 동일 형식, 다른 배치)"
       },
       {
         name: "reviewer",
         agent_type: "translation-reviewer",
         model: "opus",
         prompt: "당신은 translation-reviewer 에이전트입니다.
           translator들이 번역을 완료하면 SendMessage로 알림을 받습니다.
           원본과 번역본을 대조하여 품질을 검수하세요.
           용어집: {용어집 경로}
           FIX 판정 시 해당 translator에게 SendMessage로 구체적 수정 사항을 전달하세요.
           PASS 판정 시 리더에게 알려주세요.
           Skill 도구로 /review-translation을 호출하여 검수 가이드를 참조하세요.
           리뷰 리포트를 _workspace/review_{filename}.md에 저장하세요."
       }
     ]
   )
   ```

2. 작업 등록:
   ```
   TaskCreate(tasks: [
     { title: "번역: {file1}", description: "원본: {path}, 출력: {path}", assignee: "translator-1" },
     { title: "번역: {file2}", description: "원본: {path}, 출력: {path}", assignee: "translator-1" },
     { title: "번역: {file3}", description: "원본: {path}, 출력: {path}", assignee: "translator-2" },
     { title: "리뷰: {file1}", description: "원본: {path}, 번역: {path}", assignee: "reviewer", depends_on: ["번역: {file1}"] },
     { title: "리뷰: {file2}", description: "원본: {path}, 번역: {path}", assignee: "reviewer", depends_on: ["번역: {file2}"] },
     { title: "리뷰: {file3}", description: "원본: {path}, 번역: {path}", assignee: "reviewer", depends_on: ["번역: {file3}"] }
   ])
   ```

### Phase 3: 번역 + 검수

**실행 방식:** 팀원들이 자체 조율

1. translator들이 배치의 파일을 순차적으로 번역
2. 각 파일 번역 완료 시 reviewer에게 SendMessage로 알림
3. reviewer가 번역 완료된 파일부터 검수 시작 (점진적 검수)
4. FIX 판정 시:
   - reviewer가 해당 translator에게 SendMessage로 수정 사항 전달
   - translator가 수정 후 다시 알림
   - reviewer가 재검수 (최대 2회)
5. PASS 판정 시 리더에게 보고

**팀원 간 통신 규칙:**
- translator → reviewer: 번역 완료 알림 (파일 경로 포함)
- reviewer → translator: 수정 요청 (구체적 수정 사항 포함)
- translator ↔ translator: 용어 선택 토론 (새 용어 발견 시)
- 모든 팀원 → 리더: 완료/에러 보고

**산출물 저장:**

| 팀원 | 출력 경로 |
|------|----------|
| translator-1, 2 | 프로젝트의 번역 파일 경로 (기존 패턴 따름) |
| reviewer | `_workspace/review_{filename}.md` |

### Phase 4: 통합 및 보고

1. 모든 작업 완료 대기 (TaskGet으로 상태 확인)
2. reviewer의 리뷰 리포트를 Read로 수집
3. 최종 요약 생성:
   ```markdown
   ## 번역 완료 보고
   - 대상 언어: {lang}
   - 총 파일: {N}개
   - PASS: {N}개, FIX 후 PASS: {N}개
   - 번역된 파일 목록:
     - {path1}
     - {path2}
   ```
4. 사용자에게 결과 요약 보고

### Phase 5: 정리

1. 팀원들에게 종료 요청 (SendMessage)
2. 팀 정리 (TeamDelete)
3. `_workspace/` 보존 (리뷰 리포트 등 사후 참조용)

## 데이터 흐름

```
[리더] → TeamCreate → [translator-1] ──SendMessage──→ [reviewer]
                       [translator-2] ──SendMessage──→ [reviewer]
                                                          │
                          ←── FIX시 수정 요청 SendMessage ──┘
                          │
                          ↓ (수정 후 재알림)
                       [reviewer] → PASS → 리더에게 보고

[리더] ← Read(리뷰 리포트들) ← 최종 요약 생성
```

## 에러 핸들링

| 상황 | 전략 |
|------|------|
| translator 1명 실패 | 리더가 감지 → 해당 배치를 다른 translator에게 재할당 |
| reviewer 실패 | 리더가 직접 간이 검수 수행 (구조 검증 위주) |
| 2회 FIX 후에도 같은 문제 | reviewer가 리더에게 보고, 리더가 판단하여 PASS 또는 사용자에게 문의 |
| 용어 불일치 발견 | reviewer가 모든 translator에게 브로드캐스트하여 통일 |
| 원본 파일 읽기 실패 | 해당 파일 건너뛰고 보고서에 명시 |

## 테스트 시나리오

### 정상 흐름
1. 사용자가 "docs/en/ 의 12개 파일을 한국어로 번역해줘"라고 요청
2. Phase 1에서 12개 파일 분석, 2명 translator로 6개씩 배치
3. Phase 2에서 팀 구성 (translator-1, translator-2, reviewer)
4. Phase 3에서 translator들이 병렬 번역, reviewer가 점진적 검수
5. Phase 4에서 12개 PASS 확인, 최종 보고서 생성
6. 예상 결과: `docs/ko/` 하위에 12개 한국어 문서 생성

### 에러 흐름
1. Phase 3에서 translator-2가 3번째 파일에서 에러로 중지
2. 리더가 유휴 알림 수신
3. translator-2의 남은 배치(3개 파일)를 translator-1에게 재할당
4. reviewer가 이미 검수한 파일은 유지
5. translator-1이 추가 배치 완료 후 정상 진행
6. 최종 보고서에 "translator-2 장애로 배치 재할당됨" 명시
