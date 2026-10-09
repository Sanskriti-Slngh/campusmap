# Concept design for Relay

In the Relay interface, users see a **Playbook**; in the formal concept specification, that same object is modeled as a `Procedure`.

The hard part of Relay is not storing a procedure. It is handling the moment when one piece of context changes and several inherited procedures may suddenly be wrong.

Suppose three club procedures all rely on the same fact: **storage space must be renewed annually**. If that rule changes, an officer should not have to remember every place where the old assumption was used. Relay keeps the procedure, its shared context, and the act of reconsidering it separate:

```text
ContextNote: "Storage must be renewed annually"
        |                    |
        v                    v
  Procedure A          Procedure B
        \                /
         \              /
          -- context changes --
                    |
                    v
             reviews requested
```

That separation is the main design idea. Relay uses five concepts: **ProcedureMaintaining**, **ContextTracking**, **Questioning**, **Reviewing**, and **RoleBasedAccessing**. Each concept owns one course of action, and reactions provide the application-specific links between them.

# UI Sketches

These sketches are intentionally low fidelity: gray boxes, simple labels, and short callouts that explain the key behaviors without polishing the interface. They focus on the essential interactions that make Relay distinct: shared playbooks, context notes, questions, and review after a meaningful context change. The linked, multi-page Relay UI prototype includes the [Club workspace](./index.html), [Playbook detail](./playbook.html), [Context change](./context-change.html), and [Review](./review.html) screens.

### Club workspace / Playbook list

The workspace presents the club's shared playbooks and surfaces a procedure that needs review after a relevant context change. Members can read the playbook list; officers can create and edit official procedures.

### Playbook detail

The playbook detail screen combines an ordered procedure, optional context notes, and questions attached to the exact step or note that needs clarification. A question belongs to the relevant Step identity even if the procedure later changes.

### Context change

When an officer records changed information, the app distinguishes between a simple wording correction and a meaningful underlying fact change. Only the latter triggers review for dependent playbooks.

### Review affected playbook

The review screen explains why the playbook appeared in review, identifies the affected step, and gives the officer a chance to confirm the procedure or update it and mark the review resolved.

## ProcedureMaintaining

```text
concept ProcedureMaintaining

types
  external Space

  ProcedureStatus is ACTIVE or RETIRED
  StepStatus is ACTIVE or REMOVED

purpose
  preserve a reusable ordered procedure so recurring work does not
  have to be reconstructed each time

principle
  A recurring process is recorded as an ordered set of steps. Later,
  the same procedure can be reused and revised instead of reconstructed
  from scratch, while each step keeps a stable identity as the ordering
  and contents change.

state
  a set of Procedures with
    a space Space
    a title String
    a status ProcedureStatus

  a set of Steps with
    a procedure Procedure
    a text String
    an optional position Number
    a status StepStatus

  Rule:
    every ACTIVE Step has a positive integer position

  Rule:
    every REMOVED Step has no position

  Rule:
    for every Procedure with n ACTIVE Steps, their positions are
    exactly 1 through n

actions

  create (
    space: Space,
    title: String
  ) : returns (procedure: Procedure)
    then
      create a new Procedure with given space and title
      set procedure status to ACTIVE
      return procedure

  addStep (
    procedure: Procedure,
    text: String
  ) : returns (step: Step)
    where
      procedure exists
      procedure status is ACTIVE
    then
      create a new Step for procedure with given text
      set step status to ACTIVE
      set step position to one more than the number of ACTIVE Steps
        belonging to procedure
      return step

  editStep (
    step: Step,
    text: String
  )
    where
      step exists
      step status is ACTIVE
      step's procedure exists
      step's procedure status is ACTIVE
    then
      replace step text with text

  moveStep (
    step: Step,
    position: Number
  )
    where
      step exists
      step status is ACTIVE
      step's procedure exists
      step's procedure status is ACTIVE
      1 <= position <= number of ACTIVE Steps belonging to step's procedure
    then
      move step to given position
      shift the other ACTIVE Step positions as needed so the positions
        remain exactly 1 through n

  removeStep (
    step: Step
  )
    where
      step exists
      step status is ACTIVE
      step's procedure exists
      step's procedure status is ACTIVE
    then
      set step status to REMOVED
      remove step position
      shift later ACTIVE Step positions down by one

  rename (
    procedure: Procedure,
    title: String
  )
    where
      procedure exists
      procedure status is ACTIVE
    then
      replace procedure title with title

  retire (
    procedure: Procedure
  )
    where
      procedure exists
      procedure status is ACTIVE
    then
      set procedure status to RETIRED
```

Removing a Step preserves its identity instead of deleting it. Old Questions can therefore continue to refer to the Step even though it is no longer part of the current procedure.

---

## ContextTracking

```text
concept ContextTracking

types
  external Scope
  external Subject
  external Anchor

purpose
  preserve shared context together with the subjects that depend on it,
  so the impact of a later change can be identified

principle
  A piece of context is recorded once and attached to the subjects that
  rely on it. If the underlying information later changes, the shared
  context is changed once while its attachments continue to identify
  every subject that depended on it.

state
  a set of ContextNotes with
    a scope Scope
    a text String

  a set of Attachments with
    a context ContextNote
    a subject Subject
    an optional anchor Anchor

  a set of Changes with
    a context ContextNote
    a before String
    an after String

  Rule:
    there is at most one Attachment with the same context, subject,
    and optional anchor

actions

  record (
    scope: Scope,
    text: String
  ) : returns (context: ContextNote)
    then
      create a new ContextNote with given scope and text
      return context

  attach (
    context: ContextNote,
    subject: Subject
  ) : returns (attachment: Attachment)
    where
      context exists
      no Attachment already connects context and subject without an anchor
    then
      create an Attachment connecting context and subject
      return attachment

  attachAt (
    context: ContextNote,
    subject: Subject,
    anchor: Anchor
  ) : returns (attachment: Attachment)
    where
      context exists
      no Attachment already connects context, subject, and anchor
    then
      create an Attachment connecting context, subject, and anchor
      return attachment

  detach (
    attachment: Attachment
  )
    where
      attachment exists
    then
      remove attachment

  correct (
    context: ContextNote,
    text: String
  )
    where
      context exists
    then
      replace context text with text

  change (
    context: ContextNote,
    text: String
  ) : returns (change: Change)
    where
      context exists
      text is not context's current text
    then
      create a new Change with
        context set to context
        before set to context's current text
        after set to text
      replace context text with text
      return change
```

`correct` and `change` both replace the current text, but only `change` preserves a `Change` record containing the previous and new text. Fixing **“renewd annually”** to **“renewed annually”** is a correction. Replacing **“renewal is not required”** with **“renewal is required annually”** is a change. Only the second action should make dependent procedures worth reviewing.

ContextNotes are not deleted when they stop applying. Their Attachments are detached instead, preserving the identity for old Questions and history.

---

## Questioning

```text
concept Questioning

types
  external Subject

  QuestionStatus is OPEN or ANSWERED

purpose
  preserve the resolution of uncertainty so the same ambiguity does not
  have to be rediscovered and resolved repeatedly

principle
  A question is raised about a particular subject. Once an answer is
  recorded, the uncertainty and its resolution remain together for
  later readers.

state
  a set of Questions with
    a subject Subject
    a text String
    a status QuestionStatus
    an optional answer String

actions

  ask (
    subject: Subject,
    text: String
  ) : returns (question: Question)
    then
      create a new Question with given subject and text
      set question status to OPEN
      return question

  answer (
    question: Question,
    answer: String
  )
    where
      question exists
      question status is OPEN
    then
      associate answer with question
      set question status to ANSWERED
```

Relay uses the same generic Questioning concept twice: once with `Subject = Step` and once with `Subject = ContextNote`. For readability, the reactions below call these instances **StepQuestioning** and **ContextQuestioning**. This is one reusable concept specification with two type instantiations, not two additional concepts.

For example, a member can ask **“Do we still upload music before paying?”** on the exact Step that says to upload the music. The Question stays attached to that Step, and only an officer can record the official answer.

---

## Reviewing

```text
concept Reviewing

types
  external Item
  external Cause

  ReviewStatus is PENDING or CHANGES_REQUIRED or COMPLETE or CANCELLED

purpose
  ensure questionable information is explicitly reconsidered for known
  reason before it continues to be treated as current

principle
  A review records an item together with the reasons it needs
  reconsideration. The item can then be confirmed as current, marked as
  requiring changes, resolved after those changes, or removed from
  consideration if the review is no longer relevant.

state
  a set of Reviews with
    an item Item
    a causes set of Cause
    a status ReviewStatus

  Rule:
    for each Item, at most one Review has status PENDING or CHANGES_REQUIRED

actions

  request (
    item: Item,
    cause: Cause
  ) : returns (review: Review)
    where
      item has no Review with status PENDING or CHANGES_REQUIRED
    then
      create a new Review for item
      set review causes to {cause}
      set review status to PENDING
      return review

  addCause (
    review: Review,
    cause: Cause
  )
    where
      review exists
      review status is PENDING or CHANGES_REQUIRED
      cause is not already in review causes
    then
      add cause to review causes

  confirm (
    review: Review
  )
    where
      review exists
      review status is PENDING
    then
      set review status to COMPLETE

  requireChanges (
    review: Review
  )
    where
      review exists
      review status is PENDING
    then
      set review status to CHANGES_REQUIRED

  resolve (
    review: Review
  )
    where
      review exists
      review status is CHANGES_REQUIRED
    then
      set review status to COMPLETE

  cancel (
    review: Review
  )
    where
      review exists
      review status is PENDING or CHANGES_REQUIRED
    then
      set review status to CANCELLED
```

---

## RoleBasedAccessing

```text
concept RoleBasedAccessing

types
  external User
  external Resource
  external Permission
  external Command

purpose
  simplify assignment of user permissions

principle
  A role is created, given permissions, and assigned to users for a
  resource. Later, a command is authorized only when one of the user's
  roles for that resource contains the required permission.

state
  a set of Roles with
    a unique name String
    a permissions set of Permission

  a set of Assignments with
    a user User
    a resource Resource
    a role Role

    unique user and resource and role

actions

  createRole (
    name: String
  ) : returns (role: Role)
    where
      no Role already has given name
    then
      create a new Role with given name and no permissions
      return role

  deleteRole (
    role: Role
  )
    where
      role exists
      no Assignment uses role
    then
      remove role

  permit (
    role: Role,
    permission: Permission
  )
    where
      role exists
      permission is not already in role permissions
    then
      add permission to role permissions

  revokePermission (
    role: Role,
    permission: Permission
  )
    where
      role exists
      permission is in role permissions
    then
      remove permission from role permissions

  assign (
    user: User,
    resource: Resource,
    role: Role
  ) : returns (assignment: Assignment)
    where
      role exists
      no Assignment already connects user, resource, and role
    then
      create a new Assignment connecting user, resource, and role
      return assignment

  unassign (
    assignment: Assignment
  )
    where
      assignment exists
    then
      remove assignment

  authorize (
    user: User,
    resource: Resource,
    permission: Permission,
    command: Command
  ) : returns (authorized: Command)
    where
      some Assignment connects user and resource to a role
      permission is in that role's permissions
    then
      return command as authorized
```

`authorize` is a significant action even though it does not modify state: its occurrence means that the command passed the role-based permission check. `Command` is opaque to RoleBasedAccessing.

Relay configures the familiar roles this way:

| Permission | Member | Officer |
| --- | :---: | :---: |
| `ASK_QUESTION` | ✓ | ✓ |
| `EDIT_PROCEDURE` | — | ✓ |
| `EDIT_CONTEXT` | — | ✓ |
| `ANSWER_QUESTION` | — | ✓ |
| `COMPLETE_REVIEW` | — | ✓ |

An officer is still a club member, so officers can ask questions too. Their role adds the ability to maintain and authoritatively answer the club's official knowledge.

---

# Essential reactions

Relay does not expose protected knowledge-changing actions directly to users. A protected user operation first occurs as `RoleBasedAccessing.authorize`; only an authorized command can trigger the corresponding concept action.

## A meaningful context change records its before/after text and updates one open review per dependent procedure

```text
when
  change = ContextTracking.change(context, text)

where
  ContextTracking:
    an Attachment connects context to procedure

  ProcedureMaintaining:
    procedure status is ACTIVE

  Reviewing:
    procedure has no Review with status PENDING or CHANGES_REQUIRED

then
  Reviewing.request(procedure, change)
```

```text
when
  change = ContextTracking.change(context, text)

where
  ContextTracking:
    an Attachment connects context to procedure

  ProcedureMaintaining:
    procedure status is ACTIVE

  Reviewing:
    existingReview item is procedure
    existingReview status is PENDING or CHANGES_REQUIRED
    change is not already in existingReview causes

then
  Reviewing.addCause(existingReview, change)
```

These are two separate reactions for the two mutually exclusive cases. Each applies once for each distinct active Procedure attached to the changed ContextNote. The returned `Change` preserves the prior and new text. If the Procedure has no unfinished Review, `request` opens one with that Change as its first cause; otherwise `addCause` appends the Change to its existing Review. Thus one Procedure has at most one unfinished Review, while that Review can preserve every distinct reason it needs reconsideration. The Review can explain why it exists and display each exact before/after text without `Reviewing` knowing anything about ContextNotes. One shared change can update several affected Procedures' reviews without `ContextTracking` knowing anything about reviews.

## Removing a step removes its current anchored context

```text
when
  ProcedureMaintaining.removeStep(step)

where
  ContextTracking:
    attachment has anchor step

then
  ContextTracking.detach(attachment)
```

The Step identity remains for historical Questions, but it no longer acts as a current location for ContextNotes.

## Retiring a procedure cancels unfinished review work

```text
when
  ProcedureMaintaining.retire(procedure)

where
  Reviewing:
    review item is procedure
    review status is PENDING or CHANGES_REQUIRED

then
  Reviewing.cancel(review)
```

## Only officers may attach ContextNotes to official procedures

```text
when
  RoleBasedAccessing.authorize(
    user,
    workspace,
    EDIT_CONTEXT,
    ATTACH_CONTEXT_AT(context, procedure, step)
  )

where
  ContextTracking:
    context scope is workspace

  ProcedureMaintaining:
    procedure space is workspace
    procedure status is ACTIVE
    step procedure is procedure
    step status is ACTIVE

then
  ContextTracking.attachAt(context, procedure, step)
```

The same pattern applies to procedure-level `ContextTracking.attach`. Keeping this constraint in the reaction lets `ContextTracking` stay generic while Relay guarantees that the ContextNote, Procedure, and Step belong together.

## Any club member may ask a question about a step

```text
when
  RoleBasedAccessing.authorize(
    user,
    workspace,
    ASK_QUESTION,
    ASK_STEP(step, text)
  )

where
  ProcedureMaintaining:
    step procedure is procedure
    procedure space is workspace
    procedure status is ACTIVE
    step status is ACTIVE

then
  StepQuestioning.ask(step, text)
```

## Any club member may ask a question about a ContextNote

```text
when
  RoleBasedAccessing.authorize(
    user,
    workspace,
    ASK_QUESTION,
    ASK_CONTEXT(context, text)
  )

where
  ContextTracking:
    context scope is workspace
    context is attached to procedure

  ProcedureMaintaining:
    procedure status is ACTIVE

then
  ContextQuestioning.ask(context, text)
```

Both the **Member** and **Officer** roles contain `ASK_QUESTION`.

## Only officers may answer a step question

```text
when
  RoleBasedAccessing.authorize(
    user,
    workspace,
    ANSWER_QUESTION,
    ANSWER_STEP(question, answer)
  )

where
  StepQuestioning:
    question subject is step
    question status is OPEN

  ProcedureMaintaining:
    step procedure is procedure
    procedure space is workspace

then
  StepQuestioning.answer(question, answer)
```

## Only officers may answer a ContextNote question

```text
when
  RoleBasedAccessing.authorize(
    user,
    workspace,
    ANSWER_QUESTION,
    ANSWER_CONTEXT(question, answer)
  )

where
  ContextQuestioning:
    question subject is context
    question status is OPEN

  ContextTracking:
    context scope is workspace

then
  ContextQuestioning.answer(question, answer)
```

Only the **Officer** role contains `ANSWER_QUESTION`.

## Only officers may create and edit official procedures

```text
when
  RoleBasedAccessing.authorize(
    user,
    workspace,
    EDIT_PROCEDURE,
    CREATE_PROCEDURE(title)
  )

then
  ProcedureMaintaining.create(workspace, title)
```

```text
when
  RoleBasedAccessing.authorize(
    user,
    workspace,
    EDIT_PROCEDURE,
    EDIT_STEP(step, text)
  )

where
  ProcedureMaintaining:
    step procedure is procedure
    step status is ACTIVE
    procedure space is workspace
    procedure status is ACTIVE

then
  ProcedureMaintaining.editStep(step, text)
```

## Protected operations require authorization

Each protected operation occurs only after `RoleBasedAccessing.authorize` succeeds for the listed permission and its Relay-specific workspace/subject constraints hold. The target concept's stated action guards must also hold. For actions whose arguments vary by subject type, Relay invokes the corresponding `Questioning` instance.

| Protected operation | Permission | Authorized concept action |
| --- | --- | --- |
| Create a Procedure | `EDIT_PROCEDURE` | `ProcedureMaintaining.create` |
| Add a Step | `EDIT_PROCEDURE` | `ProcedureMaintaining.addStep` |
| Edit a Step | `EDIT_PROCEDURE` | `ProcedureMaintaining.editStep` |
| Move a Step | `EDIT_PROCEDURE` | `ProcedureMaintaining.moveStep` |
| Remove a Step | `EDIT_PROCEDURE` | `ProcedureMaintaining.removeStep` |
| Rename a Procedure | `EDIT_PROCEDURE` | `ProcedureMaintaining.rename` |
| Retire a Procedure | `EDIT_PROCEDURE` | `ProcedureMaintaining.retire` |
| Record a ContextNote | `EDIT_CONTEXT` | `ContextTracking.record` |
| Attach a ContextNote to a Procedure or Step | `EDIT_CONTEXT` | `ContextTracking.attach` or `ContextTracking.attachAt` |
| Detach a ContextNote attachment | `EDIT_CONTEXT` | `ContextTracking.detach` |
| Correct a ContextNote's wording | `EDIT_CONTEXT` | `ContextTracking.correct` |
| Record a meaningful ContextNote change | `EDIT_CONTEXT` | `ContextTracking.change` |
| Ask about a Step or ContextNote | `ASK_QUESTION` | `StepQuestioning.ask` or `ContextQuestioning.ask` |
| Answer a Step or ContextNote question | `ANSWER_QUESTION` | `StepQuestioning.answer` or `ContextQuestioning.answer` |
| Confirm a Review | `COMPLETE_REVIEW` | `Reviewing.confirm` |
| Mark a Review as requiring changes | `COMPLETE_REVIEW` | `Reviewing.requireChanges` |
| Resolve a Review after updating the Procedure | `COMPLETE_REVIEW` | `Reviewing.resolve` |

For example, `ProcedureMaintaining.moveStep` still requires an active Step and Procedure and a valid position; authorization does not bypass those concept guards. Similarly, ContextNote attachments must connect objects in the same workspace, questions must target an active subject in that workspace, and review actions must target an active Procedure in the officer's workspace.

---

# Role of the concepts in Relay

`ProcedureMaintaining` owns the official procedure and stable Step identities; its `Space` is a club workspace. `ContextTracking` owns optional shared ContextNotes, dependency links from those notes to Procedures or particular Steps, and persistent `Change` records containing before/after text; its `Scope` is the same workspace, `Subject` is a Procedure, and `Anchor` is a Step. Procedures and ContextNotes belong to the workspace rather than to the officer who created them, so they survive leadership turnover.

`Questioning` is reused twice, with `Subject = Step` and `Subject = ContextNote`. This avoids giving Questioning knowledge of either kind of subject while still allowing members to point at exactly what is unclear. `Reviewing` is instantiated with `Item = Procedure` and `Cause = ContextTracking.Change`; it owns the lifecycle of reconsideration and a set of generic reasons for each review, without knowing the causes' internal structure.

`RoleBasedAccessing` uses Relay users as `User`, club workspaces as `Resource`, and permissions such as `ASK_QUESTION`, `ANSWER_QUESTION`, `EDIT_PROCEDURE`, `EDIT_CONTEXT`, and `COMPLETE_REVIEW`. Both Member and Officer roles permit asking questions; only Officer permits answering questions or changing official knowledge. Role creation and role assignment are workspace administration, not ordinary member operations. Members and officers read Procedures and ContextNotes through state queries; those reads are not modeled as actions because they do not represent significant behavioral events.

The difficult part of Relay is the stale-knowledge case shown at the start: **one piece of context is shared by several procedures and later changes**. A more tangled design could copy the context into every Procedure, add a `needsReview` flag to each one, and make ProcedureMaintaining responsible for deciding when that flag changes. Relay does not do that. `ContextTracking` records the dependency once and preserves each meaningful before/after change; `Reviewing` owns reconsideration and stores a set of generic causes per open review; one `ContextTracking.change -> Reviewing.request/addCause` reaction links them. The `correct`/`change` distinction further prevents a wording fix from creating unnecessary review work.

Each `Review.causes` entry refers to an exact `ContextTracking.Change`, which in turn identifies the ContextNote and its before/after text. The reasons, changed notes, old text, and new text remain available to explain a Review without coupling `Reviewing` to context. If several facts change before an officer reviews a Procedure, Relay surfaces that Procedure once and retains every distinct cause. That choice of responsibilities keeps the difficult behavior small and traceable: **one or more shared facts change, and each dependent Procedure has one review with all of its reasons attached.**
