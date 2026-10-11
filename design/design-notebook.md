# Relay Design Notebook

This notebook records the main design questions, alternatives, and decisions that came up while developing Relay.

---

## 2026-10-05 — Narrowing the problem

I started with the general problem of student-club handoffs. At first, the obvious issue seemed to be that new officers have to relearn recurring work when the previous officer leaves.

Two examples made the problem more interesting:

- A new competition chair needed a roughly two-hour call with the previous chair to reconstruct the registration process.
- In another club, an old assumption about storage survived across several leadership changes even though the renewal policy had changed. The update disappeared, while the outdated information kept being trusted.

The second case changed how I thought about the project. The problem is not only that knowledge disappears. A handoff can also fail because **outdated knowledge survives**.

I decided that Relay should address both cases rather than being a generic documentation app.

Relevant work:
- `problem-framing.md`

---

## 2026-10-05 — Choosing the core product idea

I considered a general club wiki or handoff system, but that felt too close to Google Docs, Notion, or Confluence.

The more specific opportunity was to connect a recurring procedure to the assumptions, rules, and experience behind it.

I settled on three main product ideas:

1. **Quick Playbooks** for short, ordered recurring procedures.
2. **Context Notes** for the few explanations or assumptions worth preserving.
3. **Change Check** so that when shared context changes, the procedures that relied on it are surfaced for review.

An important scope decision was that Relay should **not automatically monitor university policies or decide that a procedure is wrong**. An officer learns about a change through normal work and records it. Relay only makes the consequences of that known change visible.

This kept the product realistic and made human judgment part of the design.

Relevant work:
- `application-pitch.md`

---

## 2026-10-05 — Making Context Notes optional

One early version risked requiring officers to explain the reason behind every Step.

I rejected that because it would make maintaining Relay feel like extra paperwork. Most procedural steps do not need an explanation.

I changed the idea so that a Context Note is optional and is only added when losing the explanation would matter later.

Examples:
- an outside rule,
- a workaround learned from experience,
- a decision the club intentionally made.

This also helped distinguish Relay from simply making a more detailed checklist.

---

## 2026-10-06 — Deciding how questions should work

I wanted members to be able to point out confusing parts of a Playbook without letting everyone rewrite the club's official procedure.

I considered comments or discussion threads, but they added a lot of behavior that was not central to the problem.

I chose a much smaller model:

- any member can ask a Question,
- a Question attaches to one specific Step or Context Note,
- only an officer can record the official answer.

The useful design point is that the uncertainty stays next to the knowledge that caused it instead of disappearing into a message thread.

This also led to an important implementation decision: Steps should keep stable identities even when reordered or removed, so old Questions can continue to refer to the same Step.

---

## 2026-10-06 — Separating the concepts

The hardest concept-design question was where the "needs review" behavior should live.

One simple option was to put Context Notes and a `needsReview` flag directly inside each Playbook. I rejected that because `ProcedureMaintaining` would then be responsible for:
- storing the procedure,
- storing its reasons,
- detecting affected procedures,
- and managing review state.

Instead, I separated the design into five concepts:

- `ProcedureMaintaining`
- `ContextTracking`
- `Questioning`
- `Reviewing`
- `RoleBasedAccessing`

The important split is between `ContextTracking` and `Reviewing`.

`ContextTracking` knows which Procedures depend on a piece of context. `Reviewing` only knows that an Item needs reconsideration for some Cause. A reaction connects the two when context meaningfully changes.

This became the main conceptual idea of the design.

Relevant work:
- `concept-design.md`

---

## 2026-10-06 — Playbook vs. Procedure terminology

The interface naturally uses the word **Playbook**, while the formal concept specification uses `Procedure`.

I initially left the mapping implicit, but that made the document slightly harder to follow.

I added an explicit sentence near the beginning of the concept design:

> In the Relay interface, users see a Playbook; in the formal concept specification, that same object is modeled as a `Procedure`.

This was a small change, but it made the formal specification easier to connect back to the product.

---

## 2026-10-07 — Correction versus real change

A key question was whether every Context Note edit should trigger review.

That would create unnecessary work. Fixing a typo is very different from changing the fact that the note represents.

I added two separate actions:

- `correct` — wording changes without a change in meaning
- `change` — underlying information has changed

Only `change` creates review work.

I later added a guard requiring the new text to differ from the current text, so a no-op "change" cannot accidentally create a Change record and review.

This distinction also became visible in the UI as:

- **Save as correction**
- **Save as changed information**

---

## 2026-10-07 — Preserving why a Review exists

A specification review exposed an important gap.

Originally, `ContextTracking.change` replaced the old Context Note text, and `Reviewing` only stored the Procedure and review status.

That meant the UI could claim to show:

- the old context,
- the new context,
- and why the Playbook was surfaced,

but the concept state did not actually preserve enough information to do that.

I fixed this by adding persistent `Change` records to `ContextTracking`:

- Context Note
- before text
- after text

`Reviewing` now stores a generic `Cause`, with Relay instantiating:

`Cause = ContextTracking.Change`

This preserves traceability without making `Reviewing` understand Context Notes.

This was a useful reminder that anything promised by the UI needs to be supported by actual concept state.

---

## 2026-10-07 — Multiple changes before one review

Another edge case came up:

What if a Procedure is already waiting for review and another piece of context changes before an officer handles it?

Creating a second Review would clutter the review queue, but ignoring the second change would lose information.

I changed `Reviewing` so that each open Review stores a **set of causes**.

- `request` creates the first Review and initializes the cause set.
- `addCause` adds another Cause to an existing unfinished Review.

Two reactions handle the mutually exclusive cases.

The result is:
- one unfinished Review per Procedure,
- but every distinct change that caused the review is preserved.

I liked this better because it keeps the UI simple without throwing away traceability.

---

## 2026-10-07 — Permissions and official knowledge

I decided that Playbooks and Context Notes belong to the **club workspace**, not to the officer who created them.

That matches the purpose of Relay: the knowledge should survive the person who entered it.

The permissions became:

- Members and Officers can ask Questions.
- Only Officers can create or edit Procedures and Context Notes.
- Only Officers can answer Questions.
- Only Officers can complete Reviews.

I kept authorization in a separate `RoleBasedAccessing` concept rather than adding user fields and permission logic to every other concept.

---

## 2026-10-08 — Designing the UI sketches

I built the sketches as low-fidelity HTML pages first so each screen would resemble the layout of the eventual site rather than four isolated boxes.

I focused on four views:

1. Club workspace / Playbooks
2. Playbook detail
3. Editing a Context Note
4. Reviewing an affected Playbook

The main visual flow is:

`Playbook -> shared context changes -> dependent Playbooks surface -> officer reviews`

I intentionally kept the sketches grayscale and simple because the assignment is about interaction and structure rather than visual polish.

Relevant exports:
- `club-workspace.png`
- `playbook-detail.png`
- `context-change.png`
- `review-playbook.png`

---

## 2026-10-08 — UI and concept coherence fixes

Reviewing the sketches against the concept design exposed several small inconsistencies.

### Question authorship

The Playbook sketch originally said **"Asked by Maya."**

The `Questioning` concept did not store an asker, and authorship was not important enough to justify expanding the concept just for that label.

I removed the label instead of adding unnecessary state.

### Questions on Context Notes

The concept and pitch both allowed Questions on Context Notes, but the original sketch only visibly showed asking about a Step.

I added **"Ask about this note"** to a Context Note so the sketch now demonstrates both uses.

### Review count

A shared Context Note was used by three Playbooks, so changing it should surface all three.

I updated the workspace sketch to show `Reviews 3`, while highlighting Book Rehearsal Space as one of those reviews.

### Human judgment

An early affected Step was obviously wrong after the context change, which made the "Confirm it is still current" option feel fake.

I changed the Step to:

> Check that the rehearsal-space renewal is current before the semester begins.

Now the new semester-renewal rule makes the Step worth reconsidering, but the Step may still be correct. This better illustrates why Relay surfaces a review rather than automatically rewriting the procedure.

---

## 2026-10-08 — Building one continuous example

Earlier documents used several examples, including storage renewal and rehearsal-space renewal.

The different examples were individually valid, but the submission felt more coherent when the same example carried through the concept explanation, UI, and journey.

I standardized the main design example around:

> MIT event spaces previously needed renewal once per academic year, but now require renewal each semester.

This Context Note is shared by multiple Procedures. One update therefore demonstrates the exact behavior Relay is designed around.

The real storage example remains in the problem framing as evidence for the underlying problem.

---

## 2026-10-08 — Writing the user journey

I initially worried that the user journey could turn into a list of clicks.

I structured it around one incoming officer and one failure mode instead:

1. Tiya inherits a Playbook that looks trustworthy.
2. The procedure survives, but one assumption becomes false.
3. She records the changed shared Context Note once.
4. Relay surfaces the Procedures that depended on it.
5. She reviews the affected Procedure rather than letting Relay guess.

The important sentence for me was the realization that the dangerous case is not only when documentation disappears:

> The Playbook itself has not disappeared; in fact, that is what makes the situation dangerous.

That connects the stakeholder's experience directly to the design decision behind Change Check.

Relevant work:
- `user-journey.md`

---

## 2026-10-08 — Final submission cleanup

Before finishing P1, I checked the pitch, concepts, sketches, and journey against one another rather than reviewing them independently.

Some final fixes included:

- explicitly mapping Playbook to `Procedure`,
- using the same room-renewal example through the main design story,
- preserving before/after context in `Change`,
- allowing Reviews to accumulate multiple Causes,
- preventing same-text changes from creating reviews,
- showing Context Note Questions in the sketch,
- removing unsupported question authorship,
- clarifying that the workspace shows a review count of three while highlighting one item,
- removing references to prototype HTML pages that were no longer part of the public submission.

The README now serves as the single entry point to the design.

Relevant work:
- `README.md`
- `problem-framing.md`
- `application-pitch.md`
- `concept-design.md`
- `user-journey.md`

---

## Things I want to remember for the final reflection

A few lessons from P1 that seem worth revisiting in P5:

- **The hardest design problem was not adding features; it was assigning responsibility.** Separating context changes from review state made the design much easier to reason about.
- **UI promises create specification obligations.** Showing "before" and "after" in the Review UI exposed that the original model did not preserve the old text.
- **Edge cases can improve the abstraction.** The multiple-change case led to `Review.causes`, which is both more precise and simpler for the user than duplicate Reviews.
- **Not every inconsistency should be solved by adding state.** Removing "Asked by Maya" was better than adding question authorship just to support a nonessential UI label.
- **Human judgment is part of the product boundary.** Relay should expose what might be stale, not pretend to know whether a club's procedure is correct.
- **A coherent example helps explain a formal design.** Reusing the room-renewal story made the concepts, UI, and journey easier to connect.
- The central idea I want to keep testing is: **a handoff can fail because knowledge disappears, but it can also fail because outdated knowledge survives.**