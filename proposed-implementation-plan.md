# Implementation Plan — Liaison Tool Reimplementation

This plan describes how the reimplemented liaison application is built, migrated to, and
validated. It is the companion to [proposed-README.md](README.md), which describes
*what* the tool does, and to [proposed-issues.md](proposed-issues.md), which carries the
requirements issue by issue.

---

## 1. What "reimplementation" means here

The liaison application is rebuilt: new models, new forms, new views, new templates, new
email layer. The existing `ietf/liaisons` implementation is not incrementally edited into
shape.

**In scope**

- Purpose-built models for peer organizations, peer groups, formal liaison relationships, and
  Liaison Manager appointments — replacing the use of `Group` and `Role` for external SDOs
  (ISSUE-1).
- A statement model implementing the RFC 4053bis §2.1 contact model directly, with no ad-hoc
  contact fields (ISSUE-20, ISSUE-24).
- An event-driven workflow engine: draft → pending → posted, with approval, review, send-back,
  and approval-invalidation recorded as events (ISSUE-5, ISSUE-12, ISSUE-13).
- Liaison Manager authority answered from those appointments rather than from external-group
  roles. The IAB Liaison Coordinator role stays a `Role` in the `iab` group and is explicitly
  out of scope for the move (ISSUE-1).
- Management UI for every object in the new models (ISSUE-2).
- Migration of all existing statements, attachments, events, and SDO group records, with
  history preserved event for event (ISSUE-39).

**Out of scope**

- Changing how IETF-side bodies are modelled. WGs, Areas, the IETF, the IESG, and the IAB
  remain datatracker `Group` records with their existing roles.
- Changing the datatracker's mail infrastructure beyond what the alias work requires
  (ISSUE-31).
- Any redesign of the public statement page beyond what these issues call for — the listing
  question in §6 is the one deliberate exception.

**Compatibility surfaces that must survive the rewrite**

| Surface | Requirement |
|---------|-------------|
| Public statement URLs | Existing `/liaison/<id>/` URLs continue to resolve to the same statement |
| Attachment URLs | Legacy `www.ietf.org/lib/dt/documents/LIAISON/` paths redirect to blobstore-served files (ISSUE-37) |
| `/api/v1` | Continues to serve statements; `draft` and `dead` statements excluded. Field-level changes are expected and must be announced, since the ad-hoc contact fields do not exist in the new schema |
| Statement history | Every event on every existing statement is carried across, with original attribution and timestamps |

---

## 2. Phases

Phases are sequenced so that each one ends with something demonstrable. The milestone labels
in [proposed-issues.md](proposed-issues.md) map onto these phases.

### P0 — Foundation: the peer organization domain (milestone M0)

Build the models the rest of the application depends on: peer organizations, peer groups,
liaison relationships, and Liaison Manager appointments; plus the management UI and the
authorization helpers that answer "who is the Liaison Manager for this group?" The
coordinator-side answer — "is this person an IAB Liaison Coordinator?" — continues to come from
the `iab` group role, so the helpers read from two sources and the tests must cover both.

Issues: ISSUE-1, ISSUE-2, ISSUE-3.

Exit criteria: the new models are populated from a trial migration of the existing SDO
`Group` and `liaiman` `Role` data; a Liaison Coordinator can create and edit an organization,
its groups, a relationship, and a manager appointment through the UI; authorization helpers
pass unit tests against real appointment data and against real `iab` group roles; appointing a
new Liaison Coordinator still requires no change in the liaison application.

### P1 — Statement model and lifecycle (milestone M1)

The statement model with the RFC 4053bis contact fields, the state machine, and the event
log. Approval, review, send-back, approval invalidation, auto-post, drafts, dead-state
handling.

Issues: ISSUE-4, ISSUE-5, ISSUE-6, ISSUE-7, ISSUE-8, ISSUE-9, ISSUE-10, ISSUE-11,
ISSUE-12, ISSUE-13, ISSUE-14.

Exit criteria: a statement can be walked through every path in the README's flowcharts in a
test environment — including send-back, re-approval after a content edit, multi-group
targeting with Coordinator review, and back-entry of an incoming statement with a past
received date.

### P2 — Roles, permissions, and dashboards (milestone M2)

Acting-role capture, act-on-behalf, approval authority enforcement, action-taken authority,
and surfacing pending work to the people who owe it.

Issues: ISSUE-15, ISSUE-16, ISSUE-17, ISSUE-18, ISSUE-19.

Exit criteria: every role-restricted action is covered by a permission test; an AD sees
pending liaison approvals on their dashboard; LMs and LCs see their review queues.

### P3 — Contact fields, targeting, and purpose (milestone M3)

The addressing model as users experience it: field population, per-group To-Contact lists,
one-organization/many-groups targeting, purpose values.

Issues: ISSUE-20, ISSUE-21, ISSUE-22, ISSUE-23, ISSUE-24, ISSUE-25.

Exit criteria: entering a statement prepopulates the right addresses for the acting role with
no manual retyping; the statement page shows the statement's own fields distinctly from any
generated email.

### P4 — Email (milestone M4)

The email layer: From/Reply-To policy, the mandatory coordination CC, the notification matrix
for both directions, deadline reminders, aliases, and preview.

Issues: ISSUE-26, ISSUE-27, ISSUE-28, ISSUE-29, ISSUE-30, ISSUE-31, ISSUE-32.

Exit criteria: for each row of the README's "When email is sent" tables there is a test that
asserts recipients and headers; preview output matches what is actually sent, byte for byte.

### P5 — Presentation and remaining behaviour (milestone M5)

Listing and detail views, comments, attachments, body rendering, form behaviour.

Issues: ISSUE-33, ISSUE-34, ISSUE-35, ISSUE-36, ISSUE-37, ISSUE-38.

Exit criteria: the user testing in §6 has been run and its outcome is implemented.

### P6 — Migration and cutover (milestone M6)

Issues: ISSUE-39.

See §3 and §4.

**Sequencing note:** P0 and P1 are strictly ordered. P2–P5 overlap substantially once P1
lands, and the email work (P4) should start early enough that the notification matrix is
exercised during P2 and P3 rather than bolted on at the end.

---

## 3. Data migration

Migration is written and rehearsed continuously from P0 onward, not authored at the end. Every
phase that adds a model adds its migration step in the same change.

**Order of operations**

1. **Peer organizations and groups** — from `Group` records of type `sdo`, preserving
   acronyms and names so existing statement references resolve.
2. **Relationships and Liaison Manager appointments** — from the `liaiman` `Role` rows on those
   groups. Authorized Individual (`auth`) assignments are **not** carried into the new model as
   an entry-capable role (ISSUE-3); they are recorded in the migration report so nothing is
   lost silently. **`liaison_coordinator` roles in the `iab` group are not migrated and not
   modified** — the coordinator role stays where it is (ISSUE-1), so the migration reads
   nothing from it and writes nothing to it.
3. **Statements** — including direction, contact fields, purpose, deadline, and body.
   - `technical_contacts` and `action_holder_contacts` values are preserved as legacy display
     data attached to the migrated statement; they are not fields on new statements
     (ISSUE-24).
   - Incoming statements take their **posted date** as their received date (ISSUE-14). This is
     deliberate: the posted date is what the public record already shows, so using it changes
     nothing that anyone has previously seen or cited. Where it is likely wrong, §3.1 applies.
   - `not-approved` statements become `dead`, with an event recording the original state
     (ISSUE-11).
   - **Statements addressed to more than one peer organization migrate as they are.** The
     statement model can represent them; what the new application refuses is creating one
     (ISSUE-25). They are not split, collapsed, or dropped, and they display all their target
     organizations. The migration inventories them so the count is known rather than assumed.
4. **Attachments** — into the blobstore, with legacy URL redirects in place (ISSUE-37).
5. **Events** — every existing event, with its original type, actor, timestamp, and text, so
   that a migrated statement's history reads exactly as it did before the rewrite. Events whose
   type has no counterpart in the new workflow are carried as historical events rather than
   discarded, and nothing is summarised or re-attributed.

**Validation:** a migration report is produced on every rehearsal run — counts by model,
counts by state, every approximated or dropped value, the §3.1 flagged list, the
multi-organization inventory, and a diff of statement-page and history renderings between old
and new for a sample of statements spanning the full history. Migration is reversible until
cutover: rehearsals run against a copy, never in place.

### 3.1 Received-date reconciliation

Taking the posted date as the received date is right for the great majority of statements and
wrong for any that were back-entered — recorded well after they arrived. Those are exactly the
cases the new received-date field exists for, so they get a reconciliation step of their own
rather than a silent approximation.

- **Detection:** the migration flags every incoming statement whose recorded history begins
  significantly before its posted date — the gap between its earliest event and its posting.
  The threshold is a parameter of the migration, not a constant baked into it, so it can be
  tuned once the real distribution of gaps is visible in the first rehearsal.
- **Output:** the flagged statements appear as a worklist in the migration report, each with
  its posted date, the date its history begins, the gap, and any date the body text mentions.
- **Reconciliation:** IAB Liaison Coordinators work the list, either confirming the posted date
  or setting the true received date. This happens **after cutover**, against live data, using
  the ordinary incoming-correction path (ISSUE-17) — the list is finite and the correction path
  already records who changed what and why. Nothing blocks cutover on it.
- **Presentation while unreconciled:** a flagged statement shows its migrated received date
  like any other. The default is the same date the public record has always shown, so there is
  nothing misleading to disclaim; the worklist, not a UI caveat, is what closes the gap.

---

## 4. Cutover

1. Rehearse the full migration against a production data copy; review the migration report.
2. Run the new application against migrated data on a staging deployment; give Liaison
   Coordinators, Liaison Managers, and the Secretariat access for hands-on validation.
3. Freeze liaison entry briefly, run the migration, switch over.
4. Keep the old implementation's data readable (not writable) for a defined period so that
   any migration defect can be diagnosed against the original records.
5. Hand the received-date worklist (§3.1) to the IAB Liaison Coordinators and track it to
   completion. This runs after cutover and does not gate it.

There is no phased dual-write period: the two schemas differ too much for dual-write to be
cheaper than a short freeze.

---

## 5. Testing

- **Unit** — authorization helpers, approval-invalidation comparison, state transitions,
  recipient computation.
- **Workflow** — one test per path through each README flowchart, including negative paths
  (reviewer cannot mark dead; approver cannot review; Secretariat cannot both enter and post
  an incoming statement).
- **Email** — one test per row of the notification tables, asserting both recipients and
  headers, and a test that preview output equals sent output.
- **Migration** — assertions on the migration report; rendering diffs on a historical sample.
- **User testing** — see §6.

---

## 6. User testing: arranging the separate incoming and outgoing listings

Statements are listed today with incoming and outgoing **combined** in one table. The
reimplementation does not mix them in any listing table: their dates, useful columns, and
workflows differ (README §Display and Ordering, ISSUE-14). Only the thread view shows both
directions together. What remains open is how the two tables are arranged, and that is
settled by testing with users rather than by assertion.

**What is tested:** the two tables on one page, as sections or tabs, versus separate pages
for incoming and outgoing. The test uses representative tasks: find a statement received from
a given organization; find what the IETF last sent to a given group; see what is awaiting your
review; see what is overdue for a response; follow a conversation with a peer group across
both directions (thread view).

**Who:** Liaison Coordinators, Liaison Managers, WG chairs who have sent statements, and
Secretariat staff. Both frequent and occasional users — a view that suits people who live in
the tool may not suit a chair who uses it twice a year.

**What the outcome decides:** the default arrangement of the two tables, and how a user moves
between them. Separation itself, and direction-specific ordering (incoming by received date,
outgoing by posted date), are not under test.

**When:** during P5, early enough that the outcome shapes the listing implementation rather
than being retrofitted.

---

## 7. Decisions to revisit once there is experience

These are deliberate starting positions, chosen to be safe rather than convenient. Each should
be re-examined after the new system has been in real use.

| Decision | Recorded in | Why it may be relaxed |
|----------|-------------|----------------------|
| A multi-group statement is always reviewed by an IAB Liaison Coordinator, even when every target group has the same Liaison Manager | ISSUE-7, ISSUE-25 | If Coordinator review adds no value in the same-manager case, the manager could review directly |
| Any content edit after approval forces re-approval | ISSUE-13 | Experience may show that some content edits are routine enough not to warrant another approval round |
| No structured "replaces" relationship between an outgoing correction and the statement it corrects | ISSUE-17 | If prose references prove hard to follow in practice |
| No structured follow-up/assignee field | ISSUE-24 | If follow-up responsibility turns out to need tracking rather than convention |

---

## 8. Risks

| Risk | Mitigation |
|------|-----------|
| Migration loses or distorts history | History migrates event for event with original type, actor, timestamp, and text (§3); rehearse repeatedly; migration report on every run; history-rendering diffs across a historical sample; original data kept readable after cutover |
| Back-entered incoming statements carry a migrated received date that is wrong | Posted date is the least-disturbing default; statements whose history begins well before posting are flagged and reconciled as a tracked worklist (§3.1) rather than left as a silent approximation |
| A permissive multi-organization model lets new multi-organization statements slip through | The constraint lives in the create/edit paths and the API, and is covered by tests on both; the model's permissiveness exists only so migrated statements display correctly (ISSUE-25) |
| Authorization regressions when moving Liaison Manager off `Role` | Permission tests written in P0 against real appointment data, before any view depends on them |
| Authorization helpers now read from two sources, so a coordinator check could be wired to the wrong one | Tests cover both paths explicitly: manager authority from appointments, coordinator authority from the `iab` group role; a migration assertion confirms every pre-migration Liaison Coordinator is still one afterwards |
| API consumers break on the changed field set | Inventory consumers and announce field changes before cutover |
| The re-approval rule frustrates users enough to be worked around | Watch for it during staging validation; §7 records it as revisitable |
