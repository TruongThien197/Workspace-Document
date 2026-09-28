# Skill/Workflow Build Kickoff Guide

> **Purpose:** Tài liệu mở đầu cho agent khi tích hợp Skill/Workflow vào codebase lớn, database phức tạp hoặc repository chưa có harness hoàn chỉnh.  
> **Companion source:** `skill_workflow_v1_architecture_ideas.md`  
> **Initial target:** `/goal`  
> **Confirmed model:** `Skill + multi-step workflow + temporary execution state`

---

## 1. Vai trò của tài liệu

Đây không phải implementation spec và không phải implementation plan.

Nó giúp agent biết:

- cần review phần nào;
- cần map architecture ra sao;
- cần chốt gì trước khi code;
- khi nào database thật sự cần thay đổi;
- cách dùng Goal để proof execution model;
- cách tránh overengineering khi codebase/database lớn.

Nếu repository không cho phép thêm nhiều tài liệu, dùng file này cùng Architecture Source làm context đầu phiên chat.

---

## 2. Behavior phải hiểu đúng

```text
User gọi /goal
    ↓
Goal Skill execution bắt đầu
    ↓
Workflow chạy nhiều bước
    ↓
Temporary execution state được dùng
    ↓
Final result
    ↓
Execution kết thúc
    ↓
Chat trở về flow bình thường
```

Không xây long-lived Goal Workspace hoặc yêu cầu user exit/switch, trừ khi requirement sau này thay đổi.

---

# 3. Tám việc cần thực hiện

```text
1. Review codebase hiện tại
2. Map Skill/Workflow vào architecture hiện tại
3. Chốt Skill/Workflow Implementation Spec
4. Chốt DB/schema impact và execution persistence requirement
5. Chốt interfaces/core components
6. Tạo implementation plan
7. Build hoặc reuse Skill Workflow execution layer
8. Implement Goal Skill trên execution layer đó
```

Không nhảy trực tiếp tới Step 8.

---

# 4. STEP 1 — Review codebase hiện tại

## Mục tiêu

Hiểu đủ execution path để biết Skill tích hợp ở đâu. Không cần đọc toàn repository.

### Chat

```text
API / Controller
→ Chat Service
→ Conversation
→ Prompt
→ LLM
→ Response
```

### RAG

```text
Document
→ Ingestion
→ Chunk/Metadata
→ Embedding
→ Vector Store
→ Retrieval
→ Evidence
```

### Security

Tìm authentication, project membership và document/source permission.

### Persistence

Tìm:

- conversation/message;
- project/member;
- document/source;
- vector metadata;
- existing job/execution;
- audit fields;
- JSON/JSONB;
- migration framework;
- artifact/file storage.

### AI Integration

Tìm LLM adapter, embedding adapter, tool calling, structured output, retry, background worker và streaming nếu có.

## Output Step 1

```text
CURRENT ARCHITECTURE MAP

Chat flow:
...

RAG flow:
...

Permission flow:
...

Relevant persistence:
...

Reusable execution/job abstractions:
...

Risks / unknowns:
...
```

Chưa code.

---

# 5. STEP 2 — Map Skill/Workflow vào architecture hiện tại

Map:

```text
User Command
    ↓
Skill Router
    ↓
Skill Definition
    ↓
Execution Context
    ↓
Workflow
    ↓
RAG / Tool / LLM
    ↓
Validation
    ↓
Result
```

sang component thật.

| Concept | Existing component | Action |
|---|---|---|
| Command routing | ... | reuse/extend/new |
| Skill registry | ... | reuse/extend/new |
| Workflow orchestration | ... | reuse/extend/new |
| Temporary state | ... | request/cache/DB |
| Retrieval | ... | reuse/extend |
| Permission | ... | reuse |
| Output validation | ... | reuse/new |

Phải trả lời:

- command parse ở đâu;
- orchestration point ở đâu;
- existing job/workflow abstraction có không;
- RAG có filter/scope API không;
- Tool calling hoạt động thế nào;
- Goal có thể chạy synchronous không;
- có async/retry/resume requirement không.

---

# 6. STEP 3 — Chốt Implementation Spec

## Generic Skill Contract

```text
Skill ID
Command / Trigger
Input Contract
Source Policy
Workflow Steps
Allowed Tools
Temporary State
Output Contract
Validation Policy
Failure Policy
```

## Execution Contract

```text
executionId
skillType
status
currentStep
input
executionContext
temporaryState
result
```

## Goal Contract

Xác định Goal trong domain thật: free-text, entity, document, document section, task/ticket hay metadata reference.

Không tạo `goal_id` hoặc Goal table từ giả định.

---

# 7. STEP 4 — Chốt DB/schema impact

Đây là bước đánh giá, không phải mặc định tạo migration.

### Nếu synchronous + short-lived

```text
request-scoped temporary state
→ không cần database mới
```

### Nếu async/retry/resume/audit

Có thể cần execution persistence:

```text
ai_skill_execution
├── id
├── skill_type
├── status
├── current_step
├── conversation_id
├── project_id
├── input/state
└── timestamps
```

Agent phải trả lời:

```text
Execution có cần survive backend restart?
User có cần xem execution status?
Có retry?
Có background worker?
Có audit/history requirement?
Có intermediate artifact?
```

Nếu không có requirement tương ứng, không tạo DB chỉ để lưu temporary state.

## Output Step 4

```text
DB IMPACT

Existing objects reused:
...

New objects required:
...

Why persistence is/is not needed:
...

Migration risk:
...

Rollback:
...
```

---

# 8. STEP 5 — Chốt interfaces/core components

Conceptual boundaries:

```text
SkillRegistry
SkillRouter
SkillExecutor / WorkflowOrchestrator
ExecutionContextBuilder
ExecutionStateManager
ScopeResolver
ToolPolicy
OutputValidator
```

Tên class cuối phải theo project.

Ưu tiên:

```text
reuse existing abstraction
→ extend existing abstraction
→ create new component only when necessary
```

Dependency direction:

```text
Chat/Application Layer
        ↓
Skill Execution Layer
        ↓
Existing Capabilities
        ↓
RAG / Tool / LLM / Storage
```

---

# 9. STEP 6 — Tạo implementation plan

Chỉ tạo plan sau Steps 1–5.

Plan nên chia theo capability:

```text
Foundation
Routing / Registry
Execution Context
Workflow Lifecycle
State Handling
Scope / Permission
RAG / Tools
Prompt / LLM
Output Validation
Goal Workflow
Testing / Hardening
```

Mỗi milestone cần objective, affected areas, constraints, expected behavior, tests và Definition of Done.

---

# 10. STEP 7 — Build hoặc reuse execution layer

Không xây framework mới nếu hệ thống đã có abstraction phù hợp.

Execution layer cần chứng minh:

```text
Resolve Skill
Validate Input
Create Execution Context
Run Workflow Steps
Manage Temporary State
Enforce Scope/Permission
Call Existing Capabilities
Validate Output
Close Execution
Cleanup State
```

Nếu hệ thống có job/workflow engine tốt, tích hợp vào đó.

---

# 11. STEP 8 — Implement Goal Skill

Suggested flow:

```text
/goal
  ↓
VALIDATE INPUT
  ↓
UNDERSTAND GOAL
  ↓
RESOLVE EFFECTIVE SCOPE
  ↓
RETRIEVE EVIDENCE
  ↓
ANALYZE
  ↓
VALIDATE CLAIMS
  ↓
SYNTHESIZE
  ↓
COMPLETE
```

Goal là proof cho routing, lifecycle, temporary state, scope, RAG, permission, evidence, failure và cleanup.

---

# 12. Goal source strategy

Ưu tiên:

```text
Direct reference
     ↓
Explicitly linked source
     ↓
Metadata-related source
     ↓
Semantic retrieval within effective scope
```

Không search rộng rồi chỉ dùng prompt để filter.

---

# 13. Goal temporary state

```json
{
  "goalInput": "...",
  "normalizedGoal": "...",
  "currentStep": "ANALYZE",
  "sourceRefs": [],
  "evidenceRefs": [],
  "warnings": []
}
```

Không lưu hidden chain-of-thought. Không giữ Goal state sau execution chỉ vì user đã gọi `/goal`.

---

# 14. Goal evidence

Phân biệt **EXPLICIT** và **INFERRED**.

Nếu evidence không đủ:

- nói rõ gap;
- không fabricate;
- không silently mở rộng source ngoài policy.

---

# 15. Khi codebase lớn

Không scan repository mù quáng.

Dùng dependency-driven exploration:

```text
Chat endpoint
→ orchestration
→ permission
→ retrieval
→ prompt
→ LLM
→ response
```

Sau đó trace persistence của conversation, project, membership, document/source, message và job/execution.

Mục tiêu là hiểu **Skill execution dependency surface**.

---

# 16. Khi database rất lớn

Không đọc toàn schema.

Tạo slice:

```text
Conversation
Project
User Membership
Message
Document / Knowledge Source
Vector Metadata
Job / Execution
Artifact / File Storage
Audit / Migration Convention
```

Với mỗi object ghi Purpose, Primary key, Relations, Constraints, Delete behavior và Skill/Workflow impact.

---

# 17. Khi spec và code mâu thuẫn

Ghi rõ:

```text
Observed implementation:
...

Documented expectation:
...

Difference:
...

Impact on Skill/Workflow:
...

Recommended resolution:
...
```

Không tự reconcile âm thầm.

---

# 18. Khi chỉ làm việc qua phiên chat

Sau mỗi phase duy trì:

```text
SKILL/WORKFLOW BUILD STATE

Confirmed:
- ...

Current architecture map:
- ...

Execution model:
- sync/async/unknown

DB findings:
- ...

Reusable components:
- ...

New components likely required:
- ...

Goal representation:
- ...

Open questions:
- ...

Decisions:
- ...

Next step:
- ...
```

Phiên sau cung cấp Architecture Source, Kickoff Guide, working summary gần nhất và các spec/code snippets cần thiết.

---

# 19. Definition of Ready trước implementation

Không code generic execution layer nếu chưa trả lời được:

```text
Current chat execution path là gì?
/goal được trigger ở đâu?
Skill orchestration chèn vào đâu?
Goal input/domain representation là gì?
RAG nhận scope/filter thế nào?
Permission enforce ở đâu?
Workflow sync hay async?
Temporary state cần persist không?
Existing job/workflow abstraction có gì?
Output validation nằm ở đâu?
```

Không cần biết toàn codebase/database; chỉ cần các boundary này.

---

# 20. Definition of Ready trước Goal

Phải có hoặc xác định cách reuse:

```text
Skill registration/routing
Input validation
Execution context
Workflow lifecycle
Temporary state handling
Effective scope resolution
Permission enforcement
RAG/tool integration
Output validation
Basic observability
```

---

# 21. Definition of Done cho Goal proof

Goal thành công khi:

1. User gọi `/goal`.
2. Goal execution bắt đầu đúng.
3. Workflow chạy theo phase rõ.
4. State tồn tại đủ cho execution.
5. State không leak sang chat sau complete.
6. Retrieval đúng scope.
7. Permission đúng.
8. Kết luận có evidence.
9. Explicit/inferred được phân biệt.
10. Error/insufficient evidence được handle.
11. Execution close/cleanup đúng.
12. Có thể thêm Skill thứ hai mà không rewrite lớn generic layer.

Điểm 12 là architecture test quan trọng nhất.

---

# 22. Dấu hiệu implementation đang đi sai

```text
Conversation có active_workspace_id dù product không có workspace
/goal chỉ đổi system prompt
Goal state tồn tại vô hạn
Generic ChatService đầy if (skill == GOAL)
LLM tự quyết định DB state transition
Permission chỉ check một lần rồi dùng state cũ
RAG search ngoài scope
Tạo nhiều table trước khi chốt persistence requirement
Mỗi workflow step bị over-abstract thành service riêng
Skill engine mới duplicate existing job/orchestration system
```

---

# 23. Sau khi Goal build thành công

Không implement Skill tiếp theo ngay.

```text
Goal Implementation
       ↓
Architecture Review
       ↓
Find Goal-specific leakage
       ↓
Refine generic contracts
       ↓
Confirm persistence strategy
       ↓
Freeze Skill/Workflow Core v1
       ↓
Add next Skill
```

Deep Research và Document-to-MD là extension test của abstraction.

---

# 24. Sơ đồ tổng thể

```text
                       USER
                         │
                         ▼
                  Chat / Command API
                         │
                         ▼
                    Skill Router
                         │
                         ▼
                   Skill Registry
                         │
                         ▼
              Skill Execution Context
                         │
                         ▼
                 Workflow Orchestrator
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
       RAG              Tools            LLM
        │                │                │
        └────────────────┼────────────────┘
                         ▼
                 Temporary State
                         │
                         ▼
                  Output Validator
                         │
                         ▼
                    Final Result
                         │
                         ▼
                 Execution Closed
```

Goal là implementation đầu tiên; sau đó Deep Research, Document-to-MD và future skills phải tái sử dụng cùng execution model thay vì tạo core mới.

---

# 25. Nguyên tắc cuối

Agent ưu tiên:

```text
Understand current system
    ↓
Reuse existing capabilities
    ↓
Define clear execution boundaries
    ↓
Security / data correctness
    ↓
Goal proof
    ↓
Extensibility
    ↓
Simplicity
```

Mục tiêu không phải xây workflow framework tổng quát cho mọi hệ thống.

> **Mục tiêu là tích hợp một Skill execution model đúng với hệ thống hiện tại, đủ để Goal chạy workflow nhiều bước an toàn và đủ mở để Skill tiếp theo tái sử dụng cùng nền tảng.**
