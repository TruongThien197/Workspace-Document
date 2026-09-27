# Workspace Build Kickoff Guide

> **Purpose:** Tài liệu mở đầu để định hướng agent khi bắt đầu xây dựng Workspace trên một codebase lớn, chưa có harness hoàn chỉnh hoặc không thể thêm nhiều tài liệu vào repository.  
> **Companion source:** `workspace_v1_architecture_ideas.md`  
> **Initial implementation target:** `/goal`  
> **Primary stack assumption:** Java + Spring Boot nếu backend hiện tại sử dụng Java Spring Boot.

---

## 1. Vai trò của tài liệu này

Tài liệu này không thay thế Architecture Source, Implementation Spec hay Implementation Plan.

Nó giúp agent:

1. hiểu đúng mục tiêu Workspace;
2. biết phải khảo sát codebase theo hướng nào;
3. biết tám bước cần hoàn thành trước implementation;
4. không áp kiến trúc giả định lên một hệ thống lớn chưa được hiểu đầy đủ;
5. tạo đủ thông tin để sau đó xây `Goal Workspace` đúng trên hệ thống hiện tại;
6. giữ thiết kế đủ tổng quát để thêm các workspace khác sau `/goal`.

Có thể sử dụng tài liệu này ngay cả khi repository không có harness hoàn chỉnh, database rất lớn, tài liệu dự án bị phân tán, hoặc agent chỉ có context của phiên chat.

---

# 2. Nguồn kiến trúc phải đọc trước

Trước khi phân tích code, agent phải đọc:

```text
workspace_v1_architecture_ideas.md
```

Tài liệu đó là nguồn concept và architectural direction.

Các nguyên tắc không được tự ý thay đổi nếu chưa có lý do kỹ thuật rõ ràng:

```text
Workspace
=
Scope
+ Context
+ State
+ Rules
+ Skills / Tools
+ Output Contract
```

Workspace là execution context có trạng thái, không phải system prompt preset, alias của slash command, một skill lớn hay wrapper đơn giản quanh LLM.

Slash command như `/goal` chỉ là entry point để activate workspace.

---

# 3. Mục tiêu triển khai trước mắt

Không bắt đầu bằng việc code `/goal`.

Thứ tự bắt buộc:

```text
Understand Existing System
        ↓
Map Workspace
        ↓
Define Implementation Contract
        ↓
Confirm DB / Schema Impact
        ↓
Confirm Core Interfaces
        ↓
Create Implementation Plan
        ↓
Build Core Workspace Engine
        ↓
Implement Goal Workspace
```

`/goal` là workspace đầu tiên dùng để kiểm chứng Workspace Engine.

Không được thiết kế Core chỉ để `/goal` chạy được. Core phải đủ tổng quát để sau này có thể bổ sung `/deep-research`, `/document-to-md` và các workspace khác mà không phải viết lại toàn bộ execution flow.

---

# 4. Nguyên tắc khi làm trên codebase lớn

## 4.1 Không scan toàn bộ một cách mù quáng

Không cần đọc từng file trong repository. Khảo sát theo các đường đi chính:

```text
Chat Request
Conversation
Authentication / Authorization
Project
Document / Knowledge Source
Retrieval / RAG
LLM Integration
Persistence
Background Processing
API Response
```

Sau đó mở rộng sang những module có liên quan trực tiếp.

## 4.2 Existing architecture là constraint

Không bắt hệ thống hiện tại phải giống architecture ví dụ trong source document.

Ưu tiên:

```text
Reuse existing abstraction
→ Extend cleanly
→ Add new abstraction only when necessary
```

Không tạo layer mới chỉ để tên class giống tài liệu.

## 4.3 Database phải được hiểu trước khi đề xuất migration

Trước khi đề xuất migration cần xác định tối thiểu:

- conversation lưu ở đâu;
- user/project relationship;
- message model;
- document/source model;
- metadata model;
- RAG/vector metadata;
- permission enforcement;
- existing JSON/JSONB conventions;
- audit fields;
- soft delete / hard delete policy;
- migration mechanism;
- artifact/file storage nếu có.

Database lớn không có nghĩa phải hiểu toàn bộ schema. Chỉ cần lập **dependency slice** liên quan tới Workspace.

## 4.4 Không để Workspace bypass security

Workspace state chỉ lưu reference, không cấp quyền.

```text
Effective Scope
=
Workspace Requested Scope
∩ Project Scope
∩ Current User Permission
∩ Resource Availability
```

## 4.5 Không để LLM quyết định architecture

LLM/agent có thể phân tích, đề xuất, generate code và generate structured output.

Nhưng các quyết định sau phải nằm ở application architecture:

```text
Workspace lifecycle
State contract
Permission boundary
DB relationship
Skill/tool allow-list
Source scope
```

---

# 5. Tám bước trước implementation

## STEP 1 — Review codebase hiện tại

### Mục tiêu

Hiểu execution path hiện tại đủ để biết Workspace sẽ đi vào đâu.

### Cần tìm

**Chat**

```text
Chat Controller / API
Chat Service
Conversation Service
Message persistence
Prompt building
LLM call
Response building
```

**Knowledge / RAG**

```text
Document ingestion
Chunk model
Embedding
Vector store
Retrieval
Metadata filters
Evidence / citation
Index status
```

**Security**

```text
Authentication
Project membership
Document access
Authorization checks
```

**Persistence**

```text
Conversation tables/entities
Message tables/entities
Project
Documents
Vector metadata
Migration framework
Storage abstraction
```

**AI abstraction**

```text
LLM provider adapter
Embedding adapter
Spring AI / SDK integration
Tool calling if present
```

### Output của Step 1

```text
Current Architecture Map

Chat flow:
A → B → C → LLM

RAG flow:
A → B → Vector Store → C

Permission flow:
A → B → C

Relevant persistence:
...

Existing abstractions reusable for Workspace:
...

Risks / unknowns:
...
```

Không code ở bước này.

---

## STEP 2 — Map Workspace vào architecture hiện tại

### Mục tiêu

Xác định Workspace nằm ở đâu trong flow hiện tại.

Target conceptual flow:

```text
User Request
    ↓
Conversation
    ↓
Active Workspace Resolution
    ↓
Workspace Execution Context
    ↓
Skill / Tool / Retrieval
    ↓
LLM
    ↓
State Update
    ↓
Response
```

Agent phải map từng node trên sang component thực tế của dự án.

Ví dụ:

```text
Concept                  Existing Component
-------------------------------------------------
Conversation             ExistingConversationService
Workspace Resolution     NEW / existing candidate
RAG                      ExistingRetrievalService
Permission               ExistingProjectAccessService
Prompt Build             ExistingPromptAssembler
LLM                      ExistingGeminiAdapter
```

Phải xác định:

- Workspace nên nằm trước hay trong chat orchestration;
- component nào cần biết `activeWorkspace`;
- component nào không nên biết Workspace;
- RAG nhận scope bằng cách nào;
- permission được revalidate ở đâu;
- state được update ở transaction nào.

### Output của Step 2

`Workspace Integration Map` với sơ đồ before/after.

---

## STEP 3 — Chốt Workspace Implementation Spec

### Mục tiêu

Chuyển architecture concept thành contract phù hợp hệ thống thực tế.

Spec tối thiểu phải chốt:

```text
Workspace Definition
Workspace Runtime Instance
Workspace Type
Workspace Status / Lifecycle
Required Context
Scope
State
Instructions
Allowed Skills
Allowed Tools
Output Contract
Switch / Activate / Close behavior
Failure behavior
```

Phải phân biệt rõ:

```text
Definition
= static behavior/configuration of workspace type

Instance
= runtime state of one workspace in one conversation/project
```

### Với Goal phải chốt

```text
Goal được represent bằng gì?
Goal entity?
Document?
Section?
Logical reference?
Metadata?
```

Không được giả định có `goal_id` nếu domain hiện tại không có Goal entity.

---

## STEP 4 — Chốt DB / Schema Impact

### Mục tiêu

Xác định chính xác Workspace cần persistence ở đâu.

Agent phải trả lời:

```text
Có cần table mới không?
Conversation có cần active_workspace reference không?
State lưu normalized hay JSON/JSONB?
Scope lưu ở đâu?
Context lưu ở đâu?
Có reuse audit fields hiện tại không?
Delete behavior là gì?
```

Concept baseline:

```text
Conversation
    ↓
active workspace

Workspace Runtime Instance
├── identity
├── conversation/project reference
├── type/status
├── context
├── scope
├── state
└── definition version
```

Schema cuối phải dựa trên database hiện tại.

### Không được làm

- tự tạo nhiều table workspace-specific ngay từ đầu;
- duplicate conversation history vào workspace state;
- lưu permission snapshot rồi dùng nó làm authorization;
- lưu large artifact vào generic state nếu hệ thống đã có storage tốt hơn.

### Output của Step 4

```text
DB Impact Proposal

Existing objects reused:
...

New objects:
...

Changed objects:
...

Migration risk:
...

Rollback considerations:
...
```

---

## STEP 5 — Chốt interfaces / core components

### Mục tiêu

Xác định boundary của Workspace Engine trước khi implement.

Conceptual components có thể gồm:

```text
WorkspaceRegistry
WorkspaceService
WorkspaceResolver
WorkspaceContextBuilder
WorkspaceStateManager
WorkspaceScopeResolver
WorkspaceInstructionResolver
SkillRegistry
WorkspaceToolPolicy
```

Không bắt buộc dùng đúng các class name này.

Agent phải tìm xem:

```text
component nào đã tồn tại?
component nào có thể extend?
component nào thực sự cần tạo mới?
```

Core dependency direction nên giữ:

```text
Chat Orchestration
       ↓
Workspace Core
       ↓
Application capabilities
       ↓
RAG / Tool / LLM / Storage
```

Workspace Core không nên phụ thuộc cứng vào Goal. Goal phụ thuộc vào Workspace Core.

---

## STEP 6 — Tạo Implementation Plan

Chỉ tạo plan sau Steps 1–5.

Plan phải dựa trên dependency thật của codebase.

Không tạo milestone theo số lượng file. Nên chia theo capability:

```text
Foundation
Persistence
Lifecycle
Context Resolution
Scope / Permission
Chat Integration
RAG Integration
Skill / Tool Integration
Goal Workspace
Testing / Hardening
```

Mỗi milestone cần:

```text
Objective
Affected areas
Expected behavior
Tests
Definition of Done
Do-not-break constraints
```

---

## STEP 7 — Build Core Workspace Engine

Chỉ bắt đầu khi implementation plan đã được chốt.

Core cần chứng minh được ít nhất:

```text
Create workspace
Persist workspace
Activate workspace
Resolve active workspace
Switch workspace
Restore state
Build execution context
Resolve effective source scope
Apply permissions
Expose allowed capabilities
Update state safely
Return to default context
```

Core không được chứa logic như:

```java
if (workspace == GOAL) {
   ...
}
```

rải rác trong generic services.

Workspace-specific behavior phải đi qua definition/strategy/registry hoặc abstraction tương đương.

---

## STEP 8 — Implement `/goal`

`/goal` là implementation đầu tiên trên Core Workspace Engine.

Nó dùng để test:

```text
persistent context
state
scope
RAG filtering
permission
skills
evidence
multi-turn behavior
workspace switching
```

Goal Workspace phải được implement như một consumer của Workspace Core.

Không được biến Workspace Core thành Goal Engine.

---

# 6. Goal Workspace — hướng build đầu tiên

Goal Workspace phải hỗ trợ tối thiểu:

```text
Goal hiện tại là gì?
Goal nói gì?
Goal phụ thuộc vào gì?
Requirement nào hỗ trợ Goal?
Có evidence gì?
Goal A khác Goal B thế nào?
Có conflict/gap nào trong source?
```

## Goal Context

Cần resolve một Goal reference rõ ràng, có thể là:

```text
goal entity ID
```

hoặc:

```text
document + section reference
```

hoặc:

```text
existing project metadata
```

Phải dựa vào domain thật.

## Goal Scope

Ưu tiên nguồn theo thứ tự:

```text
Direct Goal Source
        ↓
Explicitly Linked Sources
        ↓
Related Requirements / Specs
        ↓
Metadata-related Sources
        ↓
Semantic Retrieval within allowed scope
```

Không search toàn project trước rồi yêu cầu model tự bỏ qua phần không liên quan.

## Goal Skills

Danh sách ban đầu có thể gồm:

```text
analyze_goal
summarize_goal
trace_requirement
find_dependency
compare_goals
detect_conflict
find_evidence
```

Nhưng trước khi implement từng skill, agent phải kiểm tra:

```text
Skill này có thực sự cần executor riêng?
Hay chỉ là một behavior trên cùng retrieval/orchestration pipeline?
```

Không tạo class/service riêng cho mỗi skill nếu không cần.

## Goal Evidence Rules

Phải phân biệt:

```text
EXPLICIT
```

source nói trực tiếp relationship;

và:

```text
INFERRED
```

AI suy luận từ evidence.

Không promote inference thành domain fact vĩnh viễn nếu chưa có workflow xác nhận.

---

# 7. Cách agent nên làm việc khi không có harness đầy đủ

Nếu chỉ có một phiên chat, agent nên duy trì một working summary ngắn sau mỗi phase.

Format đề xuất:

```text
WORKSPACE BUILD STATE

Confirmed:
- ...

Architecture map:
- ...

DB findings:
- ...

Reusable components:
- ...

New components likely required:
- ...

Open questions:
- ...

Decisions:
- ...

Next step:
- ...
```

Mỗi khi sang phiên mới, cung cấp:

1. `workspace_v1_architecture_ideas.md`;
2. file hướng dẫn này;
3. working summary gần nhất.

Như vậy không cần repository phải có harness đầy đủ mới tiếp tục được.

---

# 8. Khi codebase quá lớn

Agent không cần hiểu toàn bộ repository trước.

Dùng dependency-driven exploration.

Bắt đầu từ:

```text
Chat endpoint
```

trace xuống:

```text
conversation
→ auth
→ retrieval
→ prompt
→ LLM
→ persistence
```

Sau đó trace ngược từ database/repository đối với:

```text
conversation
project
document
message
```

Mục tiêu là hiểu **workspace dependency surface**, không phải toàn bộ enterprise system.

---

# 9. Khi database quá lớn

Không dump hoặc đọc toàn schema nếu không cần.

Tạo một database slice:

```text
Workspace Persistence Slice

Conversation
Project
User Membership
Message
Document
Knowledge Metadata
Vector Metadata
Artifact/File Storage
Audit/Migration conventions
```

Với mỗi object, ghi:

```text
Purpose
Primary key
Important relations
Relevant constraints
Delete behavior
Workspace impact
```

Chỉ mở rộng khi phát hiện dependency mới.

---

# 10. Khi spec hiện tại mâu thuẫn với code

Ưu tiên xác minh theo thứ tự:

```text
Current running code
        ↓
Current database / migration
        ↓
Current maintained spec
        ↓
Old plan / historical docs
```

Không silently chọn một phía.

Agent phải ghi:

```text
Observed implementation:
...

Documented expectation:
...

Difference:
...

Recommended resolution:
...
```

---

# 11. Các dấu hiệu architecture đang đi sai

Dừng và review nếu xuất hiện:

```text
Goal-specific if/else rải rác trong ChatService
Workspace chỉ là prompt string
Workspace state chỉ là chat history
RAG không enforce workspace scope
Permission chỉ check lúc workspace được tạo
LLM tự ghi arbitrary state vào DB
Mỗi skill tạo một service dù không có behavior riêng
Goal schema được tạo trước khi hiểu Goal domain
Workspace Core phụ thuộc Goal implementation
```

---

# 12. Definition of Ready trước khi code Core Workspace

Không bắt đầu Step 7 nếu chưa trả lời được:

```text
Current chat execution path là gì?
Workspace chèn vào điểm nào?
Conversation được persisted thế nào?
Permission được enforce ở đâu?
RAG filter hiện tại hoạt động thế nào?
Workspace state cần persist những gì?
DB thay đổi gì?
Core interface boundaries là gì?
Goal được represent thế nào?
Goal source scope được resolve thế nào?
```

Một vài chi tiết nhỏ có thể còn mở, nhưng các boundary trên phải rõ.

---

# 13. Definition of Ready trước khi code Goal

Core phải có ít nhất:

```text
Workspace Definition registration
Workspace runtime persistence
Workspace lifecycle
Active workspace resolution
Execution context builder
Effective scope resolution
Permission revalidation
Workspace-aware retrieval entry point
State update mechanism
Basic observability
```

Sau đó Goal mới được thêm.

---

# 14. Definition of Done của Goal proof

Goal Workspace được xem là proof thành công nếu:

```text
1. User activate /goal.
2. Goal context được resolve.
3. Context tồn tại qua nhiều turn.
4. Reload conversation không mất state.
5. Retrieval chỉ dùng effective workspace scope.
6. Permission thay đổi được revalidate.
7. Goal analysis có evidence.
8. Explicit và inferred relationship được phân biệt.
9. Có thể compare / trace / dependency analysis.
10. Có thể exit/switch workspace.
11. Generic Workspace Core không bị Goal-specific hóa.
12. Có thể mô tả cách thêm workspace thứ hai mà không sửa core lớn.
```

Điểm 12 là test kiến trúc quan trọng nhất.

---

# 15. Sau Goal

Sau khi `/goal` hoạt động, chưa triển khai workspace tiếp theo ngay.

Thực hiện một vòng architecture review:

```text
Goal Implementation
       ↓
Review Core Abstraction
       ↓
Identify Goal-specific leakage
       ↓
Refactor generic contracts if needed
       ↓
Freeze Workspace Core v1
       ↓
Add next workspace
```

Nếu `Deep Research` hoặc `Document-to-MD` yêu cầu mở rộng Core, ưu tiên extension point thay vì hard-code workspace type mới.

---

# 16. Sơ đồ tổng thể

```text
                           USER
                             │
                             ▼
                    Chat / Command API
                             │
                             ▼
                       Conversation
                             │
                             ▼
                    Workspace Resolver
                             │
                             ▼
              ┌─────────────────────────┐
              │ Workspace Core          │
              │                         │
              │ Definition              │
              │ Runtime Instance        │
              │ Lifecycle               │
              │ Context                 │
              │ State                   │
              │ Scope                   │
              │ Skill / Tool Policy     │
              └────────────┬────────────┘
                           │
                           ▼
                Workspace Execution Context
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
        Skills           Tools             RAG
          │                │                │
          └────────────────┼────────────────┘
                           ▼
                     Prompt / Agent
                           │
                           ▼
                           LLM
                           │
                           ▼
                 Validation / State Update
                           │
                           ▼
                Response / Evidence / Artifact
```

Goal là một implementation/configuration phía trên Core:

```text
Workspace Core
      │
      └── Goal Workspace
            ├── Goal Context
            ├── Goal Scope Policy
            ├── Goal Instructions
            ├── Goal Skills
            └── Goal Output Rules
```

Sau này:

```text
Workspace Core
├── Goal
├── Deep Research
├── Document-to-MD
└── Future Workspaces
```

---

# 17. Nguyên tắc cuối

Agent phải tối ưu theo thứ tự:

```text
Correct understanding
    ↓
Compatibility with existing system
    ↓
Clear boundaries
    ↓
Security / data correctness
    ↓
Extensibility
    ↓
Implementation simplicity
```

Không tối ưu cho số lượng class nhiều, architecture đẹp trên sơ đồ, framework mới hoặc agentic complexity.

Mục tiêu là:

> **Xây một Workspace Core đúng với hệ thống hiện tại, đủ sạch để Goal hoạt động đầy đủ và đủ mở để các workspace tiếp theo có thể phát triển trên cùng nền tảng.**
