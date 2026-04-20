---
name: pipeline-orchestration
description: "Orchestrate multi-agent development pipelines with automated handoffs, retry loops, and quality gates. TRIGGER when: user asks to run a pipeline, coordinate multiple agents on a project, execute a dev-QA loop, manage agent handoffs, or automate a multi-phase workflow. Also triggers on: 'run nexus', 'start pipeline', 'orchestrate agents', 'dev qa loop'. SKIP when: single-agent tasks, simple one-shot prompts, or tasks that don't involve agent coordination."
license: MIT
compatibility: "Requires Python 3.11+, access to at least one LLM provider (Grok/Gemini/Groq), and the NEXUS agent definitions from the agency-agents repository."
metadata:
  category: "orchestration"
  complexity: "expert"
  token-budget: "4000"
---

# Multi-Agent Pipeline Orchestration

You are executing a structured multi-agent pipeline. This skill governs how agents are activated, how work flows between them, and how quality is enforced at every boundary.

## Pipeline Architecture

The pipeline has 4 phases. Each phase has entry criteria, assigned agents, and exit gates. No phase is skippable. No gate is advisory.

```
SPEC --> [Phase 1: Planning] --> GATE --> [Phase 2: Architecture] --> GATE --> [Phase 3: Dev<>QA Loop] --> GATE --> [Phase 4: Integration] --> SHIP
```

### Phase 1: Project Decomposition
**Agent**: Senior Project Manager
**Input**: Raw project specification (markdown, PRD, or freeform description)
**Output**: Structured task list with the following per task:
- Task ID (sequential: TASK-001, TASK-002, ...)
- Description (one sentence, imperative mood)
- Type classification: `frontend` | `backend` | `design` | `infra` | `data` | `docs`
- Acceptance criteria (3-5 testable conditions per task)
- Dependencies (list of task IDs that must complete first)
- Estimated complexity: `S` | `M` | `L` | `XL`

**Gate criteria**: Minimum 3 tasks. Every task has acceptance criteria. No circular dependencies.

### Phase 2: Technical Architecture
**Agents**: Backend Architect + UX Architect (parallel execution)
**Input**: Project spec + Phase 1 task list
**Output**: Two documents consumed by all Phase 3 agents:
1. System architecture (DB schema, API contracts, service boundaries, tech stack decisions)
2. Design system (color tokens, typography scale, spacing grid, component inventory, responsive breakpoints)

**Gate criteria**: Architecture doc references every task from Phase 1. Design system includes at least: color palette, type scale, spacing unit.

### Phase 3: Dev/QA Loop (THE CORE)
This is the heart of the pipeline. For each task in the Phase 1 list (respecting dependency order):

```
FOR task IN tasks (topological order):
    attempt = 0
    WHILE attempt < 3:
        attempt += 1
        
        dev_output = call_agent(
            agent = select_developer(task.type),
            context = {
                "architecture": phase_2_output,
                "task": task,
                "qa_feedback": previous_qa_feedback OR null,
                "attempt": attempt
            }
        )
        
        qa_output = call_agent(
            agent = select_qa(task.type),
            context = {
                "task": task,
                "dev_output": dev_output,
                "acceptance_criteria": task.acceptance_criteria
            }
        )
        
        verdict = parse_verdict(qa_output)
        
        IF verdict == "PASS":
            record_completion(task)
            BREAK
        ELIF attempt == 3:
            escalate(task, qa_feedback_history)
            BREAK
        ELSE:
            previous_qa_feedback = qa_output
            CONTINUE
```

**Agent assignment matrix**:
| Task Type | Developer Agent | QA Agent |
|-----------|----------------|----------|
| frontend | Frontend Developer | Evidence Collector |
| backend | Backend Architect | API Tester |
| design | UX Architect | Evidence Collector |
| infra | DevOps Automator | Performance Benchmarker |
| data | Backend Architect | Evidence Collector |
| docs | (any available) | Reality Checker |

**QA verdict parsing**: Search the QA agent's output for `Verdict: PASS` or `Verdict: FAIL`. If neither is found, default to FAIL. The QA agent does not get the benefit of the doubt.

**Retry context injection**: On retry, the developer agent receives the FULL QA feedback (not a summary). The handoff document includes:
- Original task description and acceptance criteria
- The developer's previous output (what was rejected)
- The QA agent's specific failure reasons with evidence references
- Attempt number (so the developer knows urgency increases)

### Phase 4: Integration Validation
**Agent**: Reality Checker
**Input**: All Phase 3 outputs concatenated, plus the original spec
**Output**: Final verdict: `READY` | `NEEDS_WORK` | `NOT_READY`

**Gate criteria**: Reality Checker must reference specific evidence for each verdict category. "Looks good" is not a valid pass.

## Handoff Document Format

Every agent-to-agent transfer uses this structure:

```markdown
# Handoff: [FROM_AGENT] --> [TO_AGENT]
## Metadata
- Phase: [current phase]
- Task: [task ID and description]
- Attempt: [N of 3]
- Priority: [P0/P1/P2/P3]
- Timestamp: [ISO 8601]

## Context
[What has been done so far. Include relevant outputs from prior agents.]

## Your Assignment
[Exactly what this agent needs to produce.]

## Acceptance Criteria
- [ ] Criterion 1
- [ ] Criterion 2
- [ ] Criterion 3

## Reference Materials
- Architecture doc: [summary or path]
- Design system: [summary or path]
- Previous agent output: [included below or referenced]
```

## Pipeline State Tracking

Maintain a running status document throughout execution:

```markdown
# Pipeline Status: [PROJECT_NAME]
## Progress: [N/TOTAL] tasks complete
## Current Phase: [1-4]
## Current Task: [TASK-ID] (attempt [N/3])

| Task ID | Description | Status | Attempts | Agent | QA Verdict |
|---------|-------------|--------|----------|-------|------------|
| TASK-001 | ... | PASSED | 1/3 | Frontend Dev | PASS |
| TASK-002 | ... | IN_PROGRESS | 2/3 | Backend Arch | FAIL->retry |
| TASK-003 | ... | PENDING | 0/3 | - | - |

## Quality Metrics
- First-attempt pass rate: [X]%
- Total QA cycles: [N]
- Escalated tasks: [N]
- Average attempts per task: [X.X]
```

## Error Handling

**Agent fails to produce output**: Retry once with the same prompt. If still no output, substitute a different agent from the same category. If no substitute available, escalate the task.

**QA output is ambiguous**: Default to FAIL. Ask the QA agent to re-evaluate with explicit instructions: "Respond with exactly 'Verdict: PASS' or 'Verdict: FAIL' on its own line, followed by your reasoning."

**Dependency deadlock**: If task B depends on task A, and task A is escalated (3 failures), mark task B as BLOCKED. Continue with independent tasks. Report blocked tasks in the status document.

**Pipeline interrupted**: State is persisted after every task completion. Resume by loading state and continuing from the last incomplete task.

## Completion Criteria

The pipeline is complete when:
- All tasks are PASSED or ESCALATED (no tasks remain IN_PROGRESS or PENDING)
- Phase 4 Reality Checker has issued a verdict
- The status document is final with all metrics populated
- Escalated tasks (if any) are documented with failure history and recommended resolution
