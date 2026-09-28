# Skill/Workflow V1 — Architecture & Product Concept Source

> **Status:** Architecture / product concept source  
> **Purpose:** Tài liệu nguồn trước khi viết implementation spec, implementation plan và task breakdown.  
> **Confirmed model:** `Skill + multi-step workflow + temporary execution state`  
> **Initial implementation target:** `/goal`  
> **Future candidates:** `/deep-research`, `/document-to-md`

---

## 1. Quyết định kiến trúc

Sau khi xác nhận lại product behavior, `/goal` không phải một Workspace tồn tại qua nhiều lượt chat cho tới khi user chủ động exit.

Behavior đúng:

```text
User gọi /goal
    ↓
Khởi tạo Goal Skill execution
    ↓
Chạy workflow nhiều bước
    ↓
Dùng context / source / tools cần thiết
    ↓
Validate kết quả
    ↓
Trả output
    ↓
Execution kết thúc
    ↓
Chat quay lại trạng thái bình thường
```

Vì vậy abstraction chính là:

> **Skill + Workflow + Temporary Execution State**

Không cần xây Workspace Engine, active workspace hay workspace switching cho use case này.

---

## 2. Mental model

### 2.1 Skill

Skill là capability mà user chủ động gọi để hoàn thành một mục tiêu cụ thể.

Ví dụ:

```text
/goal
/deep-research
/document-to-md
```

Một Skill nên xác định:

- identity và trigger;
- input contract;
- instructions/rules;
- workflow;
- source policy;
- tools/capabilities;
- temporary execution state;
- output contract;
- validation/failure policy.

Skill kết thúc khi task hoàn tất, thất bại, bị hủy hoặc timeout.

### 2.2 Workflow

Workflow là chuỗi bước bên trong Skill.

Ví dụ Goal:

```text
UNDERSTAND
    ↓
RESOLVE SOURCES
    ↓
RETRIEVE EVIDENCE
    ↓
ANALYZE
    ↓
VALIDATE
    ↓
SYNTHESIZE
    ↓
COMPLETE
```

Workflow có thể deterministic, agent-assisted, tool-driven hoặc kết hợp. Không cần biến mỗi step thành một service/class riêng nếu codebase hiện tại không cần.

### 2.3 Temporary Execution State

Trong lúc Skill chạy, hệ thống có thể cần giữ:

```text
execution id
skill type
input
current step
resolved scope
selected sources
evidence refs
warnings
intermediate structured results
status
```

State này chỉ phục vụ execution hiện tại. Sau khi execution hoàn tất, state có thể bị giải phóng hoặc chỉ giữ execution summary nếu product cần audit/history.

> Temporary state không đồng nghĩa với bắt buộc lưu database.

Persistence phải được quyết định sau khi review runtime thực tế.

---

## 3. Công thức kiến trúc

> **Skill = Definition + Workflow + Execution Context + Temporary State + Tools + Output Contract**

- **Definition:** cấu hình/behavior tĩnh.
- **Workflow:** chuỗi phase/step.
- **Execution Context:** user/project/conversation/input/scope hiện tại.
- **Temporary State:** dữ liệu runtime giữa các bước.
- **Tools:** RAG/search/parser/LLM/application capabilities.
- **Output Contract:** kết quả cuối và rule validate.

---

## 4. Skill Definition và Skill Execution

### 4.1 Skill Definition

Definition là cấu hình tĩnh dùng chung cho mọi invocation.

```yaml
id: GOAL
command: /goal

input:
  required:
    - goal

sourcePolicy:
  projectScoped: true
  evidenceRequired: true

workflow:
  - understand
  - resolve_sources
  - retrieve
  - analyze
  - validate
  - synthesize

tools:
  - document_fetch
  - vector_search
  - metadata_search

output:
  mode: conversational
  evidenceRequired: true
```

V1 nên ưu tiên definition trong code/config version-controlled thay vì database động.

### 4.2 Skill Execution

Execution là một lần chạy cụ thể:

```json
{
  "executionId": "exec_123",
  "skill": "GOAL",
  "conversationId": "conv_10",
  "projectId": "project_1",
  "status": "RUNNING",
  "currentStep": "RETRIEVE",
  "input": {
    "goal": "..."
  },
  "scope": {},
  "evidenceRefs": [],
  "warnings": []
}
```

Khác với mô hình Workspace:

- không có `activeWorkspaceId`;
- không cần switch workspace;
- không giữ Goal mode cho turn tiếp theo;
- không mặc định resume Goal context ở message sau.

---

## 5. Lifecycle đề xuất

```text
CREATED
   ↓
VALIDATING_INPUT
   ↓
RUNNING
   ↓
VALIDATING_OUTPUT
   ↓
COMPLETED
```

Nhánh khác:

```text
FAILED
CANCELLED
TIMED_OUT
```

Chỉ thêm `WAITING`, `RETRYING` hoặc phase async nếu requirement thật sự cần.

---

## 6. Kiến trúc tổng thể

```text
User Message / Slash Command
           │
           ▼
     Command / Skill Router
           │
           ▼
       Skill Registry
           │
           ▼
      Skill Definition
           │
           ▼
   Execution Context Builder
           │
           ▼
    Workflow Orchestrator
           │
   ┌───────┼───────────┐
   ▼       ▼           ▼
  RAG     Tools       LLM
   │       │           │
   └───────┼───────────┘
           ▼
   Temporary State
           │
           ▼
    Output Validation
           │
           ▼
       Final Result
           │
           ▼
     Execution Closed
```

Sau khi execution đóng, chat quay về flow bình thường.

---

## 7. Các component conceptual

Tên class/package cuối cùng phải theo codebase thực tế.

- **SkillRegistry:** đăng ký và resolve Skill Definition.
- **SkillRouter:** nhận slash command/UI action và resolve Skill.
- **SkillExecutor / WorkflowOrchestrator:** điều phối lifecycle và workflow step.
- **ExecutionContextBuilder:** ghép user, conversation, project, definition, input, permission và scope thành runtime context.
- **ExecutionStateStore:** quản lý temporary state bằng request memory, cache, DB hoặc existing job infrastructure.
- **ScopeResolver:** tạo effective source scope.
- **ToolPolicy:** giới hạn tool/capability Skill được phép dùng.
- **OutputValidator:** kiểm tra schema, evidence, fidelity và output rules.

Không mặc định tạo component mới nếu codebase đã có abstraction tương đương.

---

## 8. Database / Persistence

### 8.1 Không mặc định cần database mới

Nếu workflow synchronous, ngắn và không cần resume:

```text
request
→ run workflow
→ return result
→ discard runtime state
```

thì có thể không cần table Skill Execution.

### 8.2 Khi nào nên persist execution

Cân nhắc persistence nếu có:

- workflow chạy lâu;
- async/background execution;
- retry;
- resume;
- audit;
- execution history;
- failure investigation;
- intermediate artifact;
- usage analytics;
- user cần xem execution status.

Nếu cần, concept có thể là:

```text
ai_skill_execution
├── id
├── skill_type
├── conversation_id
├── project_id
├── status
├── current_step
├── input_json
├── state_json
├── definition_version
├── started_at
├── completed_at
└── updated_at
```

Không tạo table chỉ vì tài liệu kiến trúc có ví dụ.

---

## 9. Scope và Security

Mỗi execution phải resolve source scope dựa trên quyền hiện tại:

```text
Effective Scope
=
Skill Requested Scope
∩ Project Scope
∩ User Accessible Sources
∩ Available Sources
```

Temporary state không phải authorization. Nếu workflow dài, operation nhạy cảm phải revalidate permission ở thời điểm thực thi.

---

## 10. RAG integration

Skill-aware retrieval nên nhận:

```text
query
+ project
+ effective scope
+ skill context
+ optional entity/reference context
```

Không nên search toàn project rồi chỉ dùng prompt yêu cầu model tự filter. Scope cần enforce ở retrieval/tool layer khi infrastructure hỗ trợ.

---

## 11. Instruction / Prompt composition

Có thể compose:

```text
Base System Rules
+
Project / Security Rules
+
Skill Instructions
+
Current Workflow Step Instructions
+
Execution Context
+
Retrieved Evidence / Tool Results
+
User Input
```

Không hard-code toàn bộ Goal logic vào một prompt khổng lồ nếu workflow có nhiều phase thực sự khác nhau.

---

## 12. Shared primitives

Các Skill tương lai có thể dùng chung primitive:

```text
retrieve_sources
collect_evidence
compare_sources
validate_evidence
extract_structure
write_artifact
```

Phân biệt:

- **User-facing Skill:** `/goal`, `/deep-research`, `/document-to-md`.
- **Workflow step/capability:** bước nội bộ.
- **Tool:** primitive thao tác hệ thống.

Không cần gọi mọi capability nội bộ là Skill.

---

# 13. Goal Skill — implementation đầu tiên

## 13.1 Mục tiêu

`/goal` phân tích một goal/task dựa trên nguồn được phép, tạo kết quả có evidence, rồi kết thúc execution.

```text
/goal "Đánh giá mục tiêu X và dependency của nó"
```

Flow:

```text
Parse Goal Input
      ↓
Clarify / Normalize Goal
      ↓
Resolve Relevant Source Scope
      ↓
Retrieve Evidence
      ↓
Analyze Goal
      ↓
Find Dependencies / Requirements / Conflicts
      ↓
Validate Claims
      ↓
Synthesize Result
      ↓
Complete Goal Execution
```

## 13.2 Goal input

Không được giả định Goal là database entity. Sau khi review hệ thống, Goal có thể là:

- free-text user goal;
- goal entity;
- document reference;
- section/reference trong tài liệu;
- metadata object;
- task object.

Implementation phải dựa vào domain thật.

## 13.3 Goal temporary state

```json
{
  "goalInput": "...",
  "normalizedGoal": "...",
  "currentStep": "ANALYZE",
  "resolvedSources": [],
  "evidenceRefs": [],
  "relationships": [],
  "warnings": []
}
```

Không lưu hidden chain-of-thought. Chỉ giữ structured runtime facts cần cho workflow.

## 13.4 Goal workflow

### Understand

- parse input;
- xác định target;
- validate context.

### Resolve Scope

- xác định project/source boundary;
- validate permission;
- xác định nguồn ưu tiên.

### Retrieve Evidence

Ưu tiên:

```text
Direct referenced source
      ↓
Explicitly linked source
      ↓
Metadata-related source
      ↓
Semantic retrieval within effective scope
```

### Analyze

Có thể gồm:

- summarize goal;
- identify success criteria;
- identify constraints;
- trace requirements;
- find dependencies;
- detect gap/conflict;
- compare reference khi input yêu cầu.

### Validate

Kiểm tra claim có evidence, inference có được label đúng, source có đúng scope và output có đủ phần bắt buộc.

### Synthesize

Trả final answer theo output contract.

## 13.5 Evidence rules

Phân biệt:

```text
EXPLICIT
```

source nói trực tiếp; và:

```text
INFERRED
```

AI suy luận từ evidence.

Inference phải label rõ và không tự persist thành domain fact nếu chưa có workflow xác nhận.

## 13.6 Output contract

```text
Goal Interpretation
Key Findings
Dependencies / Requirements / Conflicts (nếu có)
Evidence
Gaps / Uncertainty
```

UX có thể linh hoạt, nhưng evidence behavior phải nhất quán.

## 13.7 Definition of Done

Goal Skill là proof thành công khi:

1. `/goal` route đúng.
2. Input được validate.
3. Workflow có lifecycle rõ.
4. Temporary state không leak sang chat bình thường.
5. Retrieval bị giới hạn bởi effective scope.
6. Permission được enforce.
7. Kết luận quan trọng có evidence.
8. Explicit/inferred được phân biệt.
9. Insufficient evidence/failure được handle rõ.
10. Execution kết thúc và chat trở về flow bình thường.
11. Không cần active workspace/workspace switching.
12. Có thể thêm Skill thứ hai mà không sửa lớn Goal implementation.

---

# 14. Deep Research — hướng mở rộng

`/deep-research` dùng cùng execution architecture nhưng workflow dài hơn:

```text
DEFINE QUESTION
     ↓
PLAN
     ↓
COLLECT
     ↓
CROSS-CHECK
     ↓
SYNTHESIZE
     ↓
VALIDATE
     ↓
COMPLETE
```

Temporary state có thể chứa research question, sub-questions, plan, evidence refs, contradictions, open gaps và current phase.

Deep Research dễ có nhu cầu persistence hơn Goal nếu execution dài/async.

---

# 15. Document-to-MD — hướng mở rộng

`/document-to-md` là transformation skill:

```text
SOURCE RESOLUTION
      ↓
EXTRACTION
      ↓
STRUCTURE DETECTION
      ↓
MARKDOWN NORMALIZATION
      ↓
VALIDATION
      ↓
FINALIZE ARTIFACT
```

Temporary state có thể chứa source reference, conversion options, parser warnings, artifact reference và validation result.

Main path có thể không cần semantic RAG. Vì vậy generic execution layer không được hard-code theo RAG.

---

# 16. So sánh ba Skill Workflow

| Dimension | Goal | Deep Research | Document-to-MD |
|---|---|---|---|
| Primary input | Goal/task | Research question | Source document |
| Main pattern | Evidence-based analysis | Multi-step research | Transformation |
| Typical duration | Short/medium | Medium/long | Medium |
| RAG importance | High | High | Low/optional |
| Temporary state | Medium | High | Medium |
| Persistence need | Optional | More likely | Depends on artifact flow |
| Main output | Analysis | Research synthesis | Markdown artifact |
| Main risk | Unsupported inference | Unsupported synthesis | Content loss/mutation |

---

# 17. API / Invocation Concept

Có thể map slash command vào endpoint chat hiện tại bằng metadata:

```json
{
  "message": "...",
  "skill": "GOAL"
}
```

hoặc endpoint riêng. Không chốt API trước khi review backend hiện tại.

Nếu async mới cân nhắc execution status/cancel APIs.

---

# 18. Observability

Mỗi execution nên trace tối thiểu:

```text
executionId
skillType
definitionVersion
conversationId
projectId
workflowStep
resolvedScopeSummary
toolsUsed
retrievalCount
executionStatus
duration
```

Không log secret hoặc nội dung nhạy cảm không cần thiết.

---

# 19. Error / Fallback Behavior

- **Invalid Input:** không chạy workflow chính.
- **Missing Source:** không silently mở rộng scope ngoài policy.
- **Insufficient Evidence:** trả uncertainty/gap rõ.
- **Tool Failure:** phân loại retryable/non-retryable nếu infrastructure hỗ trợ.
- **Invalid Temporary State:** fail safely.
- **Timeout / Cancel:** đóng execution và cleanup đúng lifecycle.

---

# 20. Non-goals V1

V1 không mặc định xây:

- Workspace Engine;
- active workspace;
- workspace switch/stack;
- long-lived Goal mode;
- autonomous loop vô hạn;
- generic workflow DSL phức tạp;
- dynamic skill marketplace;
- distributed orchestration platform;
- persistence cho mọi Skill;
- table riêng cho từng Skill;
- nested skill graph nếu chưa cần.

Mục tiêu:

> **Một execution model đủ sạch để user-facing Skill chạy workflow nhiều bước với context/state tạm thời và dùng chung application capabilities.**

---

# 21. Nguyên tắc kiến trúc

1. Skill lifetime phải rõ: start → run → complete/fail/cancel.
2. Workflow là implementation detail có cấu trúc.
3. Temporary state có boundary và không leak sang chat sau execution.
4. Persistence theo requirement, không theo architecture mẫu.
5. Security lấy quyền hiện tại làm authority.
6. Scope phải enforce được ở application/retrieval layer.
7. Model không sở hữu lifecycle/state transition.
8. Definition tách khỏi execution.
9. Goal không hard-code vào generic layer.
10. Reuse infrastructure hiện có trước khi tạo framework mới.

---

# 22. Anti-pattern cần tránh

```text
/goal chỉ đổi system prompt
Goal state tồn tại vô hạn trong conversation
Conversation.active_workspace_id cho use case này
LLM tự quyết định arbitrary state transition
Search toàn project rồi filter bằng prompt
Mỗi workflow step = một service dù không cần
Tạo skill_execution table trước khi biết persistence requirement
Generic executor chứa nhiều if (skill == GOAL)
Goal inference được persist như fact không validate
```

---

# 23. Suggested implementation sequence

### Phase A — Understand and Map

1. Review chat/RAG/security/persistence flow.
2. Map Skill/Workflow execution vào architecture hiện tại.
3. Xác định sync vs async.
4. Xác định persistence requirement.

### Phase B — Core Contract

5. Skill Definition.
6. Skill Registry/Router.
7. Execution Context.
8. Workflow lifecycle.
9. Temporary State contract.
10. Scope/Tool policy.
11. Output validation.

### Phase C — Integration

12. Chat/command integration.
13. RAG integration.
14. Permission enforcement.
15. LLM/prompt integration.
16. Observability.

### Phase D — Goal

17. Goal input contract.
18. Goal workflow.
19. Goal retrieval/evidence.
20. Goal output validation.
21. Goal integration tests.

### Phase E — Review

22. Review Goal-specific leakage.
23. Refine generic extension points.
24. Chuẩn bị Skill tiếp theo.

---

# 24. Definition of Done cho architecture proof

Architecture V1 đạt khi:

1. Skill invocation có lifecycle rõ.
2. Goal chạy multi-step workflow.
3. Temporary execution state được quản lý an toàn.
4. Không cần long-lived Workspace abstraction.
5. Scope/permission được enforce.
6. RAG/tool integration không Goal-specific.
7. Output được validate.
8. Execution cleanup rõ.
9. Persistence chỉ có khi requirement thật sự cần.
10. Có thể thêm Skill thứ hai bằng extension thay vì rewrite core.

---

# 25. Short Architecture Summary

> Hệ thống sử dụng mô hình **Skill + multi-step Workflow + Temporary Execution State**. Slash command như `/goal` kích hoạt một Skill execution có input, source scope, rules, tools, workflow và output contract riêng. Trong lúc workflow chạy, hệ thống giữ execution state tạm thời để điều phối các bước và evidence. Khi task hoàn thành, execution kết thúc và chat trở về flow bình thường. Runtime state chỉ được persist khi có requirement thực tế như async, retry, resume, audit hoặc artifact history. Generic execution layer phải đủ mở để sau Goal có thể thêm Deep Research và Document-to-MD mà không cần thiết kế lại nền tảng.
