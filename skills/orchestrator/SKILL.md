---
name: orchestrator
description: 메인 에이전트가 서브 에이전트들을 기동하고 비동기/병렬 상태를 제어하며, JSON 메시지를 통해 오케스트레이션을 지휘합니다.
allowed-tools:
  - define_subagent
  - invoke_subagent
  - send_message
  - read_file
  - write_file
  - command
---

# Orchestrator Workflow SOP

이 스킬은 메인 에이전트가 하위 서브 에이전트들을 기동하고, 표준 JSON 메시지 기반 협업 루프를 효율적으로 지휘하여 사용자가 내린 복잡한 개발 작업을 완수하도록 설계된 Antigravity CLI 오케스트레이터용 핵심 스킬입니다.

## When to use this skill
- 사용자가 복잡한 아키텍처 분석, 다중 컴포넌트 개발 또는 종합적인 코드 수정 및 검증을 포함하는 복잡한 태스크를 요청했을 때 사용합니다.
- 복수의 에이전트가 동시(Fan-out/Fan-in) 또는 직렬(Pipeline)로 실행되도록 제어해야 할 때 사용합니다.

## Instructions
1. **서브 에이전트 생명주기 관리**:
   - 세션 초기화 시, 아래 명시된 서브 에이전트 정의 프로필(`.agents/agents/{agent_name}/agent.json`)을 읽고 `define_subagent` 도구로 즉시 등록하십시오.
   - 각 하위 태스크 성격에 맞는 에이전트를 `invoke_subagent`를 통해 백그라운드 태스크로 구동하고, 반환된 Conversation ID를 상태 큐(Queue)에 기록하여 관리하십시오.
2. **협업 메시지 파싱 및 라우팅**:
   - 에이전트 간 통신 시, 반드시 아래의 JSON 메시지 규격을 문자열 형태로 페이로드에 래핑하여 송수신해야 합니다.
     ```json
     {
       "sender": "송신 에이전트명",
       "action": "REQUEST_REVIEW | TASK_COMPLETE | REFACTOR_REQUEST | STATUS_UPDATE",
       "target_artifact": "대상 파일의 절대 경로",
       "content": "업무 세부 내용 및 피드백 내용",
       "metadata": {
         "task_id": "작업 식별자",
         "status": "상태값",
         "additional_context": "기타 문맥 데이터"
       }
     }
     ```
   - 메시지를 수신하면 JSON을 우선 파싱하여, `action` 필드에 따라 알맞은 후속 에이전트에게 포워딩하거나 작업을 승인하십시오.
3. **격리 작업 공간 사용**:
   - 모든 작업 및 중간 결과물은 에이전트들이 [_workspace/](file:///D:/01_Source/oh-my-agy/_workspace) 하위에 저장 및 수정하도록 지침을 설계하십시오.

---

## 배포된 서브 에이전트 및 스킬 명세 목록

본 오케스트레이터 스킬 하에 등록되어 구동되는 서브 에이전트와 그들이 소유한 커스텀 스킬 목록입니다.

### 1) [omc-architect](file:///D:/01_Source/oh-my-agy/.agents/agents/omc-architect.md)
- **역할**: Task Decomposer & Architect
- **장착 커스텀 스킬**: [omc-task-decomposer](file:///D:/01_Source/oh-my-agy/.agents/skills/omc-task-decomposer/SKILL.md)
  - **자동화 임무**: 복잡한 요구사항과 기존 코드를 분석하여 의존성이 정돈된 JSON 형태의 세부 태스크 구현 계획서를 자동 스캐폴딩합니다.

### 2) [omc-executor](file:///D:/01_Source/oh-my-agy/.agents/agents/omc-executor.md)
- **역할**: Code Synthesizer & Implementer
- **장착 커스텀 스킬**: [omc-code-synthesizer](file:///D:/01_Source/oh-my-agy/.agents/skills/omc-code-synthesizer/SKILL.md)
  - **자동화 임무**: 할당받은 단일 태스크 계획에 맞춰 `_workspace/` 하위에 코드를 직접 작성, 수정, 구현합니다.

### 3) [omc-reviewer](file:///D:/01_Source/oh-my-agy/.agents/agents/omc-reviewer.md)
- **역할**: Quality Assurer & Code Reviewer
- **장착 커스텀 스킬**: [omc-quality-assurance](file:///D:/01_Source/oh-my-agy/.agents/skills/omc-quality-assurance/SKILL.md)
  - **자동화 임무**: 구현된 코드 파일에 대해 정적 분석, 스타일/린트 검사, 단위 테스트 생성을 통해 무결성을 정적/동적으로 검증하고 리포트를 산출합니다.

---

## Workflow
1. **초기화 및 분석**:
   - 오케스트레이터가 가동되면, `omc-architect` 서브 에이전트를 기동하여 사용자가 요구한 프로젝트 아키텍처에 대한 분석과 세부 구현 태스크 맵(`tasks.json`) 생성을 위임합니다.
2. **비동기 분배 (Fan-out)**:
   - `omc-architect`가 반환한 태스크 맵을 확인하여, 의존 관계가 없는 선행 태스크 카드들을 `omc-executor` 서브 에이전트 인스턴스들에게 병렬로 할당 및 기동합니다.
3. **구현 수집 및 피드백 루프**:
   - `omc-executor`들이 작업을 완료하여 JSON 메시지로 보고하면, 이를 `omc-reviewer` 인스턴스에 즉시 매핑하여 검증을 요청합니다.
4. **통합 및 검증 (Fan-in)**:
   - 검증이 불합격(`FAIL`)인 경우, `omc-reviewer`가 도출한 리포트를 JSON의 `content`에 적어 `omc-executor`에게 리팩토링 요청(`REFACTOR_REQUEST`)을 송신하고 수정을 반복하게 합니다.
   - 모든 검증이 최종 `PASS`한 경우에 한해, 임시 산출물들을 사용자가 확인한 이후에 병합하여 최종적으로 인도합니다.
