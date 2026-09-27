# Workspace V1 — Architecture & Product Concept Source

> **Status:** Architecture / product concept source  
> **Purpose:** Làm tài liệu nguồn trước khi viết implementation spec, milestone plan và task breakdown.  
> **Scope V1:** `Goal Workspace`, `Deep Research Workspace`, `Document-to-MD Workspace`  
> **Target system:** AI Chatbot gắn với Project / Knowledge Base, sử dụng RAG và cơ chế evidence-based answering.

---

## 1. Mục tiêu của tài liệu

Tài liệu này chốt mô hình tư duy, kiến trúc đề xuất, phạm vi V1 và nguyên tắc triển khai **Workspace** cho AI Chatbot.

Workspace không được xem đơn giản là một phiên bản mới của “skill” hay một prompt preset. Workspace là **execution context có trạng thái**, quyết định AI đang làm việc trong phạm vi nào, với nguồn nào, rule nào, capability nào và output contract nào.

Mục tiêu chính của V1:

1. Định nghĩa được abstraction Workspace đủ rõ để tái sử dụng.
2. Hỗ trợ một conversation có thể chuyển giữa các workspace.
3. Workspace giữ context/state xuyên suốt nhiều lượt chat.
4. Workspace kiểm soát source scope, instruction, skill/tool và output behavior.
5. Không phá kiến trúc chatbot/RAG hiện tại; Workspace nằm như một orchestration layer phía trên.
6. Triển khai trước 3 workspace:
   - `GOAL`
   - `DEEP_RESEARCH`
   - `DOCUMENT_TO_MD`
7. Giữ thiết kế đủ mở để sau này thêm `DOCUMENT`, `PLANNING`, `ANALYSIS`, `CODE`, `REVIEW`, v.v. mà không phải sửa lại core flow.

---

# 2. Mental model cốt lõi

## 2.1 Workspace khác Skill

Cần chốt ranh giới ngay từ đầu:

> **Workspace quyết định AI đang làm việc trong “môi trường” nào.**  
> **Skill quyết định AI có thể thực hiện “hành động” nào trong môi trường đó.**

Ví dụ:

```text
Goal Workspace
├── analyze_goal
├── compare_goals
├── trace_requirement
├── find_dependency
└── detect_conflict
```

`Goal Workspace` không phải là một skill lớn. Nó là môi trường chứa context, state, source scope và các capability liên quan đến Goal.

### Skill

Một skill thường:

- xử lý một intent/task tương đối cụ thể;
- có input → execution → output;
- có thể kết thúc sau một lần chạy;
- không nhất thiết duy trì state lâu dài;
- có thể dùng lại trong nhiều workspace.

Ví dụ:

```text
summarize
compare
extract
trace
validate
```

### Workspace

Một workspace:

- tồn tại qua nhiều turn;
- giữ working state;
- giới hạn nguồn dữ liệu;
- định nghĩa instruction;
- quyết định skill/tool khả dụng;
- có lifecycle;
- có thể tạo artifact;
- có thể được activate/deactivate/reactivate.

---

# 3. Vị trí của Workspace trong kiến trúc tổng thể

Hierarchy đề xuất:

```text
User
 │
 └── Project
       │
       ├── Documents / Knowledge Sources
       │
       └── Conversation
              │
              └── Active Workspace
                     │
                     ├── Scope
                     ├── Context
                     ├── State
                     ├── Instructions
                     ├── Skills
                     ├── Tools
                     └── Artifacts
```

Ý nghĩa:

| Layer | Trách nhiệm |
|---|---|
| Project | Boundary dữ liệu và quyền truy cập lớn nhất |
| Conversation | Lịch sử tương tác User ↔ AI |
| Workspace | Execution context hiện tại |
| Skill | Capability / action |
| Tool | Cơ chế thực thi bên ngoài LLM |
| Artifact | Kết quả có trạng thái như Markdown, report, research note |

---

# 4. Công thức Workspace V1

Có thể dùng công thức ngắn gọn:

> **Workspace = Scope + Context + State + Rules + Skills/Tools + Output Contract**

Trong đó:

- **Scope**: AI được phép nhìn vào nguồn nào.
- **Context**: AI đang tập trung vào entity/task nào.
- **State**: thông tin cần nhớ xuyên nhiều turn.
- **Rules**: behavior/instruction/evidence policy.
- **Skills/Tools**: capability được phép sử dụng.
- **Output Contract**: workspace tạo ra loại kết quả gì và yêu cầu chất lượng nào.

---

# 5. Tách Workspace Definition và Workspace Instance

Đây là một quyết định kiến trúc quan trọng.

## 5.1 Workspace Definition

Là cấu hình **tĩnh**, dùng chung cho mọi user/conversation.

Ví dụ:

```yaml
type: GOAL
description: Analyze goals and their relationships to requirements and project evidence

sourcePolicy:
  projectScoped: true
  evidenceRequired: true

skills:
  - analyze_goal
  - compare_goal
  - trace_requirement
  - find_dependency
  - detect_conflict

tools:
  - vector_search
  - document_search
  - metadata_search
```

Definition nên chứa:

- type;
- description;
- default instructions;
- source policy;
- allowed skills;
- allowed tools;
- state schema;
- required context;
- output contract;
- optional UI metadata.

## 5.2 Workspace Instance / Session

Là trạng thái **động**, gắn với conversation/user/project cụ thể.

Ví dụ:

```json
{
  "workspaceId": "ws_123",
  "type": "GOAL",
  "conversationId": "conv_10",
  "projectId": "project_1",
  "status": "ACTIVE",
  "context": {
    "goalId": "goal_25"
  },
  "scope": {
    "documentIds": ["doc_1", "doc_2"]
  },
  "state": {
    "currentTask": "dependency-analysis",
    "selectedRequirementIds": ["req_10"]
  }
}
```

Tách Definition/Instance giúp:

- dùng chung engine cho nhiều workspace;
- tránh hard-code logic theo command;
- dễ thêm workspace mới;
- dễ test;
- dễ version workspace definition;
- không trộn configuration với session data.

---

# 6. Các thành phần bắt buộc của Workspace V1

## 6.1 Identity

Tối thiểu:

```text
workspaceId
workspaceType
conversationId
projectId
status
createdAt
updatedAt
```

Workspace type V1:

```text
DEFAULT
GOAL
DEEP_RESEARCH
DOCUMENT_TO_MD
```

`DEFAULT` là context chat bình thường và không nhất thiết phải có full workspace behavior như ba workspace chuyên biệt, nhưng nên được xem là fallback execution context.

---

## 6.2 Knowledge / Source Scope

Scope quyết định:

> AI được phép sử dụng nguồn nào trong workspace hiện tại?

Ví dụ:

```json
{
  "projectId": "p1",
  "documentIds": ["d1", "d2"],
  "documentTypes": ["GOAL", "REQUIREMENT", "SPEC"]
}
```

Scope phải được áp dụng **trước retrieval**, không phải search toàn bộ rồi mới lọc sau.

Công thức an toàn:

```text
Effective Source Scope
=
Workspace Scope
∩ Project Sources
∩ User Accessible Sources
```

Workspace tuyệt đối không được bypass permission.

---

## 6.3 Working Context

Context trả lời:

> AI hiện đang tập trung vào đối tượng nào?

Ví dụ Goal:

```json
{
  "currentGoalId": "goal_10",
  "relatedRequirementIds": ["req_1", "req_2"]
}
```

Deep Research:

```json
{
  "researchQuestion": "...",
  "researchMode": "project-only"
}
```

Document-to-MD:

```json
{
  "sourceDocumentId": "doc_18",
  "conversionProfile": "faithful"
}
```

---

## 6.4 Workspace State

State là phần cần tồn tại xuyên nhiều lượt chat.

Không nên lưu raw chain-of-thought hoặc reasoning nội bộ.

Chỉ lưu **working facts/state có ích cho workflow**.

Ví dụ:

```json
{
  "currentTask": "compare-goals",
  "selectedEntities": ["goal_a", "goal_b"],
  "completedSteps": ["source-selection", "initial-analysis"],
  "warnings": [],
  "artifactId": null
}
```

State nên:

- nhỏ;
- structured;
- serializable;
- versionable;
- có giới hạn kích thước;
- không biến thành bản sao của conversation history.

---

## 6.5 Instructions / Rules

Prompt cuối không nên chỉ có workspace prompt.

Composition đề xuất:

```text
Base System Rules
      +
Project / Security Rules
      +
Workspace Instructions
      +
Skill Instructions (nếu đang chạy skill)
      +
Workspace Context
      +
Retrieved Evidence / Tool Results
      +
User Request
```

Workspace rules có thể bao gồm:

- evidence requirement;
- source boundary;
- output format;
- refusal/fallback behavior;
- artifact rules;
- citation/provenance rules;
- workflow-specific constraints.

---

## 6.6 Skills / Capabilities

Workspace definition chỉ expose capability phù hợp.

Ví dụ:

```text
GOAL
├── analyze_goal
├── compare_goal
├── trace_requirement
├── find_dependency
└── detect_conflict
```

Một skill có thể được nhiều workspace dùng chung.

Ví dụ:

```text
summarize
validate
compare
extract_structure
```

V1 nên ưu tiên **allow-list** thay vì cho mọi workspace thấy toàn bộ skill.

---

## 6.7 Tools

Tool là primitive thực thi.

Ví dụ:

```text
vector_search
document_search
metadata_search
repository_search
artifact_writer
document_parser
```

Skill có thể gọi một hay nhiều tool.

Workspace quyết định tool nào được phép dùng.

---

## 6.8 Output Contract / Artifact

Không phải workspace nào cũng chỉ trả text chat.

Ví dụ:

- Goal → analysis answer + evidence.
- Deep Research → research synthesis / structured report.
- Document-to-MD → Markdown artifact.

Vì vậy workspace definition nên có output contract.

Ví dụ:

```yaml
output:
  mode: artifact
  artifactType: markdown
  requiresValidation: true
```

---

# 7. Workspace Lifecycle V1

V1 nên giữ lifecycle đơn giản:

```text
CREATED
   ↓
AWAITING_CONTEXT
   ↓
ACTIVE
   ↓
INACTIVE
   ↓
CLOSED
```

Ý nghĩa:

- `CREATED`: workspace vừa được tạo.
- `AWAITING_CONTEXT`: thiếu goal/document/research question cần thiết.
- `ACTIVE`: đang là workspace xử lý request.
- `INACTIVE`: còn state nhưng hiện không active.
- `CLOSED`: kết thúc, không dùng tiếp trừ khi tạo/reopen theo rule.

## 7.1 Một active workspace trên mỗi conversation

V1 nên có rule:

> **Mỗi conversation chỉ có một active workspace tại một thời điểm.**

Điều này đơn giản hóa:

- routing;
- UI;
- prompt composition;
- state consistency;
- debugging.

Một conversation có thể giữ nhiều workspace instance inactive để quay lại sau.

---

# 8. Workspace Switching / “Swap”

`/goal`, `/deep-research`, `/document-to-md` trong mô hình mới nên được xem là:

> **Workspace Entry Command**

Không phải skill invocation.

Ví dụ:

```text
/goal
   ↓
Workspace Router
   ↓
Resolve/Create Goal Workspace
   ↓
Collect required context
   ↓
Activate workspace
   ↓
Subsequent chat uses Goal Workspace
```

## 8.1 V1 nên explicit switch

V1 nên ưu tiên:

- slash command;
- UI selector;
- suggestion card/action.

Không nên để model tự ý đổi workspace dựa trên intent.

Lý do:

- tránh UX bất ngờ;
- dễ debug;
- dễ audit;
- không có ambiguity;
- không cần classifier phức tạp ngay từ đầu.

Auto-routing có thể là V2.

---

# 9. Workspace Router / Resolver

Nhiệm vụ:

1. đọc command/UI action;
2. resolve target workspace type;
3. validate permission;
4. validate required context;
5. create/reactivate workspace instance;
6. deactivate workspace cũ;
7. set active workspace;
8. trả metadata cho UI.

Pseudo flow:

```text
switchWorkspace(conversationId, targetType, inputContext)
    ↓
validate conversation
    ↓
validate project access
    ↓
load workspace definition
    ↓
resolve required context
    ↓
create or reuse instance
    ↓
deactivate old workspace
    ↓
activate target
```

---

# 10. Workspace Context Builder

Đây là component core.

Input:

```text
Conversation
+
Active Workspace
+
Workspace Definition
+
Current User Message
+
Project Metadata
```

Output:

```text
WorkspaceExecutionContext
```

Ví dụ:

```json
{
  "workspaceType": "GOAL",
  "projectId": "p1",
  "scope": {},
  "workingContext": {},
  "state": {},
  "instructions": [],
  "allowedSkills": [],
  "allowedTools": []
}
```

Không nên để controller hoặc LLM adapter tự ghép các phần này thủ công.

---

# 11. Workspace-Aware RAG

RAG hiện tại thường:

```text
query
+ projectId
→ retrieve
```

Workspace-aware RAG:

```text
query
+ projectId
+ workspaceScope
+ permissionFilter
+ optional entity context
→ retrieve
```

Ví dụ Goal:

```text
projectId = P1
documentType IN [GOAL, REQUIREMENT, SPEC]
goalId / relatedIds = current goal context
```

Deep Research có thể dùng scope rộng hơn.

Document-to-MD có thể không dùng semantic RAG làm pipeline chính; nó thiên về full-document extraction/transform/validation.

Điều này quan trọng vì **workspace không bắt buộc phải chạy cùng một pipeline**.

---

# 12. Skill Registry

Đề xuất có registry riêng:

```text
SkillRegistry
├── summarize
├── compare
├── analyze_goal
├── trace_requirement
├── research_plan
├── synthesize_evidence
├── extract_structure
└── validate_markdown
```

Workspace definition chỉ tham chiếu ID:

```yaml
skills:
  - analyze_goal
  - trace_requirement
```

Không hard-code skill trực tiếp trong WorkspaceService.

---

# 13. Tool Policy

Tương tự skill registry:

```text
Workspace Tool Policy
```

Ví dụ:

| Workspace | Tools |
|---|---|
| Goal | vector search, metadata search, doc fetch |
| Deep Research | vector search, doc search, optional external search |
| Document-to-MD | document parser, file reader, artifact writer, validator |

Tool access phải được allow-list theo workspace.

---

# 14. Artifact Management

V1 nên định nghĩa artifact như một output có lifecycle riêng.

Đặc biệt cần cho `DOCUMENT_TO_MD`, và sau này có thể dùng cho Deep Research report.

Artifact metadata đề xuất:

```text
artifactId
workspaceId
type
name
status
sourceRefs
version
createdAt
updatedAt
```

Không nhất thiết V1 phải có hệ thống artifact phức tạp, nhưng interface nên tồn tại để tránh trả toàn bộ content chỉ như một chat message.

---

# 15. Data Model đề xuất cho V1

Có thể bắt đầu gọn bằng một bảng workspace và JSON/JSONB cho state.

Ví dụ logic:

```text
ai_workspace
├── id
├── conversation_id
├── project_id
├── type
├── status
├── scope_json
├── context_json
├── state_json
├── definition_version
├── created_at
└── updated_at
```

Conversation:

```text
conversation
└── active_workspace_id
```

Nếu DB hiện tại không phù hợp với JSONB, có thể normalize dần về sau.

### Lưu ý

Không nên đưa mọi thứ vào `state_json`.

Những field cần query/index/filter thường xuyên nên có column riêng.

---

# 16. API Concept V1

Không cần chốt URL chính xác ở tài liệu ý tưởng, nhưng cần đủ operation.

Ví dụ:

```text
POST /conversations/{id}/workspaces
GET  /conversations/{id}/workspaces
GET  /conversations/{id}/workspaces/active
POST /conversations/{id}/workspaces/{workspaceId}/activate
PATCH /workspaces/{workspaceId}/context
PATCH /workspaces/{workspaceId}/state
POST /workspaces/{workspaceId}/close
```

Hoặc command-level API đơn giản:

```text
POST /conversations/{id}/workspace/switch
```

V1 có thể expose API đơn giản nhưng service/domain bên trong vẫn giữ abstraction đầy đủ.

---

# 17. UI / UX Concept V1

User phải luôn biết mình đang ở workspace nào.

Ví dụ:

```text
┌────────────────────────────────────────┐
│ 🎯 Goal Workspace                     │
│ Goal: Improve onboarding              │
│ Sources: 8 project documents          │
└────────────────────────────────────────┘
```

Tối thiểu UI cần:

- workspace badge/name;
- active target/context;
- source scope summary;
- switch workspace;
- exit workspace / return default;
- suggested actions phù hợp workspace.

## 17.1 Hai tầng suggestion

### Workspace suggestion

```text
Goal
Deep Research
Document-to-MD
```

### Contextual action / skill suggestion

Ví dụ Goal:

```text
Analyze goal
Find dependencies
Compare goals
Trace requirements
```

Điều này giữ lại giá trị của “skill suggestion” cũ nhưng đặt nó đúng tầng.

---

# 18. Permission / Security

Workspace không phải security boundary độc lập thay thế Project.

Mọi request phải validate:

```text
User
  ↓
Conversation Access
  ↓
Project Membership
  ↓
Document Permission
  ↓
Workspace Scope
```

Effective scope:

```text
workspace requested sources
∩
project sources
∩
user-accessible sources
```

Các case phải handle:

- user bị remove khỏi project;
- document bị delete;
- project bị delete;
- workspace state giữ documentId không còn tồn tại;
- workspace được restore sau thời gian dài;
- source permission thay đổi.

State cũ không được coi là authorization.

---

# 19. Evidence & Provenance

Với chatbot knowledge-base, evidence policy nên là rule cấp hệ thống/workspace.

Nguyên tắc:

1. Claim liên quan project knowledge phải có evidence.
2. Workspace không được trích nguồn ngoài effective scope.
3. Nếu không đủ evidence → nói rõ thiếu dữ liệu.
4. Research phải giữ provenance theo source.
5. Document-to-MD phải giữ provenance về source document và conversion warnings.
6. Không lưu “model reasoning” làm evidence.

---

# 20. Error / Fallback Behavior

Workspace engine nên có behavior rõ cho:

### Missing context

Ví dụ `/goal` nhưng chưa chọn goal:

```text
Workspace = GOAL
Status = AWAITING_CONTEXT
```

Chatbot/UI yêu cầu chọn goal.

### Missing source

Goal tồn tại nhưng source không indexed:

- không silently search ngoài scope;
- trả trạng thái thiếu source/evidence;
- có thể suggest indexing nếu hệ thống hỗ trợ.

### Unsupported skill

Không cố chạy skill ngoài allow-list.

### Invalid state

Nếu state trỏ tới entity đã bị xóa:

- clear invalid reference;
- chuyển `AWAITING_CONTEXT`;
- không hallucinate entity cũ.

---

# 21. Observability

Workspace làm flow phức tạp hơn nên cần logging/tracing tối thiểu.

Nên log:

```text
conversationId
workspaceId
workspaceType
workspaceDefinitionVersion
resolvedScopeSummary
selectedSkill
toolsUsed
retrievalCount
artifactId
executionStatus
```

Không log secret hoặc nội dung nhạy cảm không cần thiết.

Việc có `workspaceDefinitionVersion` rất hữu ích khi debug prompt/behavior thay đổi.

---

# 22. Non-goals của V1

Để tránh over-engineering, V1 **không cần**:

- auto workspace switching bằng LLM classifier;
- nested workspace;
- arbitrary workspace composition;
- workspace marketplace/plugin dynamic;
- realtime multi-user shared workspace;
- workspace stack phức tạp;
- branch/merge workspace state;
- autonomous long-running agent loop;
- unrestricted external web research;
- full artifact collaboration editor.

V1 tập trung vào:

> **Một active workspace / conversation, explicit switching, persistent structured state, source-aware execution.**

---

# 23. Kiến trúc component đề xuất

```text
ChatController / Chat API
        │
        ▼
ConversationService
        │
        ▼
WorkspaceService
        │
        ├── WorkspaceRegistry
        ├── WorkspaceRouter
        ├── WorkspaceStateStore
        ├── WorkspaceContextBuilder
        ├── WorkspaceInstructionResolver
        ├── SkillRegistry
        └── ToolPolicy
        │
        ▼
Workspace Execution Context
        │
        ├── Retrieval Adapter
        ├── Skill Executor
        ├── Tool Executor
        └── Artifact Manager
        │
        ▼
Prompt / Agent Orchestrator
        │
        ▼
LLM Provider
        │
        ▼
Response + Evidence + Artifact
```

Workspace không nên tự implement LLM call.

Workspace cấu hình và điều phối **cách** request được xử lý.

---

# 24. Luồng chat chuẩn

```text
User Message
    ↓
Load Conversation
    ↓
Resolve Active Workspace
    ↓
Build Workspace Execution Context
    ↓
Resolve Intent / Skill
    ↓
Apply Workspace Tool Policy
    ↓
Retrieve / Execute Tools
    ↓
Build Prompt
    ↓
LLM
    ↓
Validate Output
    ↓
Update Workspace State
    ↓
Return Response / Artifact
```

---

# 25. Workspace V1 #1 — Goal Workspace

## 25.1 Mục đích

Goal Workspace phục vụ việc:

> **đọc hiểu, phân tích và truy vết một goal trong phạm vi knowledge của project.**

Không chỉ trả lời “goal là gì”, mà duy trì context để user làm việc với goal qua nhiều lượt.

Ví dụ:

```text
/goal
→ chọn Goal A

"Phân tích goal này"

"Nó phụ thuộc requirement nào?"

"Có conflict với architecture hiện tại không?"

"So với Goal B thì khác gì?"

"Cho tôi evidence của kết luận thứ 2"
```

Tất cả request tiếp theo dùng cùng goal context cho tới khi user đổi goal hoặc workspace.

---

## 25.2 Required Context

Tối thiểu:

```text
projectId
goalId / goal reference
```

Nếu chưa có `goalId`:

```text
status = AWAITING_CONTEXT
```

và UI/chat yêu cầu chọn goal.

---

## 25.3 Source Scope

Nguồn ưu tiên:

```text
Goal document
Related requirements
Related specifications
Architecture documents
Roadmap / planning documents
Explicitly linked project documents
```

Không nên mặc định search toàn project nếu đã có relationship metadata tốt.

Retrieval strategy nên ưu tiên:

```text
Directly linked source
    ↓
Same entity / metadata relation
    ↓
Semantic retrieval within allowed types
    ↓
Broader project retrieval only if policy allows
```

---

## 25.4 State

Ví dụ:

```json
{
  "currentGoalId": "goal_a",
  "comparisonGoalIds": [],
  "currentTask": "analysis",
  "selectedRequirementIds": [],
  "findings": [],
  "openQuestions": []
}
```

`findings` chỉ lưu structured working result ngắn nếu thực sự cần; không copy toàn bộ câu trả lời.

---

## 25.5 Skills

V1 đề xuất:

```text
analyze_goal
summarize_goal
trace_requirement
find_dependency
compare_goals
detect_conflict
find_evidence
```

Một số skill có thể được implement bằng cùng retrieval + prompt engine nhưng khác instruction/output schema.

---

## 25.6 Tools

```text
document_fetch
vector_search
metadata_search
relationship_lookup
```

Nếu hệ thống chưa có relationship graph, V1 có thể dùng metadata + semantic retrieval trước.

---

## 25.7 Instruction Rules

Goal Workspace nên enforce:

1. Không tự tạo requirement/dependency không có nguồn.
2. Phân biệt:
   - explicit dependency;
   - inferred relationship.
3. Inference phải ghi rõ là inference và có evidence hỗ trợ.
4. Khi user hỏi conflict:
   - nêu hai statement/source liên quan;
   - mô tả conflict cụ thể;
   - không gắn nhãn conflict nếu evidence chưa đủ.
5. Nếu goal thiếu definition/success criteria → chỉ ra gap.
6. Không dùng nguồn ngoài effective scope nếu workspace policy không cho phép.

---

## 25.8 Suggested Actions

UI/chat suggestion:

```text
Analyze this goal
Summarize goal
Find dependencies
Trace requirements
Compare with another goal
Find conflicts
Show supporting evidence
```

---

## 25.9 Output Contract

Goal response nên có:

```text
Answer / Analysis
Evidence
Optional gaps / uncertainty
Optional next contextual actions
```

Không bắt buộc mọi response có cùng template cứng, nhưng evidence behavior phải nhất quán.

---

## 25.10 Acceptance Criteria V1

Goal Workspace đạt V1 nếu:

- activate được từ `/goal` hoặc UI;
- chọn/resolve được một goal;
- giữ goal context qua nhiều turn;
- retrieval bị giới hạn đúng scope;
- user có thể đổi goal;
- có evidence;
- không retrieve tài liệu user không có quyền;
- switch sang workspace khác rồi quay lại không làm state hỏng;
- invalid/deleted goal được handle an toàn.

---

# 26. Workspace V1 #2 — Deep Research Workspace

## 26.1 Mục đích

Deep Research Workspace không nên được hiểu đơn giản là:

> “search nhiều lần rồi trả lời dài”.

Nó là workspace dành cho:

> **nghiên cứu một câu hỏi phức tạp theo nhiều bước, nhiều nguồn, có kế hoạch, evidence ledger và synthesis cuối.**

Khác Goal Workspace:

- Goal tập trung vào một entity chính.
- Deep Research tập trung vào một **research question/problem** có thể cần nhiều sub-question và nhiều loại nguồn.

---

## 26.2 Required Context

Tối thiểu:

```text
researchQuestion
projectId hoặc source mode
```

Ví dụ:

```json
{
  "researchQuestion": "Các rủi ro chính nếu chuyển indexing pipeline sang async?",
  "sourceMode": "PROJECT_ONLY"
}
```

---

## 26.3 Source Modes

V1 nên định nghĩa rõ.

### Baseline đề xuất

```text
PROJECT_ONLY
```

Research chỉ dùng knowledge đã được project cho phép.

Nếu hệ thống có external search tool, có thể feature-gate:

```text
PROJECT_PLUS_EXTERNAL
```

Nhưng external source nên là **optional**, không mặc định.

Lý do:

- dễ kiểm soát quality;
- giữ đúng privacy/security;
- tránh định nghĩa “deep research” đồng nghĩa với “web search”;
- cho phép V1 chạy được ngay trên KBase.

---

## 26.4 Research State

State cần giàu hơn Goal:

```json
{
  "researchQuestion": "...",
  "subQuestions": [],
  "researchPlan": [],
  "coveredSubQuestions": [],
  "evidenceRefs": [],
  "contradictions": [],
  "openGaps": [],
  "currentPhase": "COLLECTING"
}
```

Không lưu hidden reasoning.

Chỉ lưu research artifacts/structured state.

---

## 26.5 Research Phases

V1 có thể dùng flow:

```text
DEFINE
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

### DEFINE

Chuẩn hóa research question và constraint.

### PLAN

Phân rã thành sub-question.

### COLLECT

Tìm source/evidence.

### CROSS-CHECK

So sánh source, tìm conflict/gap.

### SYNTHESIZE

Tổng hợp thành answer/report.

### VALIDATE

Kiểm tra claim có evidence, source có trong scope, gap có được disclose.

---

## 26.6 Skills

V1 đề xuất:

```text
create_research_plan
search_sources
collect_evidence
compare_sources
detect_contradiction
identify_gap
synthesize_findings
validate_evidence
summarize_research
```

---

## 26.7 Tools

```text
vector_search
document_search
document_fetch
metadata_search
optional_external_search
artifact_writer
```

External search chỉ khả dụng nếu definition/config cho phép.

---

## 26.8 Evidence Ledger

Deep Research nên có khái niệm evidence ledger ở mức structured state.

Ví dụ:

```json
[
  {
    "claimId": "c1",
    "sourceRef": "doc_10",
    "supportType": "SUPPORTS",
    "note": "..."
  },
  {
    "claimId": "c1",
    "sourceRef": "doc_11",
    "supportType": "CONTRADICTS",
    "note": "..."
  }
]
```

Không cần triển khai entity/table riêng ngay V1 nếu quá nặng; có thể giữ transient/JSON.

Nhưng concept này quan trọng để Deep Research không biến thành prompt dài.

---

## 26.9 Contradiction Handling

Nếu hai nguồn mâu thuẫn:

- không tự chọn một nguồn thắng;
- nêu rõ source nào nói gì;
- xem metadata/date/version nếu có;
- ưu tiên source chính thức/current theo rule nếu system có policy;
- nếu chưa resolve được → đánh dấu unresolved.

---

## 26.10 Output Contract

Deep Research nên có hai mode output:

### Conversational Result

Dùng khi user hỏi giữa quá trình.

### Research Summary / Report

Khi kết thúc:

```text
Research Question
Key Findings
Evidence
Contradictions / Alternative Findings
Gaps / Unknowns
Conclusion / Synthesis
```

Có thể tạo Markdown artifact sau này hoặc ngay V1 nếu Artifact Manager sẵn sàng.

---

## 26.11 Suggested Actions

```text
Create research plan
Investigate sub-question
Find more evidence
Cross-check sources
Show contradictions
Identify missing information
Synthesize findings
Create research summary
```

---

## 26.12 Acceptance Criteria V1

- giữ research question xuyên nhiều turn;
- phân rã được thành sub-question;
- search có scope rõ;
- evidence được track;
- source conflict được expose;
- incomplete evidence không bị trình bày như fact chắc chắn;
- có thể synthesize kết quả;
- state không phụ thuộc vào hidden model reasoning;
- source permission luôn được revalidate.

---

# 27. Workspace V1 #3 — Document-to-MD Workspace

## 27.1 Vì sao đây vẫn là Workspace chứ không chỉ là Skill?

Nếu chỉ có:

```text
/document-to-md file.pdf
→ convert
→ return
→ done
```

thì đây đúng là một skill/tool.

Nhưng nếu product muốn user có thể:

```text
/document-to-md
→ chọn document

"Giữ nguyên heading hierarchy"

"Bảng ở trang 12 đang sai, sửa lại"

"Không chuyển phần appendix"

"Chuẩn hóa code block"

"Cho tôi preview"

"Xuất bản final markdown"
```

thì cần:

- selected source;
- conversion options;
- intermediate artifact;
- validation state;
- warnings;
- iteration history.

Đó là một **stateful conversion workspace**.

---

## 27.2 Mục đích

> Chuyển một source document sang Markdown có cấu trúc, có thể kiểm tra và chỉnh sửa qua nhiều lượt trước khi finalize.

V1 nên ưu tiên **faithful conversion**, không phải rewriting.

---

## 27.3 Required Context

```text
sourceDocumentId / uploaded source
```

V1 nên giới hạn:

> một primary document cho mỗi active Document-to-MD Workspace.

Batch conversion có thể để V2.

---

## 27.4 Supported Source Types

Phụ thuộc parser hiện có, nhưng concept có thể bao gồm:

```text
PDF
DOCX
HTML
Plain text
Other supported project documents
```

Không nên quảng bá format chưa có parser đáng tin cậy.

---

## 27.5 State

```json
{
  "sourceDocumentId": "doc_20",
  "conversionProfile": "FAITHFUL",
  "includeSections": [],
  "excludeSections": [],
  "preserveTables": true,
  "preserveCodeBlocks": true,
  "currentArtifactId": "artifact_99",
  "warnings": [],
  "validationStatus": "PENDING"
}
```

---

## 27.6 Conversion Pipeline

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
PREVIEW / ITERATION
      ↓
FINALIZE ARTIFACT
```

### Extraction

Lấy text, structure, table/code/media references nếu parser hỗ trợ.

### Structure Detection

Xác định:

- heading hierarchy;
- lists;
- paragraphs;
- tables;
- code;
- quotes;
- footnotes;
- links.

### Markdown Normalization

Render về Markdown theo rule.

### Validation

Kiểm tra:

- missing section;
- duplicated section;
- broken heading hierarchy;
- malformed table;
- lost code block;
- suspicious content loss.

### Iteration

User có thể yêu cầu chỉnh phần cụ thể.

### Finalize

Tạo Markdown artifact.

---

## 27.7 Skills

```text
extract_document_structure
convert_to_markdown
normalize_markdown
repair_section
preserve_table
preserve_code_block
validate_conversion
compare_with_source
finalize_markdown
```

---

## 27.8 Tools

```text
document_fetch
document_parser
page/section_reader
artifact_writer
markdown_validator
optional_structure_detector
```

Document-to-MD không cần vector RAG là main path.

Điều này chứng minh Workspace Engine phải hỗ trợ nhiều execution pattern.

---

## 27.9 Fidelity Rules

Default profile V1:

```text
FAITHFUL
```

Nguyên tắc:

1. Không tự rewrite nội dung trừ khi user yêu cầu.
2. Không tự thêm thông tin không có trong source.
3. Giữ hierarchy hợp lý.
4. Preserve code/table/link khi có thể.
5. Nếu layout không thể chuyển chính xác → warning.
6. Không che giấu content loss.
7. Mọi phần bị bỏ qua theo user rule phải được track trong state/config.

Có thể thêm profile tương lai:

```text
CLEAN
COMPACT
DOCS_STYLE
```

nhưng không cần V1.

---

## 27.10 Artifact Behavior

Output chính nên là:

```text
Markdown Artifact
```

Chat response chỉ nên nói:

- conversion status;
- warning;
- changed sections;
- validation result.

Không nên nhồi toàn bộ tài liệu dài vào conversation nếu đã có artifact layer.

---

## 27.11 Suggested Actions

```text
Convert document
Preview Markdown
Fix heading structure
Repair table
Preserve code blocks
Validate against source
Exclude section
Finalize Markdown
```

---

## 27.12 Acceptance Criteria V1

- activate được workspace với một source document;
- giữ conversion settings qua nhiều turn;
- tạo được Markdown artifact;
- validate được cấu trúc cơ bản;
- user có thể sửa một phần mà không cần bắt đầu lại toàn bộ;
- source permission luôn được revalidate;
- không thêm nội dung ngoài source mặc định;
- warning rõ khi parser không giữ được structure.

---

# 28. So sánh ba Workspace V1

| Dimension | Goal | Deep Research | Document-to-MD |
|---|---|---|---|
| Primary focus | Goal entity | Research question | Source document |
| Main data pattern | Entity-centric | Multi-source / multi-step | Full-document transformation |
| RAG importance | High | High | Low / optional |
| State complexity | Medium | High | Medium |
| Main output | Analysis | Synthesis/report | Markdown artifact |
| Evidence requirement | High | Very high | Source fidelity |
| External source | No by default | Optional feature-gated | No |
| Artifact | Optional | Optional/recommended | Required |
| Typical duration | Multi-turn | Multi-turn, longer | Multi-turn conversion |
| Key risk | Hallucinated relationships | Unsupported synthesis | Content loss / mutation |

---

# 29. Workspace Definition examples

## 29.1 Goal

```yaml
type: GOAL

requiredContext:
  - goalId

sourcePolicy:
  projectScoped: true
  evidenceRequired: true

skills:
  - analyze_goal
  - summarize_goal
  - trace_requirement
  - find_dependency
  - compare_goals
  - detect_conflict
  - find_evidence

tools:
  - document_fetch
  - vector_search
  - metadata_search

output:
  mode: conversational
  evidenceRequired: true
```

## 29.2 Deep Research

```yaml
type: DEEP_RESEARCH

requiredContext:
  - researchQuestion

sourcePolicy:
  projectScoped: true
  evidenceRequired: true
  externalSearch: false

skills:
  - create_research_plan
  - search_sources
  - collect_evidence
  - compare_sources
  - detect_contradiction
  - identify_gap
  - synthesize_findings
  - validate_evidence

tools:
  - document_fetch
  - vector_search
  - document_search
  - metadata_search
  - artifact_writer

output:
  mode: conversational_or_artifact
  evidenceRequired: true
```

## 29.3 Document-to-MD

```yaml
type: DOCUMENT_TO_MD

requiredContext:
  - sourceDocumentId

sourcePolicy:
  projectScoped: true
  evidenceRequired: false
  sourceFidelityRequired: true

skills:
  - extract_document_structure
  - convert_to_markdown
  - normalize_markdown
  - repair_section
  - validate_conversion
  - finalize_markdown

tools:
  - document_fetch
  - document_parser
  - artifact_writer
  - markdown_validator

output:
  mode: artifact
  artifactType: markdown
  requiresValidation: true
```

---

# 30. Recommended V1 Boundary

V1 nên chốt rõ:

```text
1 conversation
→ tối đa 1 active workspace
→ workspace explicit switch
→ persistent structured state
→ workspace-specific source scope
→ workspace-specific skills/tools
→ shared LLM/RAG infrastructure
→ optional artifact output
```

Không cần xây “agent operating system” ngay từ đầu.

---

# 31. Suggested Implementation Sequence

Đây chưa phải milestone plan chính thức, nhưng thứ tự kỹ thuật hợp lý là:

## Phase A — Core Workspace Engine

1. Workspace domain model.
2. Workspace Definition registry.
3. Workspace persistence.
4. active workspace reference trong conversation.
5. workspace lifecycle.
6. explicit switch/activate.
7. context builder.
8. instruction resolver.

## Phase B — Execution Integration

9. workspace-aware chat orchestration.
10. workspace-aware retrieval scope.
11. skill registry + allow-list.
12. tool policy.
13. state update contract.
14. permission revalidation.
15. evidence handling.

## Phase C — Three V1 Workspaces

16. Goal.
17. Deep Research.
18. Document-to-MD.
19. artifact support cần thiết.
20. contextual suggestions.

## Phase D — Hardening

21. integration test.
22. permission test.
23. invalid-state recovery.
24. observability.
25. definition versioning.
26. performance / token-budget tuning.

---

# 32. Testing Strategy

## Core Workspace Tests

- create workspace;
- activate workspace;
- switch workspace;
- only one active workspace;
- reactivate inactive workspace;
- state persistence;
- invalid state recovery;
- conversation reload;
- deleted workspace context;
- definition version mismatch.

## Security Tests

- cross-project source access;
- removed project member;
- deleted document;
- unauthorized document ID injected into workspace state;
- stale cached scope;
- artifact source permission.

## RAG Tests

- scope applied before retrieval;
- no result outside workspace;
- correct entity context;
- evidence links match retrieved source;
- empty evidence handling.

## Goal Tests

- current goal persisted;
- compare another goal;
- dependency with evidence;
- no fabricated relationship;
- switch goal.

## Deep Research Tests

- research plan persistence;
- multiple sub-question;
- evidence ledger;
- contradiction detection;
- incomplete research disclosure;
- synthesis references valid sources.

## Document-to-MD Tests

- heading preservation;
- list preservation;
- table preservation;
- code block preservation;
- missing section detection;
- partial repair;
- finalize artifact;
- no invented content.

---

# 33. Key Architectural Rules

Các rule sau nên được coi là nguyên tắc nền:

### Rule 1 — Workspace is context, not feature label

Không tạo workspace chỉ để đổi prompt/title.

### Rule 2 — Scope must be enforceable

Nếu source scope chỉ tồn tại trong prompt mà retrieval vẫn search toàn project thì workspace boundary chưa thực sự tồn tại.

### Rule 3 — State is structured

Không dùng conversation history làm workspace state duy nhất.

### Rule 4 — No hidden reasoning persistence

Lưu facts, selections, evidence refs, workflow status — không lưu chain-of-thought.

### Rule 5 — Skill is reusable

Skill không nên bị hard-code chỉ dùng cho một workspace nếu bản chất nó generic.

### Rule 6 — Workspace controls capability

Allowed skills/tools phải được resolve từ workspace definition.

### Rule 7 — Permission beats workspace state

State không bao giờ cấp quyền.

### Rule 8 — Explicit switch first

V1 tránh tự động thay đổi execution context.

### Rule 9 — Output contract matters

Document-to-MD chứng minh workspace không phải lúc nào cũng “chat answer”.

### Rule 10 — Workspace does not own the LLM

Workspace orchestration nằm phía trên LLM provider để giữ khả năng đổi Gemini SDK / Spring AI hoặc provider khác.

---

# 34. Các điểm cần tránh

## 34.1 Workspace = system prompt

Nếu chỉ có:

```text
"You are now a Goal Analyst"
```

thì đó là prompt preset, chưa phải workspace.

## 34.2 Workspace = command alias

Nếu `/goal` chỉ gọi `analyze_goal()` rồi kết thúc thì vẫn là skill command.

## 34.3 State = full chat history

Conversation history không thay được structured working state.

## 34.4 Search toàn project rồi bảo model “chỉ dùng Goal”

Scope phải enforce ở retrieval/tool layer.

## 34.5 Deep Research = answer dài hơn

Deep Research phải có plan, evidence collection, cross-check và synthesis.

## 34.6 Document-to-MD = LLM rewrite

Conversion mặc định cần fidelity, parser/validator và artifact handling.

## 34.7 Cho phép workspace tự bypass tool policy

Tool phải được controlled ở orchestration layer.

---

# 35. Open Design Decisions cần chốt ở Implementation Spec

Tài liệu này đề xuất direction, nhưng một số chi tiết nên chốt khi viết spec:

1. Một conversation có giữ nhiều inactive workspace instance hay chỉ snapshot state?
2. Khi activate cùng workspace type lần nữa:
   - reuse instance cũ;
   - hay tạo instance mới?
3. Workspace state lưu JSONB hay normalize?
4. Deep Research V1 có external search không?
5. Deep Research report có tạo Markdown artifact ngay V1 không?
6. Document-to-MD V1 support chính xác những file type nào?
7. Artifact persistence dùng storage hiện tại hay layer mới?
8. Workspace definition config nằm code, YAML hay DB?
9. Definition version migration xử lý ra sao?
10. Slash command parser nằm FE hay BE?
11. Suggested action được rule-based hay LLM-generated?
12. State update do deterministic service hay structured LLM output?

Khuyến nghị V1:

- definition trong code/config version-controlled;
- slash command resolve ở backend dù FE có hỗ trợ UI;
- suggested workspace/action rule-based trước;
- state update ưu tiên deterministic, chỉ dùng structured LLM output khi cần;
- external research off mặc định;
- artifact abstraction nhỏ nhưng có interface rõ.

---

# 36. Proposed V1 Product Semantics

## `/goal`

```text
"Chuyển conversation sang môi trường làm việc với một Goal cụ thể."
```

## `/deep-research`

```text
"Chuyển conversation sang môi trường nghiên cứu nhiều bước quanh một research question."
```

## `/document-to-md`

```text
"Chuyển conversation sang môi trường chuyển đổi và tinh chỉnh một document thành Markdown artifact."
```

Slash command là **entry point**, không phải định nghĩa workspace.

---

# 37. Definition of Done cho Workspace Engine V1

Workspace Engine V1 có thể coi là hoàn thiện khi:

1. Conversation có active workspace.
2. User explicit switch được workspace.
3. Workspace có persisted context/state.
4. Workspace definition resolve được instructions/skills/tools.
5. Retrieval áp dụng source scope.
6. Permission được revalidate ở mỗi execution.
7. Workspace state tồn tại qua reload.
8. Goal hoạt động multi-turn.
9. Deep Research có plan → collect → synthesize.
10. Document-to-MD tạo và refine được Markdown artifact.
11. Evidence/fidelity rules được enforce.
12. Có integration tests cho boundary và security.
13. Có logs đủ để debug workspace execution.
14. Có fallback về Default Workspace.
15. Không phụ thuộc cứng vào một LLM provider.

---

# 38. Kết luận kiến trúc

Workspace V1 nên được xây như một **orchestration/context layer** nằm giữa Conversation và các execution capability như RAG, skill, tool, artifact và LLM.

Mô hình cuối:

```text
Conversation
    ↓
Active Workspace
    ↓
Workspace Definition + Runtime State
    ↓
Scope + Context + Rules + Skills + Tools
    ↓
Workspace-specific Execution Pipeline
    ↓
RAG / Tool / Artifact / LLM
    ↓
Response
```

Ba workspace V1 đại diện cho ba pattern khác nhau:

```text
GOAL
→ entity-centric analytical workspace

DEEP_RESEARCH
→ multi-source, multi-step evidence workspace

DOCUMENT_TO_MD
→ stateful transformation + artifact workspace
```

Nếu core Workspace Engine hỗ trợ tốt cả ba pattern này thì abstraction đã đủ khỏe để mở rộng sang nhiều workspace tương lai mà không phải thiết kế lại nền tảng.

---

# 39. Short Architecture Summary

Có thể dùng đoạn sau để align team:

> Workspace là execution context có trạng thái của AI Chatbot. Mỗi workspace định nghĩa source scope, working context, state, instruction, allowed skills/tools và output contract. Slash command như `/goal`, `/deep-research`, `/document-to-md` chỉ là entry point để activate workspace. Sau khi được activate, các message tiếp theo được xử lý trong workspace đó cho tới khi user switch/exit. Workspace không thay thế Conversation, Project, Skill hay RAG; nó orchestration các thành phần đó để AI làm việc nhất quán trong một phạm vi và workflow cụ thể.

