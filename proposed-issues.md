# Proposed Issues for the Reimplemented Liaison Tool

This document proposes issues for a new repository that consolidates and restructures
work tracked in `ietf-tools/liaison-tooling-2025a` (GitHub issues #1–#45) and open issues
carrying the `component: liaison/` label in `ietf/datatracker`. It also introduces new
issues from analysis of the current codebase and RFC 4052bis / RFC 4053bis compliance gaps.

Each issue is attributed to its upstream origins, and states the decision it records.

## How to read these issues

**These issues describe a reimplementation, not a series of edits to the current code.** The
liaison application is being rebuilt: new models, forms, views, templates, and email layer.
No issue instructs anyone to change a particular line of the existing implementation.

Each issue's **Background** explains why the requirement exists, which often means describing
how the current tool behaves. Each issue's **Requirements** state what the new application
must do. The current implementation appears in three roles only:

1. **Origin of a requirement** — recorded in the `Origins` line.
2. **Data to be migrated** — covered by ISSUE-39.
3. **A compatibility surface** — public URLs, attachment URLs, and `/api/v1`.

Two structural decisions run through the whole set:

- External organizations are **not** modelled as datatracker `Group` records. The liaison
  application owns purpose-built models for peer organizations, their groups, formal liaison
  relationships, and appointments (ISSUE-1, ISSUE-2).
- Liaison Manager authority is answered from those appointment records, not from `Role` rows on
  external groups. The IAB Liaison Coordinator role stays a datatracker `Role` in the `iab`
  group, because it is an IAB responsibility rather than a per-organization one.

Phasing, migration, cutover, and the user testing that settles how the separate incoming and
outgoing listings are arranged are in the [Implementation Plan](proposed-implementation-plan.md).
The workflow itself is described in the [README](README.md).

> **Note:** All references to "RFC 4052bis" and "RFC 4053bis" will be updated to the assigned
> RFC numbers once the documents are published.

---

## Proposed Labels

| Label | Description |
|-------|-------------|
| `workflow` | State machine and approval workflow |
| `roles` | Authorization and permissions |
| `data-model` | Models and schema |
| `email` | Email sending and notifications |
| `usability` | UI and UX |
| `attachments` | Attachment upload, removal, storage |
| `incoming` | Specific to incoming liaison statements |
| `outgoing` | Specific to outgoing liaison statements |
| `rfc-compliance` | Required or implied by RFC 4052bis / RFC 4053bis |
| `foundation` | Must land before dependent work can start |
| `migration` | Migration of existing data, or decommissioning of current structures |
| `proposed-resolution` | Has a proposed resolution needing sign-off before coding |
| `blocked` | Cannot start until another issue is resolved |

---

## Proposed Milestones

- **M0 – Foundation**: peer organization domain models, management UI, authorization
- **M1 – Statement lifecycle**: statement model, state machine, approval/review gates, posting
- **M2 – Roles and permissions**: authorization model, acting roles, dashboards
- **M3 – Contact fields and targeting**: contact model, addressing, purpose values
- **M4 – Email and notifications**: all email behaviour
- **M5 – Presentation**: listings, detail views, comments, attachments
- **M6 – Migration and cutover**: migrating existing data; retiring the current implementation

---

## Issues

---

### ISSUE-1: Peer organization, peer group, relationship, and Liaison Manager appointment models

**Labels:** `data-model`, `foundation`, `roles`
**Priority:** High
**Milestone:** M0
**Origins:** new requirement; RFC 4052bis §2, §3

#### Background

The current tool models external SDOs as datatracker `Group` records of type `sdo`, and liaison
manager appointments as `Role` rows on those groups. This borrows a structure built for IETF
working groups to describe organizations that are not IETF groups at all, and it leaves the
liaison domain — relationships, appointments, aliases, contact lists — spread across generic
tables.

The reimplementation gives the liaison application purpose-built models for this domain, and
the redesign includes managing the objects in them (ISSUE-2). Terminology follows RFC 4052bis,
which speaks of *peer organizations* and *formal liaison relationships*.

#### Requirements

**Models** — owned by the liaison application:

| Model | Holds |
|-------|-------|
| **Peer organization** | An external SDO or other body: name, acronym, URL, status. The unit a statement is addressed to or received from (ISSUE-25) |
| **Peer group** | A body within a peer organization — an ITU-T Study Group, a 3GPP working group. Carries the group's To-Contact list (ISSUE-22) and its liaison alias (ISSUE-31). Each organization can also have an **org-level** peer group representing the organization as a whole (for example "ITU-T"), for statements not addressed to or from a specific group. Its display name includes the organization |
| **Liaison relationship** | The IETF's formal relationship with a peer organization per RFC 4052bis §2: how it was established, its scope, its status, and relevant references |
| **Appointment** | A person appointed as Liaison Manager for a relationship, with start and end dates and the appointing body. An appointment **covers one or more of the relationship organization's peer groups**, by default all of them. The Liaison Manager of a peer group is whoever holds a current appointment covering it |

**The IAB Liaison Coordinator is not one of these models.** It stays a datatracker `Role` in
the `iab` group, as it is today (`liaison_coordinator`, resolved through the usual role
machinery). The coordinator role is a responsibility of an IAB member, held for the IAB as a
whole; it is not tied to any individual peer organization and there is nothing
relationship-specific to record about it. Only the Liaison Manager role — which *is* per
relationship, and today lives as a `liaiman` `Role` on an SDO `Group` — moves into the
liaison application.

- **Informal relationships are supported by omission.** RFC 4052bis is explicit that statements
  can be exchanged without a formal relationship, and that where none exists the IAB is
  responsible. A peer organization therefore does not require a relationship record, and a
  peer group does not require a Liaison Manager. Every workflow rule that says "or an IAB
  Liaison Coordinator where there is no Liaison Manager" depends on this being representable.
- **Appointments are time-bounded.** A lapsed appointment does not confer authority, and
  history remains attributable to whoever held the appointment at the time.
- **Appointments are per relationship and cover groups.** The relationship is with the peer
  organization, as RFC 4052bis describes it. Each appointment names the peer groups it covers:
  all of the organization's groups by default, or a subset, such as a single ITU-T study
  group. A group may be covered by more than one current appointment, and one person may
  cover several groups. Every rule in these issues that refers to "the Liaison Manager of a
  group" means the holders of current appointments covering that group. That includes
  reviewer selection (ISSUE-7), notifications (ISSUE-28, ISSUE-32), "action taken"
  (ISSUE-16), and aliases (ISSUE-31).

**Authorization:**

- "Is this person the Liaison Manager for this group?" is answered by the liaison
  application from appointment records, not from `Role` rows on external groups.
- "Is this person an IAB Liaison Coordinator?" is answered exactly as it is today, from the
  `liaison_coordinator` `Role` in the `iab` group. This does not move.
- Authorization for IETF-side bodies is otherwise unchanged: WG chairs, ADs, the IETF Chair,
  the IAB Chair, and the Secretariat are datatracker roles as they are today.
- The dividing line is scope, not convenience: a role held **per peer organization** belongs to
  the liaison application, and a role held **for the IETF or the IAB as a whole** stays a
  datatracker group role.
- There is no appointment type, and no role of any kind, that lets a person from a peer
  organization enter statements (ISSUE-3).

**Relationship to IETF-side groups:**

- IETF-side bodies remain datatracker `Group` records. Only the external side moves.

#### Acceptance Criteria

- [ ] The four models exist in the liaison application with the attributes above.
- [ ] A peer organization can exist with no relationship record, and a peer group with no
      Liaison Manager; the workflow handles both.
- [ ] Appointments carry start and end dates, and a lapsed appointment confers no authority.
- [ ] Each appointment records the peer groups it covers (all of the organization's groups by
      default), and "Liaison Manager of a group" resolves from the appointments covering that
      group.
- [ ] Liaison Manager authority resolves through appointments, with no dependence on `Role`
      rows on external groups.
- [ ] IAB Liaison Coordinator authority still resolves through the `liaison_coordinator` role
      in the `iab` group; the liaison application defines no coordinator appointment type.
- [ ] Statement targeting references peer organizations and peer groups (ISSUE-25).
- [ ] Existing SDO `Group` and `Role` data migrates into these models (ISSUE-39).

---

### ISSUE-2: Management UI for peer organizations, relationships, and appointments

**Labels:** `usability`, `foundation`, `roles`
**Priority:** High
**Milestone:** M0
**Origins:** new requirement

#### Background

The models in ISSUE-1 are only useful if the people responsible for them can maintain them
without a database shell. Today, adding an SDO or changing a liaison manager means Secretariat
intervention through group administration built for something else. The redesign includes
management of these objects.

#### Requirements

- Full create, edit, and retire capability, through the application UI, for:
  - peer organizations,
  - peer groups, including their To-Contact lists (ISSUE-22) and liaison aliases (ISSUE-31),
  - liaison relationships,
  - Liaison Manager appointments, including end-dating.
- **Who can maintain what:**

  | Object | Who |
  |--------|-----|
  | Peer organizations, peer groups | IAB Liaison Coordinators, Secretariat |
  | To-Contact lists, liaison aliases | IAB Liaison Coordinators, Secretariat, and the relationship's Liaison Manager for their own groups |
  | Relationships | IAB Liaison Coordinators, Secretariat |
  | Liaison Manager appointments | IAB Liaison Coordinators, Secretariat (the IAB appoints; the tool records) |

- **IAB Liaison Coordinators are not managed here.** The coordinator role stays a `Role` in the
  `iab` group (ISSUE-1), maintained wherever IAB group roles are maintained today. This UI
  neither creates nor ends coordinator appointments, and adding a coordinator must not require
  touching the liaison application.

- **Ending a formal relationship is not retiring an organization.** An informal relationship
  needs no record. It exists simply because statements have been exchanged, and it cannot be
  ended (ISSUE-1, RFC 4052bis).
  - **Ending a formal relationship** end-dates the relationship record and its Liaison
    Manager appointments. The organization and its groups stay available on new statements.
    Statements continue informally, with an IAB Liaison Coordinator acting where there is no
    Manager. The group aliases fall back to `liaison-coordination@iab.org` (ISSUE-31).
  - **Retiring a peer organization or peer group** happens only when it ceases to exist or is
    restructured, for example when an ITU-T study period ends and study groups are
    reorganized. A retired organization or group no longer appears as a choice on new
    statements.
- Nothing referenced by a statement is ever deleted. Retired organizations and groups, and
  ended relationships and appointments, still resolve and display on historical statements.
- Changes are recorded with who made them and when.
- A public or semi-public view lists the current peer organizations, their relationship status,
  and the responsible Liaison Manager — the "who is our liaison manager for X?" question the
  community currently answers by searching.

#### Acceptance Criteria

- [ ] Every object in ISSUE-1 can be created, edited, and retired through the UI.
- [ ] The permission table above is enforced.
- [ ] Coordinator membership is not managed here, and appointing one requires no change in the
      liaison application.
- [ ] Ending a formal relationship end-dates it and its appointments, and leaves the
      organization and its groups available on new statements.
- [ ] Retiring an organization or group preserves historical statement references and removes
      it from new statement choices.
- [ ] No object referenced by a statement can be deleted.
- [ ] Changes are attributed and timestamped.
- [ ] A listing shows current organizations, relationship status, and responsible managers.

---

### ISSUE-3: No external-party entry role

**Labels:** `roles`, `incoming`, `migration`
**Priority:** High
**Milestone:** M0
**Origins:** analysis of the current implementation ("Authorized Individual" in the incoming
liaison roles)

#### Background

The current tool includes "Authorized Individual" among the roles that may enter incoming
liaison statements — a role held by people from an external SDO, granted datatracker access to
enter statements on that SDO's behalf. The new model has no equivalent: incoming statements are
entered only by IETF-side roles, and external organizations are no longer datatracker groups
that can carry roles at all (ISSUE-1).

#### Requirements

- The new implementation has no role by which a person from a peer organization enters
  statements. Entry roles are: Liaison Manager, IAB Liaison Coordinator, Secretariat.
- Existing Authorized Individual assignments are **not** migrated into an entry-capable
  appointment. They are listed in the migration report so that nothing disappears silently, and
  the affected people are identified for the Liaison Coordinators to contact if needed
  (ISSUE-39).
- Any `Role` rows for the role in the datatracker are left alone if the role is used for other
  purposes; what is removed is liaison entry access.

#### Acceptance Criteria

- [ ] No entry path exists for a person holding only an external-party role.
- [ ] Migration reports every Authorized Individual assignment it did not carry forward.
- [ ] The three entry roles are the complete set.

---

### ISSUE-4: No approval bypass — every outgoing statement is approved in the tool

**Labels:** `workflow`, `outgoing`
**Priority:** High
**Milestone:** M1
**Origins:** liaison-tooling-2025a #1; RFC 4053bis §4.1

#### Background

The current tool offers an "Obtained prior approval" checkbox on the outgoing form. When it is
checked the statement is posted immediately, bypassing the approval workflow entirely. RFC
4053bis §4.1 requires AD approval for all outgoing statements with no exceptions, and the
proposed workflow has no bypass path.

The reimplementation simply does not provide such a bypass. This issue exists to make that a
tested requirement rather than an omission, and to cover the case it was used for: approval
genuinely obtained out of band.

#### Requirements

- The new implementation offers no mechanism by which an outgoing statement reaches `posted`
  without an `approved` event recorded against it.
- Every outgoing statement enters `draft`, and reaches `pending` only when marked ready —
  regardless of who submits it, including the Secretariat and IAB Liaison
  Coordinators.
- Approval obtained out of band is recorded **in the tool** as a proxy approval by a
  Coordinator or the Secretariat, with a mandatory comment naming the actual approver
  (ISSUE-9, ISSUE-18). That is the only supported route for prior approval.

#### Acceptance Criteria

- [ ] No form field, view, API path, or management command posts an outgoing statement that
      has no valid `approved` event.
- [ ] Submitting an outgoing statement in any role results in `draft`, and marking it ready
      results in `pending`.
- [ ] Out-of-band approval is recordable only as a proxy approval with a naming comment.
- [ ] A regression test asserts that no code path skips the approval gate.

---

### ISSUE-5: Statement lifecycle — states, transitions, and event-based gate tracking

**Labels:** `workflow`, `data-model`
**Priority:** High
**Milestone:** M1
**Origins:** liaison-tooling-2025a README; liaison-tooling-2025a #34

#### Background

The current state machine is `pending → approved → posted`, with a `dead` state reversible via
"resurrect". The new workflow needs a draft stage, a two-gate outgoing path (approval then
review), a send-back path, and a way to express that an approval has been invalidated by a
content edit — without multiplying states into a combinatorial matrix.

#### Requirements

**States** — exactly four:

| State | Meaning |
|-------|---------|
| `draft` | Being edited; not in the review queue. Also where a statement lands when sent back for rework (ISSUE-12) |
| `pending` | Outgoing only: marked ready, awaiting approval and then review |
| `posted` | Published |
| `dead` | Marked not to be posted, with a mandatory reason (ISSUE-11) |

**Transitions:**

- `draft → pending` — marked ready (outgoing).
- `draft → posted` — marked ready (incoming). Marking an incoming statement ready records its
  single `reviewed` event and posts it immediately (ISSUE-8, ISSUE-6).
- `pending → draft` — sent back for rework by the reviewer, with a mandatory comment
  (ISSUE-12).
- `pending → posted` — automatic (outgoing), triggered by completing the review after a valid
  approval (ISSUE-6).
- `draft → dead`, `pending → dead`, and `posted → dead` — with a mandatory reason (ISSUE-11).
  Available to Coordinators and the Secretariat at any point, and to approvers (as rejection)
  from `pending`. **Not** available to a
  reviewer acting as reviewer.
- `dead → draft` — resurrection by a Coordinator or the Secretariat.

**Gate tracking** — gate completion is recorded as events on the statement, not as states:

- `approved` — AD/Chair approval (ISSUE-9). Outgoing only.
- `reviewed` — the single review (ISSUE-7 outgoing). For incoming statements, recorded by the
  mark-ready action itself (ISSUE-8).
- `sent_back` — returned to draft for rework, carrying the mandatory comment (ISSUE-12).
- Approval validity is derived, not stored as a state: an `approved` event stops counting when
  the statement's content changes after it (ISSUE-13).

**Posting trigger** (ISSUE-6):

- Outgoing: the review action, which is available only while a currently valid `approved`
  event exists. A `reviewed` event from before the current approval never triggers posting.
- Incoming: the mark-ready action.

#### Acceptance Criteria

- [ ] The four states and only these transitions are implementable and enforced.
- [ ] Attempting an undefined transition raises rather than silently succeeding.
- [ ] Gate completion is queryable from the event log alone.
- [ ] A reviewer cannot reach `dead` from the review action.
- [ ] Tests cover every transition, including resurrection into `draft`.

---

### ISSUE-6: Posting is triggered automatically by the last human action; attribute posting to the (System) user

**Labels:** `workflow`
**Priority:** High
**Milestone:** M1
**Origins:** liaison-tooling-2025a README

#### Background

The current tool requires an explicit human posting action and attributes the posting to the
approving person. In the new workflow no one "posts" a statement. Posting follows
automatically from the last human action the workflow requires, and is attributed to the
`(System)` user.

For outgoing statements, approval comes first and review second (ISSUE-7, ISSUE-9), so **the
review is always what triggers posting**. For incoming statements there is no approval or
second review (ISSUE-8), so marking the statement ready triggers posting.

The "(System)" person is the datatracker's existing system `Person` record used for automated
events.

#### Requirements

- **Outgoing:** completing the review (ISSUE-7) posts the statement immediately. The review
  action is available only while the statement has a currently valid approval (ISSUE-9,
  ISSUE-13), so approval is always in place before the review that triggers posting.
  - Only a review made **after** the current valid approval counts. A review recorded
    earlier, for example before the statement was sent back, marked dead, or re-approved,
    never triggers posting. If the statement returns to `pending`, it needs a fresh review.
  - Recording or restoring an approval never posts the statement by itself.
- **Incoming:** marking the statement ready (ISSUE-8) posts it immediately.
- The posting event is attributed to the system `Person`, not to the reviewer or the person who
  marked it ready.
- The posted date recorded at this moment is what outgoing statements are displayed and
  ordered by (ISSUE-14).
- The external email is dispatched at this moment, unless the exceptional "Do not send email
  when posted" option was set (ISSUE-30). Internal notifications are sent regardless
  (ISSUE-28, ISSUE-32).

#### Acceptance Criteria

- [ ] Completing the review of an outgoing statement with a valid approval posts it
      immediately.
- [ ] Recording an approval does not post a statement, even if a review exists from an earlier
      round.
- [ ] A statement resurrected or sent back and readied again needs a fresh review before it
      can post.
- [ ] Marking an incoming statement ready posts it immediately.
- [ ] The posting event's actor is the system `Person`.
- [ ] No separate "post" action exists or is required.
- [ ] External email is dispatched on posting unless the do-not-send option is set. Internal
      notifications are sent in both cases.

---

### ISSUE-7: Single review of outgoing statements, after approval — LM for one group, LC for several

**Labels:** `workflow`, `roles`, `outgoing`
**Priority:** High
**Milestone:** M1
**Origins:** liaison-tooling-2025a README; liaison-tooling-2025a #41

#### Background

An outgoing statement is addressed to one peer organization and to one or more groups within
it (ISSUE-25). It requires, in order:

1. Any required AD/Chair **approval** (ISSUE-9) — first, per RFC 4053bis §4.1.
2. A single **review** — after approval.

Who performs the review depends on how many groups are targeted. Reviewing a statement that
goes to several groups is a coordination job, and the IAB Liaison Coordinator is the role that
does it.

#### Requirements

- **Ordering:** the review action is unavailable until a valid `approved` event exists. Review
  never runs in parallel with approval.
- **Who reviews:**
  - **One target group, and that group has a Liaison Manager** → that Liaison Manager reviews.
    An IAB Liaison Coordinator may review on the Manager's behalf.
  - **More than one target group** → an IAB Liaison Coordinator reviews, **even when every
    target group has the same Liaison Manager**.
  - **The target group has no Liaison Manager** → an IAB Liaison Coordinator reviews.
- **One review only:** exactly one `reviewed` event is recorded, whatever the number of target
  groups. No workflow path requires more than one review of the same statement.
- **Coordination is out of scope for the tool:** the Coordinator is expected to coordinate with
  the Liaison Managers of the target groups before completing a multi-group review. **The tool
  is not involved in that coordination** and records only the Coordinator's single review. It
  does not solicit, track, or wait for Manager sign-off.
- **Managers are kept informed:** the Liaison Managers of every target group are notified as
  the statement progresses — marked ready, approved, reviewed, posted — whether or not
  they are the reviewer (ISSUE-28).
- **Reviewer's outcomes:** complete the review (which posts the statement, ISSUE-6), or send it
  back for rework with a mandatory comment (ISSUE-12). A reviewer cannot reject a statement and
  cannot mark it dead.
- **Editable addressing fields at review:** the reviewer may edit the statement's addressing
  fields, constrained only to syntactically valid addresses; the review screen highlights how
  each differs from its initial population. Editing addressing fields does not invalidate
  approval; editing content does (ISSUE-13).
- **Review queue:** Liaison Managers and Coordinators see a list of statements awaiting their
  review (ISSUE-19).

#### Acceptance Criteria

- [ ] A single `reviewed` event gates posting, for any number of target groups.
- [ ] The review action is unavailable until a valid approval is recorded.
- [ ] One target group with a Liaison Manager → only that Manager (or a Coordinator standing
      in) can review.
- [ ] More than one target group → only an IAB Liaison Coordinator can review, including when
      all target groups share one Liaison Manager.
- [ ] No target-group Liaison Manager → an IAB Liaison Coordinator can review.
- [ ] The Liaison Managers of all target groups are notified at each transition.
- [ ] The reviewer can send back for rework but cannot reject or mark dead.
- [ ] The reviewer can edit addressing fields with syntactic validation only, and changes are
      highlighted.
- [ ] Auto-post occurs once approval and the single review are both valid (ISSUE-6).
- [ ] A pending-review list is available to Liaison Managers and Coordinators.

---

### ISSUE-8: Incoming statements post when the Liaison Manager (or Coordinator) marks them ready; no second review

**Labels:** `workflow`, `roles`, `incoming`
**Priority:** High
**Milestone:** M1
**Origins:** liaison-tooling-2025a README
**Decision:** An incoming statement is entered by the Liaison Manager for the sending group if
one exists, otherwise by an IAB Liaison Coordinator. No second-person review is required before
it is marked ready and posted.

#### Background

The current tool auto-posts every incoming statement on submission with no human gate. The new
model requires exactly one responsible human — the Liaison Manager for the sending group, or an
IAB Liaison Coordinator where there is no Manager — but no more than one. There is explicitly
**no** gate in which a coordinator reviews a statement entered by a manager, or vice versa.

#### Requirements

- Incoming statements do not post on submission. They are created in `draft`, and posting
  follows from an explicit "mark ready" action.
- **Entry:** the Liaison Manager for the sending group enters the statement if one exists;
  otherwise an IAB Liaison Coordinator enters it. Manager authority comes from appointment
  records and coordinator authority from the `iab` group role (ISSUE-1).
- **A Coordinator can always act, even when a Liaison Manager exists.** If the Manager is
  unavailable, an IAB Liaison Coordinator can enter the statement and mark it ready on the
  Manager's behalf (ISSUE-18). The history records that the Coordinator acted for that
  Manager.
- **No second review:** the same Manager or Coordinator who enters the statement marks it ready.
  Because incoming statements need no approval, marking ready posts immediately. No second
  person's review is required or solicited.
- **Received date:** the person entering the statement records the date the IETF received it,
  which may be well before the date of entry (ISSUE-14).
- **Secretariat on behalf (exception):** the Secretariat may enter an incoming statement on
  behalf of a Manager or Coordinator, but only into `draft`. A Manager or Coordinator must then
  mark it ready. The Secretariat cannot both enter and post the same statement (ISSUE-18).
- **Coordinator acting as manager:** if an IAB Liaison Coordinator is also the Liaison Manager
  for the sending group, the tool records the action in the **Liaison Manager** role (ISSUE-15).
- Authorized entry roles: Liaison Manager, IAB Liaison Coordinator, Secretariat. There is no
  external-party entry role (ISSUE-3).
- The post event is attributed to `(System)` (ISSUE-6).

#### Acceptance Criteria

- [ ] Submitting an incoming statement does not post it.
- [ ] A Liaison Manager for the sending group — or, absent one, an IAB Liaison Coordinator —
      can enter a statement and post it with no second-person review.
- [ ] Posting is attributed to `(System)`.
- [ ] A Secretariat-entered statement lands in `draft` and cannot be posted by the Secretariat.
- [ ] A Coordinator who is also the group's Liaison Manager has the action recorded in the
      Liaison Manager role.
- [ ] No approval is required or solicited for incoming statements.
- [ ] No external-party role can reach the incoming entry form.

---

### ISSUE-9: Approval authority for outgoing statements

**Labels:** `roles`, `outgoing`
**Priority:** High
**Milestone:** M1
**Origins:** liaison-tooling-2025a #2, #41; RFC 4053bis §4.1; ietf/datatracker #9457

#### Background

RFC 4053bis §4.1 defines approval authority for outgoing liaison statements:

> "All liaison statements sent by any group in the IETF need AD approval… This does not
> include statements sent by the IAB, which require IAB approval. Statements sent from an
> area… need approval by at least one of the responsible ADs. Statements sent by the IETF
> or IESG require IETF Chair approval."

**Approver by originating group:**

| From group | Required approver |
|------------|------------------|
| WG or Area | At least one responsible AD |
| IETF / IESG | IETF Chair |
| IAB | IAB Chair (acting for the IAB as a body) |

IAB Liaison Coordinators and Liaison Managers are **not** approvers. Approval happens first;
the single review (ISSUE-7) follows. Liaison managers are consulted but do not approve.

ietf/datatracker#9457 documents a bug in the current implementation: approver computation
intersects `Role` objects rather than `Person` objects, so the intersection is always empty and
approval notification emails are never sent. The new implementation must compute approvers by
person.

#### Requirements

- Approver computation is by **person**, from the statement's originating groups per the table
  above. Tests cover the multi-group and multi-AD cases that broke the current implementation.
- Two approval paths:
  - **Direct approval** by the actual AD / IETF Chair / IAB Chair. No comment required, though
    one may be added.
  - **Proxy approval** by an IAB Liaison Coordinator or the Secretariat, recording approval
    obtained out of band. A comment is **required** and must name the person who actually
    approved. It is stored on the approval event. (The general act-on-behalf mechanism is
    ISSUE-18.)
- **Rejection:** an approver who will not approve marks the statement `dead` with a mandatory
  reason (ISSUE-11). Rejection is an approver's outcome; it is not available to reviewers
  (ISSUE-12).
- **Ordering:** approval precedes review (ISSUE-7); the review action is not offered until a
  valid approval exists.
- **Editable addressing fields at approval:** the approver may edit addressing fields,
  constrained only to syntactically valid addresses, with differences from the initial
  population highlighted. An approver may also edit content **before** approving, and the
  approval then covers the edited content. Content edits made after the approver has
  approved invalidate that approval (ISSUE-13).
- **The submitter is told what the approver changed:** every approver edit, content or
  addressing, is reported to the submitter with the next notification they receive (ISSUE-28).
- **Visibility:** each approver can see the statements awaiting their approval, and an AD's are
  surfaced on the AD dashboard (ISSUE-19).
- Approval-request notifications go to the computed approvers (ISSUE-28).

#### Acceptance Criteria

- [ ] Approvers are computed by person and match the table for every originating group type.
- [ ] Actual approvers can approve directly; Coordinators and Secretariat can approve as proxy.
- [ ] Proxy approval without a comment is rejected.
- [ ] The approval event records who performed the action and, for proxy approvals, who
      actually approved.
- [ ] No other role can approve.
- [ ] An approver can mark a statement dead with a reason.
- [ ] An approver's edits made before approving are reported to the submitter in the Approved
      notification (ISSUE-28).
- [ ] Review is unavailable until a valid approval exists.
- [ ] Approvers see only the statements they may act on.
- [ ] Tests cover: direct approval by each approver role; multi-AD and multi-group approver
      computation; proxy approval with and without a comment; unauthorized attempts.

---

### ISSUE-10: Draft statements

**Labels:** `workflow`, `usability`
**Priority:** High
**Milestone:** M1
**Origins:** liaison-tooling-2025a #14; liaison-tooling-2025a README

#### Background

Statements are "entered, and then edited (perhaps by multiple people)" before being readied for
review. The current tool has no draft stage; statements become pending immediately.

Drafts are also where a statement lands when a reviewer sends it back for rework (ISSUE-12).

#### Requirements

**Editing:**

- Statements are created in `draft`.
- Multiple authorized users can edit a draft, per the role rules in the README.
- Drafts are not publicly visible and are excluded from `/api/v1`. They are visible to:
  - the statement's authorized editors;
  - its approvers, actual or computed (ISSUE-9);
  - the **Liaison Managers of every relevant peer group**: every target group of an outgoing
    statement, or the sending group of an incoming one, whether or not that Manager will be
    the reviewer;
  - IAB Liaison Coordinators and the Secretariat.

  Liaison Managers see drafts early so a chair can coordinate with them before marking a
  statement ready. Their in-flight views (ISSUE-19) depend on it.
- History records all edits.
- The action that ends drafting is labelled **"Mark ready"** in both directions. For an
  outgoing statement it moves the statement to `pending`. For an incoming statement it posts
  the statement immediately (ISSUE-8). Either way it triggers the ISSUE-28 / ISSUE-32
  notifications.
- A line of text next to the button says what will happen, by direction:
  - outgoing: "Sends to the approvers:" followed by the computed approvers by name (ISSUE-9);
  - incoming: "Posts the statement immediately."
- Approvers may **view** a draft but cannot record approval until it is marked ready.

**Hard deletion — the one permitted case:**

- A statement that **no notification has ever been sent about** may be hard-deleted by an
  authorized editor, removing the statement and its associated records. This is the only hard
  deletion the tool permits. Every other statement that should not proceed is marked `dead`
  (ISSUE-11).
- In practice:
  - an **outgoing** statement can be hard-deleted until it is first marked ready;
  - an **incoming** statement is notified on entry (ISSUE-32), so it is never eligible;
  - a statement that was sent back for rework (ISSUE-12) has been notified, so it is not
    eligible, even though it is back in `draft`.
- **Why the line is drawn here:**
  - Before the first notification, a draft exists only for its editors, and its history holds
    only their own work.
  - The first notification is the first time the statement leaves its editors. Copies then
    exist in other people's mailboxes and in the `liaison-coordination@iab.org` list archive
    (ISSUE-27), which is the record-keeping copy. Hard deletion after that would leave
    archived mail pointing at a URL that no longer resolves.
  - From that point the history can also hold actions by other people, such as approvals,
    send-backs, and comments. Hard deletion would erase the record of what they were asked
    and what they did.
  - Marking `dead` meets the practical need to get a duplicate or mistaken entry out of the
    way. A never-posted dead statement is visible only to Liaison Coordinators and the
    Secretariat (ISSUE-11).

#### Acceptance Criteria

- [ ] A draft can be created and edited without triggering approval notifications.
- [ ] "Mark ready" is a distinct action with that label in both directions.
- [ ] The text next to the button names the computed approvers for an outgoing statement, and
      says the statement will post immediately for an incoming one.
- [ ] Approvers can view a draft but cannot approve it.
- [ ] The Liaison Manager of every relevant peer group can view a draft, including on a
      multi-group statement where a Coordinator is the reviewer.
- [ ] Drafts are invisible to unauthenticated users and to users without a role on the
      statement, and are excluded from `/api/v1`.
- [ ] An outgoing draft that has never been marked ready can be hard-deleted by an authorized
      editor.
- [ ] A statement that any notification has been sent about cannot be hard-deleted. This
      includes every incoming statement and every statement sent back for rework.

---

### ISSUE-11: The `dead` state — rejection, retraction, and abandonment

**Labels:** `workflow`, `data-model`
**Priority:** High
**Milestone:** M1
**Origins:** liaison-tooling-2025a #34; liaison-tooling-2025a README

#### Decision

Statements and their attachments are **never deleted**, with the single exception of a
statement that no notification has ever been sent about (ISSUE-10). Rejected, retracted, and
abandoned statements are marked `dead`.

**Exceptional removal is outside the tool.** If content must be removed for legal reasons or
because it is abusive, removing it is an operational action taken directly on the database
by the Secretariat and tools team. The tool offers no path for it, and that action is not
governed by this workflow.

#### Requirements

- **Who can mark a statement dead:**
  - an **approver**, as their rejection outcome (ISSUE-9);
  - a **Liaison Coordinator** or the **Secretariat**, at any point, including after posting;
  - **not** a reviewer acting as reviewer — a reviewer's only negative outcome is send back for
    rework (ISSUE-12).
- **Reason required:** marking a statement dead requires a reason, recorded as an event. An
  empty reason is rejected.
- **Access control:**
  - A dead statement that was **never posted** is visible only to Liaison Coordinators and the
    Secretariat; everyone else gets a 404.
  - A dead statement that **was once posted** displays "This statement has been removed" at its
    public URL rather than a 404, because that URL was public. The page also shows the reason
    entered when the statement was marked dead. Links to other statements in the reason are
    rendered as links, so a duplicate points readers to the statement that remains. There is
    no automatic redirect, because the removal itself should stay visible.
  - The statement's history is not made public by this. Only the notice and the reason are
    shown.
  - When a once-posted statement is being marked dead, the form warns that the reason will be
    shown publicly. For a statement never posted, the reason stays visible only to Liaison
    Coordinators and the Secretariat.
- **API:** dead statements are excluded from `/api/v1`.
- **Resurrection:** Liaison Coordinators and the Secretariat can resurrect a dead statement
  into `draft`, from where it re-enters the normal workflow. Any prior approval is
  re-evaluated under ISSUE-13 when it is readied again.
- **Attachments** on a dead statement are retained, not deleted.
- Statements migrated in the current `not-approved` state become `dead`, with an event
  recording the original state (ISSUE-39).

#### Acceptance Criteria

- [ ] Marking dead without a reason is rejected.
- [ ] Approvers, Coordinators, and Secretariat can mark dead; reviewers cannot.
- [ ] Never-posted dead statements 404 for everyone except Coordinators and Secretariat.
- [ ] Once-posted dead statements show "This statement has been removed" publicly, with the
      reason, and statement links in the reason are clickable.
- [ ] The mark-dead form warns that the reason will be public when the statement was once
      posted.
- [ ] The history of a dead statement is not publicly visible.
- [ ] `/api/v1` excludes dead statements.
- [ ] Coordinators and Secretariat can resurrect a dead statement into `draft`.
- [ ] No statement or attachment record is hard-deleted by this path.

---

### ISSUE-12: Reviewers send statements back for rework; they cannot reject

**Labels:** `workflow`, `roles`, `outgoing`
**Priority:** High
**Milestone:** M1
**Origins:** new requirement

#### Background

A reviewer who finds a problem with an outgoing statement is not making a judgment that the
statement should never be sent — they are identifying work that needs doing. The current tool's
review-stage rejection conflates the two and ends the statement's life.

A reviewer therefore has no reject action. Their negative outcome is to send the statement back
for more work, saying what is needed. Whether a sent-back statement is eventually abandoned is
the editors' decision — or a Coordinator's or the Secretariat's, who can mark any statement
dead (ISSUE-11).

#### Requirements

- The reviewer of an outgoing statement (ISSUE-7) has exactly two outcomes: complete the review,
  or **send back for rework**.
- Send back for rework:
  - requires a **comment**; an empty comment is rejected;
  - returns the statement to `draft`, where its editors can work on it and mark it ready again;
  - records a `sent_back` event carrying the comment, the reviewer, and the time;
  - notifies the submitter and the draft's editors with the comment, copying the Liaison
    Coordinators (ISSUE-28).
- A reviewer **cannot** mark a statement `dead`, directly or by acting on behalf of another
  role (ISSUE-18). Rejection remains available to approvers (ISSUE-9), and Coordinators and
  the Secretariat can mark any statement dead (ISSUE-11).
- A statement may be sent back any number of times; each round is recorded.
- **Approval is not erased by the send-back itself.** It is invalidated only if the content
  actually changes before the statement is readied again (ISSUE-13). A statement sent back
  over an addressing problem, fixed without touching content, returns straight to review.
- A statement that has been sent back has review history and cannot be hard-deleted
  (ISSUE-10).

#### Acceptance Criteria

- [ ] The review UI offers complete-review and send-back, and no reject or mark-dead action.
- [ ] Send-back without a comment is rejected.
- [ ] Send-back returns the statement to `draft` and records an event carrying the comment.
- [ ] The submitter and editors are notified with the comment.
- [ ] A reviewer cannot reach `dead`, including via act-on-behalf.
- [ ] Repeated send-backs are each recorded.
- [ ] A sent-back statement whose content is unchanged returns to review without re-approval;
      one whose content changed requires re-approval.
- [ ] A sent-back statement cannot be hard-deleted.

---

### ISSUE-13: Content changes after approval invalidate the approval

**Labels:** `workflow`, `outgoing`
**Priority:** High
**Milestone:** M1
**Origins:** new requirement; RFC 4053bis §4.1

#### Background

Approval attaches to what the statement says. If the text changes after an AD or Chair approved
it, the approval no longer covers what would be sent. This most often arises when a Liaison
Manager needs to change something during review.

The rule deliberately errs toward safety while there is no experience with the new system, and
is recorded as revisitable in the [Implementation Plan](proposed-implementation-plan.md) §7:
after some experience, some classes of edit may prove routine enough not to warrant another
approval round.

#### Requirements

- **Content fields** — changing any of these after an `approved` event invalidates it:
  - title,
  - body,
  - purpose and deadline,
  - attachments (added or removed),
  - the IETF group set and the peer group set (`ietf_groups`, `peer_groups`; ISSUE-20).
- **Addressing fields** — changing these does **not** invalidate approval:
  - From-Contact, From-Liaison-Contact,
  - To-Contact, To-Liaison-Contact.

  This preserves the approver's and reviewer's ability to fix addresses in place (ISSUE-7,
  ISSUE-9), which is a routine part of both steps.
- **Mechanism:** the tool compares the content fields against their values at the time of the
  approval event. Validity is derived, not a stored flag, so it cannot drift out of step with
  the content.
- **Effect:** an invalidated approval means the statement cannot be reviewed or posted until it
  is approved again. The statement stays `pending` — it is not sent back to `draft`
  automatically — and the approvers are notified that it needs approval again (ISSUE-28), along
  with the submitter.
- **Who this applies to:** everyone. A Liaison Manager editing content during review, an
  approver editing their own approved statement, and a Coordinator acting on behalf of either
  are all treated identically.
- **History:** the invalidation is recorded as an event naming who made the change and what
  changed, so the reason for a second approval round is visible.
- The same rule applies when a statement is resurrected from `dead` (ISSUE-11) or readied again
  after a send-back (ISSUE-12): a prior approval counts only if the content is unchanged.

#### Acceptance Criteria

- [ ] Changing any listed content field after approval invalidates it and blocks review and
      posting.
- [ ] Changing only addressing fields leaves the approval valid.
- [ ] Validity is derived by comparison, not stored as a flag.
- [ ] Approvers and the submitter are notified when an approval is invalidated.
- [ ] The invalidation event names who changed what.
- [ ] Re-approval restores the ability to review and post.
- [ ] Tests cover: content edit by a reviewer; addressing edit by a reviewer; content edit by an
      approver; edit during a send-back round; resurrection from dead.

---

### ISSUE-14: Record when a statement was received; order statements by the right date

**Labels:** `data-model`, `usability`, `incoming`
**Priority:** High
**Milestone:** M1
**Origins:** new requirement

#### Background

An incoming statement has two quite different dates: when the IETF received it from the peer
organization, and when someone recorded it in the datatracker. The current tool effectively has
only the second, so a statement entered late appears to have arrived late — and entering a
statement received weeks or months ago is a normal, expected occurrence, not an edge case.

An outgoing statement has no equivalent ambiguity: the date that matters is when it was posted
and sent.

#### Requirements

- **Incoming statements carry a received date**, set by the person entering the statement,
  defaulting to today and freely settable to a past date. It is required before the statement
  can be marked ready.
- A received date in the future is rejected.
- **Incoming statements are displayed with, and ordered by, their received date** — in listings,
  search results, and the API — not by when they were recorded.
- **Outgoing statements are displayed with, and ordered by, their posted date** (ISSUE-6).
- **When a statement was recorded in the datatracker is captured in its history**, for both
  directions, and remains available. It is never the sort key and is not the date shown in
  listings.
- **Incoming and outgoing statements are never mixed in one listing table.** A page that shows
  both directions shows them in **separate tables**. This covers the general statement list,
  per-organization and per-group pages, search results, and the in-flight and open-actions
  views (ISSUE-19). Each table has the columns that matter for its direction, and is sorted by
  its own direction's date.
  - Outgoing: originating IETF groups, target peer groups, state, and who it is waiting on.
    Sorted by posted date.
  - Incoming: sending peer groups, target IETF groups, deadline, and assignee (ISSUE-24).
    Sorted by received date.
- The one place directions appear together is a **thread view**: a statement with the
  statements it replies to and the replies to it, in chronological order. Each entry there
  shows its direction with a text label or icon **and** a distinguishing colour, never colour
  alone.
- Correcting a received date after posting follows the incoming correction path (ISSUE-17).
- Migrated incoming statements take their **posted date** as their received date, which
  disturbs the existing public record least. Where a statement's recorded history begins
  significantly before its posting, that default is likely wrong, and the migration flags it
  for reconciliation rather than leaving it as a silent approximation (ISSUE-39).

#### Acceptance Criteria

- [ ] An incoming statement requires a received date, which defaults to today and accepts past
      dates.
- [ ] A future received date is rejected.
- [ ] Incoming statements sort and display by received date everywhere, including the API.
- [ ] Outgoing statements sort and display by posted date.
- [ ] The datatracker recording time is in the history for both directions and is never the
      sort key.
- [ ] A statement received months earlier and entered today sorts by when it was received.
- [ ] No listing table contains both incoming and outgoing statements. Pages that show both use
      one table per direction, each with its own columns and sort date.
- [ ] The thread view shows replies in both directions chronologically, each entry labelled by
      direction with label or icon and colour.

---

### ISSUE-15: Capture the role the submitter is acting in

**Labels:** `usability`, `roles`
**Priority:** Medium
**Milestone:** M2
**Origins:** liaison-tooling-2025a #37

#### Background

liaison-tooling-2025a#37 requests that "the role the person is acting under when entering the
statement will be captured." A person may simultaneously be an AD, a WG Chair, and a Liaison
Manager; which capacity they are acting in determines the statement record, the addresses that
populate the form, and the applicable approval requirements.

#### Requirements

- The acting role is captured on every statement.
- The submitter is prompted to choose **only when it is genuinely ambiguous** — when they hold
  more than one applicable role relative to the selected groups. Otherwise the role is captured
  automatically with no prompt.
- Applicable roles are derived from the roles the person actually holds, from both sources:
  Liaison Manager appointments in the liaison application, and datatracker roles for everything
  else — including IAB Liaison Coordinator, which is a role in the `iab` group (ISSUE-1).
- The captured role's **address** — the WG chair alias, the group's liaison alias, and so on,
  rather than the individual's personal address — populates the sending form's contact fields
  (ISSUE-20, ISSUE-31).
- The captured role is stored on the statement and shown in the detail and history views.

#### Acceptance Criteria

- [ ] The acting role is captured for every statement.
- [ ] The submitter is prompted only when more than one applicable role exists.
- [ ] The captured role's address, not the individual's personal address, populates the form.
- [ ] The captured role is persisted and visible in the statement history.

---

### ISSUE-16: "Action taken" — who can mark it, and what it records

**Labels:** `roles`, `usability`
**Priority:** High
**Milestone:** M2
**Origins:** liaison-tooling-2025a #9, #16; ietf/datatracker #9438

#### Background

"Action taken" is a follow-up aid for the people responsible for follow-up: the responsible
Liaison Manager, or an IAB Liaison Coordinator where there is no Manager, and the statement's
assignee (ISSUE-24). It tells them, and the deadline reminders (ISSUE-29), that no further
follow-up is needed. RFC 4053bis §3.1 also recommends publicly recording actions taken on a
received statement.

liaison-tooling-2025a#9 requests that the Secretariat, IAB Liaison Coordinators, and the IAB
Chair can mark a statement "action taken". liaison-tooling-2025a#16 requests that marking it
can also add a comment, a file, and a reference to a related statement.

The current tool lets the Secretariat and Liaison Coordinators mark it, plus anyone whose
address appears in the statement's contact fields. It does not allow the IAB Chair, and it
offers the action only on statements with a deadline. The action stores only a tag change,
with no supporting information and no history event (ietf/datatracker#9438). The
contact-field rule fits the action's purpose poorly. For an outgoing statement, To-Contact
holds the *peer organization's* people, and the tool has no role for external parties
(ISSUE-3).

#### Requirements

**Who and when:**

- "Action taken" is offered only on statements whose purpose is "For Action" (ISSUE-23).
- The following can mark a statement "action taken":
  - the **Liaison Manager** of a relevant peer group: the sending group for an incoming
    statement, any target group for an outgoing statement;
  - IAB Liaison Coordinators;
  - the Secretariat;
  - the IAB Chair;
  - the statement's **assignee**, on an incoming statement (ISSUE-24).
- Others act through the act-on-behalf mechanism (ISSUE-18), which records who they acted for.
- Appearing in a statement's contact fields grants no right to mark action taken.

**What it records:**

- The action-taken form carries three optional fields: a comment, a file attachment, and a
  reference to a related statement.
- Marking action taken **always** records an event attributed to the person who did it, even
  when no optional field is supplied.
- Supplied comment, attachment, and reference are recorded against the statement and appear in
  its history.

**From a reply:** posting a reply can also mark the original action taken, at the replier's
choice (ISSUE-33).

#### Acceptance Criteria

- [ ] The action is offered only on "For Action" statements.
- [ ] A relevant Liaison Manager, a Coordinator, the Secretariat, the IAB Chair, and an
      incoming statement's assignee can each mark a statement "action taken".
- [ ] A person whose only connection to the statement is an address in its contact fields
      cannot.
- [ ] The form offers comment, file, and related-statement fields, all optional.
- [ ] An event is recorded even with no optional fields supplied, attributed to the person who
      acted.
- [ ] Supplied information appears in the statement history.

---

### ISSUE-17: Statements posted in error — what "wrong" means, and how each direction is corrected

**Labels:** `roles`, `workflow`, `usability`
**Priority:** Medium
**Milestone:** M2
**Origins:** liaison-tooling-2025a #35, #45

#### Background

liaison-tooling-2025a#35 asks how to handle statements posted in error, and #45 asks for
posted statements to be editable. There are two distinct cases:

- **The statement should not have been sent at all.** It is marked `dead` with a mandatory
  reason (ISSUE-11). It leaves public view and the API, and its public URL shows that it was
  removed and why. If it was already emailed, that communication cannot be recalled, and the
  tool does not pretend otherwise.
- **The content is wrong.** This issue covers that case. "Wrong" means different things in
  each direction, so correction is direction-specific.

RFC 4053bis §3 explains why hard deletion is never the answer: the value of a statement comes
from its public record, which "will attest that certain information has been communicated
between the organizations."

**What "wrong" means:**

- **Incoming — the record does not match what was received.** The IETF is recording another
  organization's communication, so the only thing that can be wrong is the recording.
  Examples: a transcription error in the title or body; a missing or wrong attachment; a wrong
  received date; wrong sending groups, contacts, or liaison contacts; a wrongly recorded
  purpose or deadline; wrong IETF target groups or To-Contacts.

  Correction is never a way to change what the other organization said. If the sender later
  issues a revised version (a "Rev.1", a corrigendum, a replacement attachment), the original
  record was accurate, and the revision is recorded as a **new incoming statement**.
- **Outgoing — the IETF should have said something else.** What was sent is what the IETF
  said, so the record cannot be wrong in the incoming sense. Any change, **including a typo
  fix**, is a new statement that goes through approval again.

  **Exception — outgoing statements not sent by the tool.** If an outgoing statement was
  posted with "Do not send email when posted" set (ISSUE-30), it was sent by other means and
  only recorded here. Its record can then be wrong in the incoming sense: it may not match
  what was actually sent. A change to what the IETF *said* is still a new statement.

#### Requirements

**Incoming statements:**

- The Liaison Manager (or an IAB Liaison Coordinator) can **correct a posted statement in
  place** when its record does not match what was received. A revised version issued by the
  sender is entered as a new incoming statement, and the original is left unchanged.
- The correction **issues mail** to the original parties, plus any new ones as needed.
- The correction mail and a history comment both carry a **summary of the correction** supplied
  by the person making it.
- History retains the prior content; corrections are recorded as events.
- Correcting the **received date** is a correction like any other and follows this path
  (ISSUE-14).

**Outgoing statements:**

- A posted outgoing statement is **not** edited in place, not even to fix a typo or to add or
  remove an attachment (ISSUE-34). A correction is entered as a **new statement** that goes
  through the normal approval and review process (ISSUE-9, ISSUE-7).
- Two exceptions:
  - the attachment safety hide (ISSUE-34);
  - an outgoing statement posted with "Do not send email when posted" set (ISSUE-30). If its
    record does not match what was actually sent, it is corrected in place exactly as an
    incoming statement is, attachments included. This is done by the Liaison Manager or a
    Coordinator, with a summary and mail to the original parties.

**Both directions:**

- A new statement that corrects or revises an earlier one may reference it **in prose**. The
  tool captures **no** "replaces" or other structured relationship. (Recorded as revisitable
  in the [Implementation Plan](proposed-implementation-plan.md) §7.)
- Comments may be added to the history of either statement.
- No hard-delete path exists for a statement that was ever posted (ISSUE-10, ISSUE-11).

#### Acceptance Criteria

- [ ] Incoming: a Liaison Manager or Coordinator can correct a posted statement in place when
      its record does not match what was received.
- [ ] An incoming correction mails the original parties (plus any new ones) and records a
      history comment, both carrying the corrector's summary.
- [ ] A sender's revised version of an incoming statement is entered as a new incoming
      statement, and the original is left unchanged.
- [ ] Outgoing: no in-place edit of a posted statement exists, not even for a typo. Correction
      is a new statement routed through approval.
- [ ] An outgoing statement posted without being sent by the tool can be corrected in place to
      match what was actually sent.
- [ ] No structured "replaces" link is stored.
- [ ] Comments can be added to the history of either statement.
- [ ] No hard-delete path exists for a statement that was ever posted.

---

### ISSUE-18: Acting on behalf of another role holder

**Labels:** `roles`, `workflow`
**Priority:** High
**Milestone:** M2
**Origins:** new requirement

#### Background

Several workflow steps need one role to act for another: a Coordinator recording an AD's
out-of-band approval (ISSUE-9), the Secretariat entering an incoming statement for a Liaison
Manager (ISSUE-8). Rather than special-casing each, the tool provides one mechanism.

#### Requirements

- The Secretariat, the IAB Chair, and IAB Liaison Coordinators may perform any role-restricted
  action on behalf of the role holder who would normally perform it.
- Every such action records **who acted** and **who they acted for**, both visible in the
  statement history.
- Where the specific case requires a comment — proxy approval must name the actual approver
  (ISSUE-9) — that requirement still applies.
- Acting on behalf does not change the attribution of automated posting, which remains
  `(System)` (ISSUE-6).
- One constraint is preserved from ISSUE-8: the Secretariat cannot both enter an incoming
  statement and post it. Acting on behalf does not create a route around a
  separation-of-duties rule.
- Reviewers gain nothing by this mechanism that reviewers do not have: acting on behalf of a
  reviewer still cannot reject or mark a statement dead (ISSUE-12).

#### Acceptance Criteria

- [ ] Secretariat, IAB Chair, and Coordinators can perform role-restricted actions on behalf of
      the responsible role holder.
- [ ] Each such action records both parties, visible in history.
- [ ] Case-specific comment requirements still apply.
- [ ] The Secretariat still cannot both enter and post the same incoming statement.
- [ ] The mechanism cannot be used to exceed the represented role's own authority.

---

### ISSUE-19: Surface owed and in-flight liaison work

**Labels:** `usability`, `roles`
**Priority:** High
**Milestone:** M2
**Origins:** new requirement

#### Background

Approval and review requests currently arrive by email and nowhere else, so a missed message is
a stalled statement. ADs already have a place they check — the existing AD dashboard view in
the datatracker — and liaison approvals owed by an AD belong there with the rest of their
queue.

The other approvers have no equivalent. The IETF Chair and the IAB Chair have no dashboard of
their own, so their pending approvals are surfaced inside the liaison application instead.

Knowing what is owed is not enough. ADs and Liaison Managers also need to see what is coming:
drafts being written for their groups, statements waiting on someone else, and statements
approved but not yet reviewed. That is the **in-flight** view.

#### Requirements

**Owed work — on the AD dashboard:**

- **Pending liaison approvals appear on the existing AD dashboard view.** An AD who is a
  required approver for a statement sees it there, alongside the rest of their queue, with
  enough context to act — the statement's title, originating group, target peer groups, and
  how long it has been waiting — and a direct link to the approval action. This extends the
  existing dashboard; it does not introduce a new one.
- The AD dashboard also links to the AD's in-flight view (below). In-flight statements that
  the AD does not yet owe are **not** listed on the dashboard itself.

**In-flight view — in the liaison application, one page per person:**

- Every in-flight statement (`draft` or `pending`) the person is entitled to see in their
  role:

  | Role | In-flight statements shown |
  |------|----------------------------|
  | AD | Outgoing statements whose originating groups are in the AD's area, that is, ones the AD would approve |
  | IETF Chair / IAB Chair | Outgoing statements the Chair would approve (from the IETF/IESG, or from the IAB) |
  | Liaison Manager | Outgoing statements targeting any of the Manager's peer groups, and incoming statements from them |
  | IAB Liaison Coordinator, Secretariat | All in-flight statements |

- Each entry shows the statement's title, direction, groups, state, **who it is waiting on**
  ("draft, not yet marked ready", "awaiting approval by …", "awaiting review by …"), and how
  long it has been waiting.
- **Outgoing and incoming are separate tables on the page** (ISSUE-14), each with its own
  columns: who it is waiting on for outgoing; deadline and assignee for incoming. A role that
  only ever sees one direction, such as an AD, gets only that table.
- **Items the person owes are listed first and highlighted.** For the IETF Chair, the IAB
  Chair, Liaison Managers, and Coordinators, that owed section is their pending-approval or
  awaiting-review list. Those lists are not separate pages.
- Drafts appear only to those entitled to see them (ISSUE-10). Dead statements do not appear.
  Posted statements appear only in the open-actions section below.
- **Open actions:** the same page lists posted "For Action" statements not yet marked action
  taken, where the person is the assignee (ISSUE-24), a responsible Liaison Manager, or, for
  Coordinators and the Secretariat, any of them. Each entry shows the deadline and assignee.
  It leaves the list once action taken is recorded (ISSUE-16, ISSUE-33).

**Queue behaviour:**

- An item leaves an owed list as soon as it is no longer owed: approved, sent back, marked
  dead, or approval satisfied by another required approver. It stays in the in-flight view
  until it is posted or dead.
- **An invalidated approval puts the statement back in the approvers' owed list** (ISSUE-13).
- These surfaces supplement the notification emails (ISSUE-28); they do not replace them.

#### Acceptance Criteria

- [ ] An AD with a pending liaison approval sees it on the existing AD dashboard view, with a
      link to act, and no new dashboard is introduced.
- [ ] The AD dashboard links to the AD's in-flight view and lists only owed approvals itself.
- [ ] Each role in the table sees exactly its in-flight statements, with drafts limited by
      ISSUE-10.
- [ ] Every in-flight entry shows who it is waiting on and for how long.
- [ ] The page also lists open actions (posted For Action statements not yet marked action
      taken) for the person as assignee, responsible Liaison Manager, or Coordinator/Secretariat.
- [ ] Owed items appear first and highlighted. For the Chairs, Liaison Managers, and
      Coordinators this is their pending-approval or review list.
- [ ] Items leave owed lists once no longer owed, and leave the in-flight view once posted or
      dead.
- [ ] An invalidated approval reappears in the approvers' owed list.

---

### ISSUE-20: Implement the RFC 4053bis contact field model

**Labels:** `data-model`, `rfc-compliance`, `email`
**Priority:** High
**Milestone:** M3
**Origins:** RFC 4053bis §2.1; liaison-tooling-2025a #8, #28, #29

#### Background

RFC 4053bis §2.1 defines the contact model the new statement schema implements directly. The
current tool has one From-Contact field, no liaison-contact fields, and two ad-hoc fields
(`technical_contacts`, `action_holder_contacts`) with no RFC basis.

A recurring source of confusion the new implementation must eliminate: **the fields recorded in
a statement are not email headers.** The statement carries addressing fields; the tool derives
headers from them when it sends mail. The two are documented separately in the
[README](README.md#contact-fields) and must be presented separately in the UI.

#### Requirements

**Statement fields** — the new statement model carries exactly these addressing fields:

| Field | RFC 4053bis name | Holds |
|-------|------------------|-------|
| `direction` | — | Outgoing (IETF → peer) or incoming (peer → IETF) |
| `ietf_groups` | From (outgoing) / To (incoming) | One or more IETF bodies: WG, Area, IETF, IESG, IAB. Datatracker `Group` records |
| `peer_groups` | To (outgoing) / From (incoming) | One or more peer groups (ISSUE-1). Each belongs to exactly one peer organization |
| `from_contact` | From-Contact | Contact address(es) of the originating body, including chairs, lists, responsible AD(s), and any additional experts or IETF participants who should be informed |
| `from_liaison_contact` | From-Liaison-Contact ("Send Reply To") | Reply routing address. Outgoing: a role or process address. Incoming: the address the statement was received from |
| `to_contacts` | To-Contact | Contact address(es) at the receiving body, including any additional liaison-management contacts and experts there |
| `to_liaison_contact` | To-Liaison-Contact ("Send To") | Outgoing: the peer organization's central intake address. Incoming: the IETF address the statement was received through |

- **Groups are stored by kind, not by side.** `ietf_groups` and `peer_groups` hold two
  different kinds of record, datatracker `Group` and liaison-app peer group (ISSUE-1).
  `direction` decides which set is presented as From and which as To. In these issues,
  "originating groups" means the From set and "target groups" means the To set for the
  statement's direction.
- **There is no stored organization field.** A statement's peer organization is derived from
  its peer groups. On the statement page and in listings, peer groups are displayed by names
  that include their organization (for example "ITU-T SG15"), and the organization is never
  shown as a separate field. A statement addressed to an organization as a whole uses that
  organization's org-level peer group (ISSUE-1).
- There is **no** `technical_contacts` field: technical experts belong in From-Contact per RFC
  4053bis. There is **no** free-text `action_holder_contacts` field. Follow-up is tracked by a
  person-based assignee on incoming "For Action" statements (ISSUE-24).
- There is **no** `cc_contacts` field. RFC 4053bis §2.1 gives every additional recipient a
  place. Additional IETF-side people, such as experts or other participants to inform, go in
  From-Contact, and additional people at the receiving body go in To-Contact. Recipients
  belonging to neither organization are not modelled. RFC 4053bis recommends a separate
  statement to each body.
- There is **no** `response_contacts` field. From-Liaison-Contact replaces it.
- Values of these four legacy fields on existing statements are preserved as legacy display
  data on those statements (ISSUE-39). A migrated `response_contacts` value also prepopulates
  a reply's To-Liaison-Contact (ISSUE-33).
- **Population:** fields are prepopulated from the acting role (ISSUE-15), the peer group's
  contact list (ISSUE-22), and liaison aliases (ISSUE-31). Responsible AD addresses are
  prepopulated into From-Contact as the statement is entered, rather than enforced as a
  separate always-CC rule at send time. The submitter sees and can adjust them.
- **Liaison contacts on incoming statements** (RFC 4053bis §2.1):
  - From-Liaison-Contact is **required** on a new incoming statement and records the email
    address the statement was originally received from. A reply to the statement is addressed
    there (ISSUE-33).
  - To-Liaison-Contact records the IETF address the statement was received through —
    `liaison-coordination@iab.org` or a Liaison Manager's liaison alias. It is prepopulated from
    the acting role (ISSUE-31).
  - After the statement is recorded, it is distributed to its To-Contacts (ISSUE-32).
- **Liaison contacts on replies:** prepopulated from the statement being answered (ISSUE-33).
- **Header derivation:** headers are generated from these fields per the mapping tables in the
  README, which differ by direction. In particular, To-Liaison-Contact **replaces** To-Contact
  in the `To` header when set — never both — and on incoming notifications the statement's
  From-Contact is not CC'd. See ISSUE-27 for the mandatory coordination CC.
- **Presentation:** the statement page shows the statement's own fields. Where the emitted
  email is shown (preview, ISSUE-30; history), it is presented as an email with headers, and
  is visibly distinct from the statement's fields.

#### Acceptance Criteria

- [ ] The statement model carries exactly the fields listed above and no ad-hoc contact fields.
- [ ] Every field is populated per the rules above on a new statement with no manual retyping.
- [ ] A new incoming statement cannot be saved without a From-Liaison-Contact, and its
      To-Liaison-Contact prepopulates with the acting role's address.
- [ ] Header derivation matches the README mapping tables for both directions, including the
      To-Liaison-Contact either/or rule.
- [ ] The statement page distinguishes statement fields from generated email headers.
- [ ] Migrated statements still display their legacy contact values (ISSUE-39).

---

### ISSUE-21: From-Contact is chosen from the sender's own addresses, with no role-based restrictions

**Labels:** `roles`, `usability`, `outgoing`
**Priority:** High
**Milestone:** M3
**Origins:** liaison-tooling-2025a #4, #8; analysis of the current implementation

#### Background

The current tool treats From-Contact differently by role: for most roles it disables the field
with a hardcoded value; for the IAB Chair and Liaison Coordinators it forces a specific initial
value; and a workaround was added to let the IAB override it on incoming statements.
liaison-tooling-2025a#8 asks that users be able to choose any address from their datatracker
account; #4 states that the IAB Chair, IETF Chair, and Liaison Coordinators should have no
field restrictions.

#### Requirements

- From-Contact is presented as a choice among the submitter's active registered addresses,
  prepopulated with the acting role's address (ISSUE-15, ISSUE-31).
- No role has the field disabled or forced to an uneditable value.
- The Secretariat, IAB Chair, IETF Chair, and Liaison Coordinators can enter any syntactically
  valid address.
- The same rules apply on incoming and outgoing statements; no direction-specific workaround
  exists.

#### Acceptance Criteria

- [ ] Every role can choose among their active registered addresses.
- [ ] No role sees a disabled or forced From-Contact field.
- [ ] Secretariat, IAB Chair, IETF Chair, and Liaison Coordinators can enter arbitrary valid
      addresses.
- [ ] Behaviour is the same for both directions.

---

### ISSUE-22: Maintain a To-Contact list per peer group

**Labels:** `data-model`, `usability`
**Priority:** Medium
**Milestone:** M3
**Origins:** liaison-tooling-2025a #31

#### Background

liaison-tooling-2025a#31 asks for a stored "To Contacts" list per external group, so that
To-Contact is prepopulated from the selected group rather than retyped on every statement. In
the new model these lists belong to peer groups (ISSUE-1), not to `Group` records.

#### Requirements

- Each peer group carries a maintained list of contact addresses.
- Selecting target groups on the statement form prepopulates To-Contact from the union of those
  groups' lists.
- The prepopulated value is editable before submission.
- Liaison Coordinators, the Secretariat, and the relationship's Liaison Manager can maintain
  these lists through the management UI (ISSUE-2).
- Changing a peer group's list does not alter statements already created.

#### Acceptance Criteria

- [ ] A contact list is stored per peer group.
- [ ] Selecting target groups prepopulates To-Contact from their lists.
- [ ] The prepopulated value is editable.
- [ ] Coordinators, Secretariat, and the responsible Liaison Manager can maintain the lists.
- [ ] Existing statements are unaffected by later list changes.

---

### ISSUE-23: Purpose values and deadlines follow RFC 4053bis

**Labels:** `data-model`, `rfc-compliance`
**Priority:** Medium
**Milestone:** M3
**Origins:** RFC 4053bis §2.2

#### Background

RFC 4053bis §2.2 defines exactly three purpose values, and ties a deadline to requests for
action:

> "For Information: The liaison statement is to inform the receiving body of something and
> expects no response. This includes calls for review comments if the expected response is
> optional.
>
> For Action: The liaison statement requests that the receiving body does something on the
> sender's behalf, usually within a stated time frame. This is also used if a document is
> sent out for comment, and the review feedback is expected in the stated time frame.
>
> In Response: The liaison statement includes a response to a liaison statement from the peer
> organization on one or more of its documents and expects no further response.
>
> Liaison statements that request action indicate a deadline when the action is required."

The current tool also offers "For Comment". The RFC's definitions absorb it: optional comments
are For Information, and comments expected by a date are For Action. "Other" has no RFC basis.

A deadline only means something when a response is expected by then, which is For Action.
That is the IETF's rule for what it sends. Peer organizations follow their own procedures, and
an incoming statement is recorded as it was received (ISSUE-17).

#### Requirements

- New statements offer exactly three purposes: For Information, For Action, In Response.
- **Outgoing statements:** the deadline field exists only when the purpose is For Action, and
  it is required there. Changing the purpose away from For Action clears the deadline. On an
  approved statement that is a content change under ISSUE-13, like any purpose change.
- **Incoming statements:** a deadline may be recorded with any purpose, because it is part of
  what was received. When the purpose is not For Action:
  - the deadline is displayed, but it triggers no reminders (ISSUE-29) and offers no
    "action taken" (ISSUE-16);
  - the entry form notes that if the sender expects a response by that date, For Action is
    probably the right label. The Liaison Manager decides.
- **Replies:** the reply form prefills "In Response" (ISSUE-33). A reply that itself requests
  something by a date is changed to For Action, at which point the deadline field appears.
- Form help text explains when to use For Information versus For Action, replacing the removed
  options.
- Statements migrated with "For Comment" or "Other", or with a deadline on a purpose other than
  For Action, retain and display those values. Legacy purposes cannot be set on a new statement
  or selected when editing (ISSUE-39).

#### Acceptance Criteria

- [ ] Only the three RFC values are selectable.
- [ ] An outgoing statement can have a deadline only when it is For Action, and must have one
      then.
- [ ] Changing an outgoing statement's purpose away from For Action clears its deadline.
- [ ] An incoming statement can record a deadline with any purpose. On a non-For-Action
      statement it causes no reminder and no "action taken" option.
- [ ] Migrated statements display their legacy purpose values and deadlines.
- [ ] A legacy purpose value cannot be set on any statement through the new UI or API.

---

### ISSUE-24: An assignee for follow-up on incoming "For Action" statements

**Labels:** `data-model`, `workflow`, `incoming`
**Priority:** Medium
**Milestone:** M3
**Origins:** RFC 4053bis §3.1; analysis of the current implementation

#### Background

The current tool has an `action_holder_contacts` field ("Who makes sure action is completed"):
free-text addresses that drive nothing and are rarely maintained.

RFC 4053bis §3.1 describes who is responsible for follow-up on a received statement, and
expects that responsibility to be handed on when appropriate:

> "The person recording the statement should also consider themself as the assignee to
> ensure any follow-up action, including potentially sending a response, are taken, if
> needed, or find another assignee, such a working group chair or area director."

The RFC no longer specifies tooling, but it does describe an assignee and a handover. The tool
records that assignee as a person. The assignment is useful only if it drives something, so it
determines who is reminded, who can close the action, and whose work list shows it.

#### Requirements

- **Incoming "For Action" statements carry an assignee**: one datatracker `Person`. Other
  statements have none.
- **Default:** the person who recorded the statement, as RFC 4053bis §3.1 says. When the
  Secretariat entered it on someone's behalf (ISSUE-18), the default is the person
  represented.
- **Reassignment:** the responsible Liaison Manager or an IAB Liaison Coordinator can reassign
  the action to any person, typically a WG chair or AD. Each reassignment is recorded in
  history with who made it. The new assignee is notified (ISSUE-32).
- **What the assignee gets:**
  - deadline reminders (ISSUE-29), in addition to the responsible Liaison Manager;
  - the right to mark the statement "action taken" (ISSUE-16);
  - the statement in their work view while the action is open (ISSUE-19).
- The responsible Liaison Manager, or a Coordinator where there is no Manager, keeps overall
  responsibility. Reassigning shares the work. It does not hand off oversight.
- Free-text action-holder addresses are not collected. Values on migrated statements are
  preserved as legacy display data (ISSUE-39). Migrated statements have no assignee, and their
  reminders go to the responsible Liaison Manager or Coordinators.

#### Acceptance Criteria

- [ ] A new incoming For Action statement gets an assignee defaulting to its recorder, or to
      the person represented when entered on someone's behalf.
- [ ] The responsible Liaison Manager or a Coordinator can reassign it to any person. The
      change is recorded and the new assignee is notified.
- [ ] The assignee receives deadline reminders, can mark action taken, and sees the open action
      in their work view.
- [ ] Statements of any other purpose or direction have no assignee.
- [ ] No free-text action-holder field exists on new statements. Migrated values still
      display.

---

### ISSUE-25: Statement targeting — one peer organization, one or more of its groups

**Labels:** `data-model`, `workflow`, `outgoing`
**Priority:** High
**Milestone:** M3
**Origins:** new requirement; RFC 4053bis §2.1

#### Background

RFC 4053bis §2.1 permits a statement to address several bodies but recommends against it:

> "A statement may be sent to multiple groups within one body or even multiple bodies.
> However, it is recommended to send separate statements to different bodies to avoid
> unnecessary interconnections."

The tool makes that recommendation a rule for new statements: a statement addresses **one peer
organization** and **one or more groups within it**. Reaching two organizations means two
statements.

This matters beyond the data model, because the number of target groups determines who reviews
the statement (ISSUE-7).

#### Requirements

- A statement records at least one peer group, and all its peer groups belong to the same
  peer organization. There is no separate organization field (ISSUE-20). The rule is
  enforced as a validation on the peer group set. It applies in both directions: for an
  outgoing statement the peer groups are its targets, for an incoming one they are its
  senders.
- The form can let the user pick the organization first to narrow the group choices, but that
  choice only filters the list and is not stored. Submitting peer groups from more than one
  organization is rejected.
- **Review consequence** (ISSUE-7): one target group with a Liaison Manager → that Manager
  reviews; more than one target group → an IAB Liaison Coordinator reviews, even when all
  target groups share the same Manager.
- **Notification consequence** (ISSUE-28): the Liaison Managers of **all** target groups are
  notified as the statement progresses.
- **Addressing consequence** (ISSUE-20, ISSUE-22): To-Contact is prepopulated from the union of
  the target groups' contact lists.
- Widening distribution before the statement is sent is done by adding addresses to
  From-Contact or To-Contact (ISSUE-20). These are addressing edits, which do not invalidate
  approval (ISSUE-13). There is no separate CC field.
- **The one-organization rule is a constraint on creating and editing statements, not a limit
  of the model.** The statement model can represent target groups belonging to more than one
  peer organization, because statements in the existing record may do so and must migrate
  intact and display correctly (ISSUE-39). What the new application refuses is *creating* such
  a statement:
  - the constraint is enforced in the create and edit paths, and in the API, not by the schema;
  - a migrated multi-organization statement displays all of its peer groups, each named with
    its organization, and its history and links behave normally;
  - editing a migrated multi-organization statement may not add organizations, and correcting
    one follows the normal correction path (ISSUE-17) — for outgoing statements that means a
    new statement, which will be single-organization.
  - **No new multi-organization statements, by any route.** The model's ability to hold them
    exists for migrated data only. They are not created by direct database entry either.
    That would bypass approval, review, history, and notifications, and ISSUE-4 forbids
    reaching `posted` without an approval event. Reaching two organizations means two
    statements, as RFC 4053bis §2.1 recommends.
  - The alternative — a separate set of legacy-only models for presenting such statements —
    was considered and rejected as more machinery than one relaxed constraint plus a
    validation rule. If the permissive model turns out to complicate the common path, that
    fallback remains available.
- Where the number of target *groups* determines behaviour (ISSUE-7 reviewer selection,
  ISSUE-28 notifications), a migrated multi-organization statement is treated as a multi-group
  statement: an IAB Liaison Coordinator handles it.

#### Acceptance Criteria

- [ ] A new statement can name several peer groups within one organization, in either
      direction.
- [ ] Creating or editing a statement with peer groups from two organizations is rejected, in
      the UI and through the API.
- [ ] No organization is stored on the statement. It is derived from the peer groups, and peer
      groups are displayed with organization-qualified names.
- [ ] The model can nonetheless represent a multi-organization statement, and a migrated one
      displays all its peer groups correctly.
- [ ] A migrated multi-organization statement is reviewed by an IAB Liaison Coordinator.
- [ ] Reviewer selection follows the group count per ISSUE-7.
- [ ] All target groups' Liaison Managers are notified.
- [ ] To-Contact prepopulates from the union of the target groups' lists.
- [ ] Coordinators and the Secretariat can widen distribution before sending by editing
      From-Contact or To-Contact, without invalidating approval.

---

### ISSUE-26: Send outgoing statement email from `liaison-coordination@iab.org`

**Labels:** `email`
**Priority:** Medium
**Milestone:** M4
**Origins:** liaison-tooling-2025a #1, #38; RFC 4053bis §2.1

#### Background

The current outgoing email From address is `statements@ietf.org`. RFC 4053bis §2.1 says of
From-Liaison-Contact ("Send Reply To"):

> "For liaison statements sent by the IETF, this address should be the alias of the liaison
> manager, if applicable, or an address maintained by the IAB for liaison management such
> as liaison-coordination@iab.org."

liaison-tooling-2025a#1 states external parties should not use `statements@ietf.org` as a
reply-to; #38 asks what address the tool should use.

Note the distinction this issue depends on: the statement's From-Liaison-Contact **field** and
the email's `Reply-To` **header** are different things, related by the mapping in ISSUE-20.

#### Requirements

- The `From` header on all liaison email is `liaison-coordination@iab.org`.
- The `Reply-To` header carries the statement's `from_liaison_contact` field, defaulting to
  `liaison-coordination@iab.org` when it is empty.
- No UI text directs external parties to `statements@ietf.org`.
- `statements@ietf.org` is retained as a **receive-only** alias for transition purposes and
  never appears in outgoing mail. This requires coordination with infrastructure.

#### Acceptance Criteria

- [ ] Every emitted liaison email has `From: liaison-coordination@iab.org`.
- [ ] `Reply-To` reflects the statement's From-Liaison-Contact, or the default when empty.
- [ ] `statements@ietf.org` appears in no outgoing header and in no UI text as a send address.

---

### ISSUE-27: Always CC `liaison-coordination@iab.org`

**Labels:** `email`
**Priority:** High
**Milestone:** M4
**Origins:** liaison-tooling-2025a #10, #42

#### Background

liaison-tooling-2025a#10 requests that `liaison-coordination@iab.org` be CC'd on all liaison
statements; #42 requests the same for approval request emails. This is a system requirement,
not a user-configurable option.

#### Requirements

- `liaison-coordination@iab.org` is in the `Cc` header of **every** email the tool emits about
  a liaison statement — statement sends, approval requests, review requests, send-back
  notices, state changes, corrections, and deadline reminders.
- It is CC'd **even when it is also the `From` address** of that email.
- Users cannot remove it: it is added at send time and is not part of any editable statement field.

#### Acceptance Criteria

- [ ] Every emitted email carries the address in `Cc`.
- [ ] It is present even when it is the `From` address.
- [ ] No form or API path can suppress it.
- [ ] A test enumerates every email the tool can emit and asserts the CC on each.

---

### ISSUE-28: State-change notifications for outgoing statements

**Labels:** `email`, `workflow`, `outgoing`
**Priority:** High
**Milestone:** M4
**Origins:** liaison-tooling-2025a #18; ietf/datatracker #8271

#### Background

liaison-tooling-2025a#18 requests email notification on state changes. This issue covers
**outgoing** statements; incoming notifications are ISSUE-32.

The principle is that the people with a stake in a statement see every step of it:
- the submitter, and the Liaison Managers of all target groups, even when a Coordinator is
  the reviewer (ISSUE-7);
- the person who took the action, as confirmation;
- the `liaison-coordination@iab.org` archive.

Each message is addressed To whoever has to act next. The send-back and approval-invalidation
paths need notifications of their own (ISSUE-12, ISSUE-13).

#### Requirements

**Who receives every notification.** Every notification about an outgoing statement has the
same base recipients, the **standard copy list**:

- the submitter;
- the Liaison Managers of all target groups, or the IAB Liaison Coordinators for any target
  group without a Manager;
- the person whose action triggered the notification, and, if they acted on someone's behalf,
  the person they represented (ISSUE-18);
- `liaison-coordination@iab.org`, the record-keeping copy (ISSUE-27).

Each transition then names who the message is **To**: the person or people who must act
next, or who are its primary audience. Everyone on the standard copy list who is not already
in To is in Cc. **Each person receives one copy.** Someone who is both in To and on the copy
list is addressed in To only.

| Transition | To | Cc |
|------------|----|----|
| Marked ready | The computed approvers (ISSUE-9) | Standard copy list |
| Approved | The reviewer for this statement (ISSUE-7) — the review window is now open | Standard copy list |
| Sent back for rework (ISSUE-12) | The submitter and all draft editors, carrying the reviewer's mandatory comment | Standard copy list, plus the Liaison Coordinators |
| Approval invalidated by a content edit (ISSUE-13) | The approvers who must approve again | Standard copy list |
| Reviewed and posted (ISSUE-6) | The submitter | Standard copy list |
| Marked dead | The submitter | Standard copy list, plus the Liaison Coordinators |

- **Review and posting are one notification.** Completing the review posts the statement
  immediately (ISSUE-6), so the internal notification covers both.
- **The statement email is separate.** Posting also sends the statement itself to the peer
  organization, with headers derived from the statement's fields exactly per ISSUE-20 and the
  README mapping. The standard copy list is **not** added to that email. The internal "reviewed
  and posted" notification is a separate message.
- **Changes made by others are reported to the submitter.** When anyone other than the
  submitter edits a statement after it was marked ready (an approver, a reviewer, a
  Coordinator, or the Secretariat acting on someone's behalf), the next notification lists
  every field that changed, content and addressing alike, with old and new values and who made
  each change. The submitter is on every notification, so that is simply the next one. Changes
  are reported at the transition, not on each save. Addressing changes are reported even
  though they do not invalidate approval (ISSUE-13). The notification is informational. A
  submitter who disagrees can edit the statement, and that edit is then subject to ISSUE-13.
- The internal "reviewed and posted" notification is sent **even when the exceptional "Do not
  send email when posted" option is set** (ISSUE-30). The option suppresses only the statement
  email to the peer organization.

#### Acceptance Criteria

- [ ] Every notification includes the full standard copy list: submitter; Liaison Managers of
      all target groups (or Coordinators where a group has none); the triggering person and
      anyone they acted for; `liaison-coordination@iab.org`.
- [ ] Each transition's To recipients match the table, and the extra Cc recipients on the
      send-back and dead rows are included.
- [ ] No person receives more than one copy of a notification. A person in To is not repeated
      in Cc.
- [ ] Completing a review produces one internal "reviewed and posted" notification, not two.
- [ ] The statement email to the peer organization carries only the headers derived per
      ISSUE-20. The standard copy list is not added to it.
- [ ] The send-back notification carries the reviewer's comment.
- [ ] When anyone other than the submitter has edited the statement since it was marked ready,
      the next notification lists each changed field (old and new values, and who changed
      it), for both content and addressing edits.
- [ ] Edits by the submitter alone produce no change report.
- [ ] The internal posted notification is sent even when the do-not-send option is set.

---

### ISSUE-29: Deadline reminder emails

**Labels:** `email`, `usability`
**Priority:** Medium
**Milestone:** M4
**Origins:** liaison-tooling-2025a #17

#### Background

liaison-tooling-2025a#17 requests automatic deadline reminders for statements with a deadline
and no "action taken" record.

#### Requirements

- A scheduled task identifies statements with purpose For Action, a deadline set, and no
  action-taken record, and sends reminders at configured intervals before the deadline.
- The reminder schedule is a **global datatracker setting** — not per-statement and not
  hardcoded.
- Reminders go to the parties responsible for follow-up: the responsible Liaison Manager, or the
  IAB Liaison Coordinators where no Manager is responsible, **and** the statement's assignee
  if it has one (ISSUE-24).
- Reminders stop once action taken is recorded.
- The reminder includes the statement title, its deadline, and a link.

#### Acceptance Criteria

- [ ] Reminders are sent by a scheduled task, not a synchronous management command.
- [ ] The schedule is a global configurable setting.
- [ ] Recipients are resolved from role data — Liaison Manager appointments, or the `iab`
      group's Liaison Coordinators — not from a stored contact field.
- [ ] No reminders are sent after action taken is recorded.

---

### ISSUE-30: Email is always sent on posting, with a guarded exception; email preview

**Labels:** `usability`, `email`
**Priority:** Medium
**Milestone:** M4
**Origins:** liaison-tooling-2025a #14; liaison-tooling-2025a README

#### Background

Posting a statement sends email. Outgoing statements are sent to the peer organization, and
incoming statements are distributed to their To-Contacts. That is the normal case, and the
tool should make it the only case most people ever see.

Two exceptional cases need a statement posted **without** that email:

- **Incoming, already distributed.** Some peer organizations, including some ITU-T groups,
  mail a statement directly to the IETF working groups, and it is recorded in the
  datatracker later. Redistributing it to the To-Contacts sends the groups a second copy and
  causes confusion.
- **Outgoing, already sent.** An outgoing statement sent by other means is recorded in the
  datatracker afterwards.

The proposed workflow also asks that the email be previewable: "The editing process will
clearly capture whether the statement should be emailed when posted, and allow preview of the
emailed message." The current tool has no preview.

#### Requirements

**Sending is the default and cannot be turned off casually:**

- Every statement is emailed on posting unless the "Do not send email when posted" option has
  been set. The option is **off by default**.
- Only **IAB Liaison Coordinators and the Secretariat** can set it. Others ask them, or act
  through ISSUE-18.
- Setting it is presented as an exceptional, potentially harmful action:
  - visually set apart as a warning, not an ordinary checkbox among the statement fields;
  - a **mandatory reason**, such as "distributed by ITU-T SG15 directly to the WGs on
    2026-08-02";
  - an explicit confirmation step;
  - a history event recording who set it and the reason.
- A statement posted with the option set shows a **visible marker** on its page: "Not
  distributed by the datatracker. Recorded after distribution by other means." The marker
  includes the reason.
- The option suppresses only the external email: for outgoing statements the statement email
  to the peer organization, and for incoming ones the To-Contact distribution. Internal
  notifications to the standard copy list are always sent (ISSUE-28, ISSUE-32).
- The option skips email, **not workflow**. An outgoing statement still needs a valid
  approval and review before it can be posted (ISSUE-4). Approval given outside the tool is
  recorded as a proxy approval (ISSUE-9).
- **Correction:** an outgoing statement posted with the option set was not sent by the tool,
  so its record can fail to match what was actually sent. It is corrected the way an incoming
  statement is, in place, when the record does not match what was sent (ISSUE-17).

**Preview:**

- A preview action renders the **exact** email that would be sent — headers, body, and
  attachment list — without sending it, and is available while editing without submitting the
  form.
- The preview presents the message as an email with headers, making the distinction between the
  statement's fields and the derived headers visible (ISSUE-20).
- When the do-not-send option is set, the preview says plainly that this email will not be
  sent.

#### Acceptance Criteria

- [ ] A new statement is emailed on posting unless the option is set. The option is off by
      default.
- [ ] Only Coordinators and the Secretariat can set the option. A reason and a confirmation
      are required, and a history event records both.
- [ ] A statement posted with the option set shows the "not distributed by the datatracker"
      marker and reason on its page.
- [ ] Internal notifications are sent even when the option is set.
- [ ] An outgoing statement cannot reach `posted` without approval and review, with or without
      the option.
- [ ] An outgoing statement posted with the option set can be corrected in place to match what
      was actually sent.
- [ ] Preview renders the full message including headers, without sending, and matches the sent
      message exactly.
- [ ] Preview is reachable without submitting the form.

---

### ISSUE-31: Liaison aliases per peer group, and role-based address prepopulation

**Labels:** `data-model`, `email`, `roles`
**Priority:** Medium
**Milestone:** M4
**Origins:** new requirement

#### Background

The tool maintains a set of email aliases for liaison work — one alias per peer group (example:
`unicode-liaison@ietf.org`). These aliases are what should appear in statement contact fields,
rather than individuals' personal addresses.

An alias exists to reach the people responsible for that relationship. Those are its
appointed Liaison Managers (ISSUE-1), who are datatracker people with accounts. Building the
alias from appointments keeps it current as appointments start and end, the way datatracker
role aliases follow roles. A hand-maintained list would drift out of date.

#### Requirements

- Each peer group (ISSUE-1) may carry one liaison alias.
- **Every alias recipient is a datatracker person.** Each recipient address is an `Email`
  record belonging to a `Person`. Mailing lists, shared mailboxes, other aliases, and
  addresses with no person behind them cannot be recipients.
- **Recipients follow appointments.** The alias always includes the group's currently
  appointed Liaison Manager(s), each at the address their appointment designates (by default
  their primary address). When an appointment starts or ends (ISSUE-1), the alias changes
  with it. No one edits it by hand.
- **Additional recipients** may be added through the management UI (ISSUE-2), subject to the
  same person-backed rule. They are listed separately from the appointment-derived
  recipients.
- **A group with no appointed Liaison Manager** has its alias deliver to
  `liaison-coordination@iab.org`, as well as to any additional recipients. This follows the
  rule that a Coordinator acts where there is no Manager. Mail to the alias never bounces for
  lack of recipients.
- The datatracker keeps the mail system's alias in sync with the recipient set: the alias must
  actually deliver mail, not merely exist as datatracker metadata.
- **Prepopulation** — wherever the corresponding address would be supplied on a form:

  | Acting as | Address used |
  |-----------|--------------|
  | Liaison Manager (group has an alias) | that group's liaison alias |
  | IAB Liaison Coordinator | `liaison-coordination@iab.org` |
  | IAB Chair | `iab-chair@iab.org` |
  | IETF Chair | `chair@ietf.org` |

- Prepopulation ties into ISSUE-15 (the captured role's address populates the form) and ISSUE-20
  (contact-field population). Prepopulated values remain editable.
- This rule applies to alias recipients only. Peer-group contact lists (ISSUE-22) and statement
  contact fields (ISSUE-20) hold addresses of external parties and are not restricted to
  datatracker records.

#### Acceptance Criteria

- [ ] One alias per peer group.
- [ ] Every alias recipient is an `Email` record belonging to a `Person`. Any other address is
      rejected.
- [ ] The alias includes the group's currently appointed Liaison Managers, and changes
      automatically when an appointment starts or ends.
- [ ] Additional recipients can be added through the management UI, and are shown separately.
- [ ] A group with no appointed Liaison Manager has its alias deliver to
      `liaison-coordination@iab.org`.
- [ ] The mail-system alias stays in sync with the recipient set.
- [ ] Form fields prepopulate per the table above, and remain editable.

---

### ISSUE-32: Notifications for incoming statements

**Labels:** `email`, `workflow`, `incoming`
**Priority:** High
**Milestone:** M4
**Origins:** liaison-tooling-2025a #18

#### Background

Incoming statements have fewer events than outgoing ones, so their notification set is smaller.
The principle is the same as for outgoing statements (ISSUE-28): the people with a stake see
every step, the person who acted gets confirmation, and each message is addressed To whoever
is its primary audience.

#### Requirements

**Who receives every notification.** Every notification about an incoming statement includes
the **standard copy list**:

- the submitter (the person who entered the statement);
- the Liaison Manager of the sending peer group(s), or the IAB Liaison Coordinators where
  there is no Manager;
- the person whose action triggered the notification, and, if they acted on someone's behalf,
  the person they represented (ISSUE-18);
- `liaison-coordination@iab.org` (ISSUE-27).

Each person receives one copy. Someone who is in To is not repeated in Cc.

| Event | To | Cc |
|-------|----|----|
| Entered (including a Secretariat-on-behalf draft) | The sending group's Liaison Manager, or the Coordinators where there is none | Standard copy list, plus the Liaison Coordinators |
| Marked ready / posted | The statement's To-Contact recipients (ISSUE-20) | Standard copy list |
| Corrected after posting (ISSUE-17) | The original To-Contact recipients, plus any new ones, carrying the corrector's summary | Standard copy list |
| Marked dead (ISSUE-11) | The submitter | Standard copy list, plus the Liaison Coordinators |
| Assignee set or changed (ISSUE-24) | The new assignee | Standard copy list, plus the previous assignee |

- **The posting notification is the distribution.** Unlike outgoing statements, posting an
  incoming statement sends nothing outside the IETF. Its distribution to the To-Contacts
  carries the standard copy list directly. No separate internal notice is sent.
- Because the entry notification is sent while the statement is still a draft, an incoming
  statement is never eligible for hard deletion (ISSUE-10). One entered by mistake is marked
  `dead`.
- The notification to the standard copy list is sent **even when the exceptional "Do not send
  email when posted" option is set** (ISSUE-30). In that case only the To-Contact distribution
  is withheld.
- The statement's From-Contact and To-Liaison-Contact are recorded on the statement but are
  **not** recipients of these notifications (ISSUE-20). The sender is not notified of the
  IETF's internal handling.

#### Acceptance Criteria

- [ ] Every notification includes the full standard copy list.
- [ ] Each event's To recipients match the table, including the extra Cc on entry and dead.
- [ ] No person receives more than one copy of a notification.
- [ ] The standard copy list is notified even when the do-not-send option is set.
- [ ] A post-posting correction notifies the original parties with the corrector's summary.
- [ ] From-Contact and To-Liaison-Contact are not recipients of incoming notifications.

---

### ISSUE-33: Replies — only to posted statements, addressed from the original

**Labels:** `workflow`, `usability`, `rfc-compliance`
**Priority:** Medium
**Milestone:** M5
**Origins:** liaison-tooling-2025a #36; RFC 4053bis §2.1; ietf/datatracker #9387

#### Background

liaison-tooling-2025a#36 questions whether replies should be possible on statements that are
not posted. RFC 4053bis treats responses as responses to publicly recorded statements. The
current tool allows a reply regardless of state.

RFC 4053bis §2.1 also says where a reply goes:

> "If a 'Send Reply To' address is provided, the expectation is that a statement sent in
> response uses this address as the To-Liaison Contact."

The same section says that every statement the IETF receives records the address it came from
as its From-Liaison-Contact (ISSUE-20). So any incoming statement has an address a reply can
be sent to.

#### Requirements

- A reply can be created only against a statement in the `posted` state.
- The reply action is absent from the UI for statements in any other state.
- **Addressing is prepopulated from the original statement:**

  | Reply field | Prepopulated from the original's |
  |-------------|----------------------------------|
  | To-Liaison-Contact | From-Liaison-Contact |
  | Peer groups and IETF groups | The same peer groups and IETF groups. The reply's direction is the opposite of the original's, so From and To swap automatically (ISSUE-20) |
  | To-Contact | From-Contact |
  | Purpose | — set to "In Response" (ISSUE-23) |
  | Related statement | — set to the original |

- If the original has no From-Liaison-Contact (a migrated statement), To-Liaison-Contact is
  prepopulated from the original's legacy response-contact value, if there is one (ISSUE-39).
- The submitter sees every prepopulated value and can change it.
- **A reply can close the original's action** (ietf/datatracker#9387). When the original is
  "For Action" and not yet marked action taken (ISSUE-16), the reply form carries a checkbox,
  "This reply completes the requested action", **checked by default**. When the reply is
  **posted** with the box checked, the original is marked action taken:
  - the event is attributed to `(System)` and references the reply;
  - this applies in both directions: an IETF reply closes an incoming request, and a recorded
    peer reply closes an outgoing IETF request;
  - an interim reply ("we are working on it") is posted with the box unchecked, so the
    action stays open and deadline reminders (ISSUE-29) continue;
  - nothing is marked when a reply is created or marked ready. Only posting marks it.
- Fields not listed above follow the normal rules for the reply's direction (ISSUE-20). In
  particular, a reply recorded as an incoming statement still takes its From-Liaison-Contact
  from the address it was actually received from.

#### Acceptance Criteria

- [ ] Reply is unavailable for `draft`, `pending`, and `dead` statements, in the UI and in any
      direct request.
- [ ] Reply is available for `posted` statements to the roles entitled to create one.
- [ ] A reply's To-Liaison-Contact is prepopulated with the original's From-Liaison-Contact,
      falling back to the legacy response-contact value on migrated statements.
- [ ] The other fields in the table above are prepopulated as listed, and every prepopulated
      value can be edited before submission.
- [ ] Posting a reply with "completes the requested action" checked marks a "For Action"
      original as action taken, attributed to `(System)` and referencing the reply, in both
      directions.
- [ ] Posting a reply with the box unchecked leaves the original open.
- [ ] The checkbox is absent when the original is not "For Action" or is already marked.

---

### ISSUE-34: Changing attachments after a statement is created

**Labels:** `attachments`, `usability`
**Priority:** Medium
**Milestone:** M5
**Origins:** liaison-tooling-2025a #43; analysis of the current implementation (attachment
removal disabled in the UI with the comment "will be replaced in next generation of liaison
tool")

#### Background

The current tool allows attachments only during the initial create/edit flow, and its UI
disables attachment removal, with a comment saying this will be replaced in the next
generation of the tool. liaison-tooling-2025a#43 asks that attachments can be added to existing
statements, including posted ones.

An attachment is part of a statement's content, so adding or removing one follows the same
rules as any other change to the statement (ISSUE-13, ISSUE-17). For posted incoming statements
that means the request in #43 is met as a correction. For posted outgoing statements it is not
adopted: anything sent in addition is a new statement.

One case needs its own path: a file that must not stay publicly available, for example because
it is malicious or because its publication is legally problematic.

#### Requirements

**When an attachment can be added or removed:**

| Statement | Adding or removing allowed |
|-----------|----------------------------|
| `draft` or `pending`, either direction | As ordinary editing by the statement's editors, at any point before posting. Once the statement is approved, either change is a content change and invalidates the approval (ISSUE-13) |
| `posted` incoming | Only as a correction under ISSUE-17, when the recorded files do not match what was received: a received file is missing, or the wrong file was recorded. The Liaison Manager or a Coordinator acts. The correction mail and history comment carry the corrector's summary. A file the sender issues later, such as a replacement attachment or corrigendum, is a new incoming statement |
| `posted` outgoing | **Never.** What was sent is what the IETF said. Additional or different material is a new statement through approval and review (ISSUE-17). The one exception: if the statement was posted without being sent by the tool (ISSUE-30), files are corrected to match what was actually sent, as for incoming. The operational safety hide below also applies |
| `dead` | No |

**Removal never deletes.** Removal hides the attachment from public views and from `/api/v1`.
Neither the attachment record nor the stored file is hard-deleted.

**History:** every addition and removal records an event naming who made it (and on whose
behalf, ISSUE-18).

**Safety exception (any posted statement):** a Liaison Coordinator or the Secretariat can hide
an attachment that must not remain publicly available, such as malware or legally required
removal.
- **This is an operational safeguard, not part of the liaison process.** It exists to deal with
  a harmful file. It is not a way to correct, withdraw, or change what a statement says. Those
  follow the rules in the table above, the same as any other change to a statement.
- A reason is mandatory.
- Where the file was, the public statement page shows "This attachment has been removed"
  with the reason, matching the treatment of removed statements (ISSUE-11).
- The statement is not re-sent, and its workflow is not reopened.
- Removing content permanently, from the record or the blobstore, is an operational database
  action outside the tool (ISSUE-11).

Other roles act through the act-on-behalf mechanism (ISSUE-18), which records who they acted
for.

#### Acceptance Criteria

- [ ] Attachments on draft and pending statements can be added and removed by their editors.
      Either change after approval invalidates the approval.
- [ ] A posted incoming statement's attachments can be added or removed only as an ISSUE-17
      correction, with the correction mail and summary.
- [ ] No path adds or removes an attachment on a posted outgoing statement, except a
      correction to one posted without being sent by the tool (ISSUE-30) and the safety hide.
- [ ] Attachments cannot be added to or removed from dead statements.
- [ ] The safety hide is available only to Coordinators and the Secretariat, requires a reason,
      and shows a public notice with the reason.
- [ ] A removed attachment disappears from public views and the API. No record or file is
      hard-deleted.
- [ ] A history event records each addition and removal, including any on-behalf party.

---

### ISSUE-35: Comments — a standalone action, and comments carried by other actions

**Labels:** `workflow`, `usability`, `roles`
**Priority:** Medium
**Milestone:** M5
**Origins:** analysis of the current implementation (comment view restricted to Secretariat
while the template offers the action more widely)

#### Background

In the current tool the add-comment view is restricted to the Secretariat, while the template
shows the button to a broader set of roles — so those users see the action and then receive a
permission error. The new implementation should be consistent, and the useful behaviour is the
broader one.

There are two kinds of comment, and both are needed:

- a **standalone comment**, added to a statement at any time, for coordination or the record;
- a **comment carried by an action**, entered as part of that action.

Both land in the same statement history.

#### Requirements

**Standalone comments:**

- Anyone who can see the statement while it is in flight can add a comment, as can anyone who
  could while it is posted. Under ISSUE-10 that is its editors, its approvers, the Liaison
  Managers of every relevant peer group, IAB Liaison Coordinators, and the Secretariat. This
  includes Liaison Managers of target groups who are not the reviewer, so they can coordinate.
- The UI shows the action to exactly the set of people who can perform it.
- Comment text is required and cannot be empty.
- Comments can be marked private, which excludes them from the public history view while
  keeping them visible to those with a role on the statement.

**Comments carried by actions** are entered on the action's own form and recorded on that
action's history event:

| Action | Comment |
|--------|---------|
| Proxy approval (ISSUE-9) | Required; names the actual approver |
| Send back for rework (ISSUE-12) | Required |
| Mark dead (ISSUE-11) | Required reason; shown publicly if the statement was once posted |
| Correction of a posted statement (ISSUE-17) | Required summary; included in the correction mail |
| Do not send email when posted (ISSUE-30) | Required reason; shown in the statement's marker |
| Attachment safety hide (ISSUE-34) | Required reason; shown publicly in place of the file |
| Mark action taken (ISSUE-16) | Optional |
| Approve, review, mark ready | Optional |

- An action-carried comment follows the visibility of its action. It cannot be marked private
  separately.

#### Acceptance Criteria

- [ ] Everyone ISSUE-10 lets see a statement can add a standalone comment, including
      non-reviewing Liaison Managers of target groups.
- [ ] The UI offers the action to exactly that set, with no permission error on submission.
- [ ] An empty standalone comment is rejected.
- [ ] Private standalone comments are hidden from the public history and visible to authorized
      users.
- [ ] Each action in the table accepts its comment, enforces "required" where listed, and records
      the comment on that action's history event.

---

### ISSUE-36: Statement bodies render exactly as submitted

**Labels:** `usability`
**Priority:** High
**Milestone:** M5
**Origins:** ietf/datatracker #10129

#### Background

The current detail view applies a filter that detects lines longer than 80 characters and
reformats them, so the rendered body can differ from what was submitted — and does so only for
some statements, making display behaviour inconsistent and template changes unpredictable.

#### Requirements

- Statement bodies are displayed exactly as submitted. No conditional reformatting, line
  wrapping, or re-flowing is applied in any view.
- Long lines are handled by presentation (the container scrolls or wraps visually) rather than
  by altering the stored text.

#### Acceptance Criteria

- [ ] A statement with lines longer than 80 characters renders exactly as submitted.
- [ ] No view or template applies a reformatting filter to the body.
- [ ] Long lines do not break page layout.

---

### ISSUE-37: Store and serve attachments from the datatracker blobstore

**Labels:** `attachments`, `migration`
**Priority:** High
**Milestone:** M5
**Origins:** ietf/datatracker #7087, #6910

#### Background

Liaison attachments are currently served from `https://www.ietf.org/lib/dt/documents/LIAISON/`,
a legacy CDN path routed through www.ietf.org infrastructure. ietf/datatracker#6910 notes this
is unsustainable through planned IT infrastructure migrations; #7087 notes the CDN rationale no
longer applies. Existing references in RFC errata and external documents point at the legacy
URLs and must keep resolving.

#### Requirements

- Attachments are stored in the datatracker blobstore and served through datatracker
  infrastructure, as close to the edge as practical.
- Legacy `www.ietf.org/lib/dt/documents/LIAISON/` URLs redirect to the new locations,
  permanently.
- Existing attachment files are migrated into the blobstore (ISSUE-39).
- No UI link points at the legacy path.

#### Acceptance Criteria

- [ ] New uploads are stored in the blobstore and served by the datatracker.
- [ ] Every legacy attachment URL resolves via redirect.
- [ ] No UI link references the legacy path.
- [ ] Migration accounts for every existing attachment file.

---

### ISSUE-38: Form input survives validation errors

**Labels:** `usability`
**Priority:** Medium
**Milestone:** M5
**Origins:** ietf/datatracker #5340

#### Background

In the current tool, when the submission form fails validation — for example an invalid address
in the CC field — the CC list is silently reset to its default rather than re-populated with
what the user typed. The submitter may not notice and may resubmit with unintended recipients.
This is a data-loss-shaped bug and the new forms must not reproduce it.

#### Requirements

- On a validation error, every field re-renders with the values the user submitted, never with
  defaults. This applies to all fields, with the contact fields being the cases that motivated
  the requirement.
- The error message identifies which field failed and why.

#### Acceptance Criteria

- [ ] A validation error on any field re-renders the whole form with the submitted values
      intact.
- [ ] No field is silently reset to a default on error.
- [ ] The error identifies the offending field and reason.

---

### ISSUE-39: Migrate existing liaison data into the reimplemented schema

**Labels:** `migration`, `data-model`
**Priority:** High
**Milestone:** M6
**Origins:** new requirement

#### Background

Everything already in the liaison tool — statements going back years, their attachments, and
their history — must land in the new schema intact. The public record is the point of the tool
(RFC 4053bis §3), so migration is a correctness requirement, not a chore at the end.

The full sequence, validation approach, and cutover steps are in the
[Implementation Plan](proposed-implementation-plan.md) §3–§4. This issue is where the work is
tracked and its acceptance stated.

#### Requirements

- **Peer organizations and groups** migrate from SDO `Group` records, preserving acronyms and
  names so existing statement references resolve (ISSUE-1).
- **Relationships and Liaison Manager appointments** migrate from the `liaiman` `Role` rows on
  those groups. Each migrated appointment covers exactly the peer group whose `Role` row it
  came from. A person holding `liaiman` on several groups of one organization becomes one
  appointment covering those groups (ISSUE-1). Authorized Individual (`auth`) assignments are **not** carried forward as
  entry-capable appointments and are itemised in the migration report (ISSUE-3).
- **Liaison Coordinator roles are not migrated and not touched.** They are
  `liaison_coordinator` `Role` rows in the `iab` group and stay exactly where they are
  (ISSUE-1). The migration must not create coordinator appointments, and must not remove or
  rewrite those role rows.
- **Statements** migrate with direction, contact fields, purpose, deadline, body, and state:
  - legacy `technical_contacts`, `action_holder_contacts`, and `cc_contacts` values are
    preserved as legacy display data on those statements (ISSUE-20, ISSUE-24);
  - the legacy `response_contacts` value ("Where to send a response") is preserved as legacy
    display data. It is not backfilled into From-Liaison-Contact, but a reply to the statement
    uses it to prepopulate its To-Liaison-Contact (ISSUE-33);
  - legacy purpose values ("For Comment", "Other") are preserved for display (ISSUE-23);
  - **incoming statements take their posted date as their received date** (ISSUE-14) — see
    the reconciliation requirement below;
  - `not-approved` statements become `dead`, with an event recording the original state
    (ISSUE-11);
  - **statements addressed to more than one peer organization migrate as they are**, using the
    model's ability to represent them (ISSUE-25). They are not split, collapsed, or dropped.
  - each statement's current `from_groups` and `to_groups` are sorted by kind. IETF groups
    go to `ietf_groups`, and SDO groups go to `peer_groups` via the migrated peer group
    (ISSUE-1, ISSUE-20). A statement whose groups cannot be sorted cleanly, for example one
    with no SDO group or with IETF groups on both sides, is listed in the migration report
    rather than guessed;
- **History is preserved in full.** Every event on every statement migrates with its original
  type, actor, timestamp, and text — including comments, private comments, approvals,
  postings, and state changes. Migrated history is not summarised, re-attributed, or
  re-timestamped, and events whose type has no counterpart in the new workflow are carried as
  historical events rather than discarded. A migrated statement's history reads the same after
  the rewrite as before it.
- **Received-date reconciliation.** Using the posted date as the received date is the right
  default — it disturbs the public record least — but it is wrong for any statement that was
  back-entered. The migration therefore **flags every incoming statement whose recorded history
  begins significantly before its posted date** (the threshold is configurable; the gap between
  the earliest event and the posting is the signal). Flagged statements are listed for the
  Liaison Coordinators to reconcile as a defined step, not left as a silent approximation. See
  the [Implementation Plan](proposed-implementation-plan.md) §3.1.
- **Attachments** migrate into the blobstore with legacy URLs redirecting (ISSUE-37).
- **A migration report** is produced on every rehearsal run: counts by model and state, every
  approximated or dropped value, the flagged received-date list, the multi-organization
  inventory, and rendering diffs for a sample of statements spanning the full history.
- Migration is rehearsed against copies and is reversible until cutover.

#### Acceptance Criteria

- [ ] Every existing statement, attachment, and event is present after migration, with original
      attribution and timestamps.
- [ ] A migrated statement's history renders the same content, in the same order, with the same
      actors and times, as it did before migration — including comments and private comments.
- [ ] No event is dropped for having a type the new workflow does not produce.
- [ ] Existing public statement URLs resolve to the same statements.
- [ ] Legacy attachment URLs redirect to blobstore-served files.
- [ ] Legacy contact and purpose values display on migrated statements.
- [ ] Every migrated incoming statement has a received date taken from its posted date.
- [ ] Every incoming statement whose history begins significantly before its posted date is
      flagged for reconciliation, and the threshold is configurable.
- [ ] `not-approved` statements are `dead` with the original state recorded.
- [ ] Multi-organization statements migrate intact and display all their peer groups.
- [ ] Every migrated statement's groups are sorted into `ietf_groups` and `peer_groups`, and
      any statement that cannot be sorted cleanly is itemised in the migration report.
- [ ] Authorized Individual assignments are itemised and none silently becomes an
      entry-capable appointment.
- [ ] `liaison_coordinator` roles in the `iab` group are unchanged by the migration, and every
      person who was a Liaison Coordinator before it still is afterwards.
- [ ] The migration report is produced on every run and reviewed before cutover.

---

## Summary

Grouped by milestone; see the [Implementation Plan](proposed-implementation-plan.md) for how
the milestones are phased.

**M0 – Foundation**

| Issue | Title |
|-------|-------|
| ISSUE-1 | Peer organization, peer group, relationship, and Liaison Manager appointment models |
| ISSUE-2 | Management UI for peer organizations, relationships, and appointments |
| ISSUE-3 | No external-party entry role |

**M1 – Statement lifecycle**

| Issue | Title |
|-------|-------|
| ISSUE-4 | No approval bypass — every outgoing statement is approved in the tool |
| ISSUE-5 | Statement lifecycle — states, transitions, and event-based gate tracking |
| ISSUE-6 | Posting is triggered automatically by the last human action; attributed to (System) |
| ISSUE-7 | Single review after approval — LM for one group, LC for several |
| ISSUE-8 | Incoming statements post when the LM (or LC) marks them ready; no second review |
| ISSUE-9 | Approval authority for outgoing statements |
| ISSUE-10 | Draft statements |
| ISSUE-11 | The `dead` state — rejection, retraction, and abandonment |
| ISSUE-12 | Reviewers send statements back for rework; they cannot reject |
| ISSUE-13 | Content changes after approval invalidate the approval |
| ISSUE-14 | Record when a statement was received; order statements by the right date |

**M2 – Roles and permissions**

| Issue | Title |
|-------|-------|
| ISSUE-15 | Capture the role the submitter is acting in |
| ISSUE-16 | "Action taken" — who can mark it, and what it records |
| ISSUE-17 | Statements posted in error — what "wrong" means, and how each direction is corrected |
| ISSUE-18 | Acting on behalf of another role holder |
| ISSUE-19 | Surface owed and in-flight liaison work |

**M3 – Contact fields and targeting**

| Issue | Title |
|-------|-------|
| ISSUE-20 | Implement the RFC 4053bis contact field model |
| ISSUE-21 | From-Contact chosen from the sender's own addresses, no role restrictions |
| ISSUE-22 | Maintain a To-Contact list per peer group |
| ISSUE-23 | Purpose values and deadlines follow RFC 4053bis |
| ISSUE-24 | An assignee for follow-up on incoming "For Action" statements |
| ISSUE-25 | Statement targeting — one peer organization, one or more of its groups |

**M4 – Email and notifications**

| Issue | Title |
|-------|-------|
| ISSUE-26 | Send outgoing statement email from `liaison-coordination@iab.org` |
| ISSUE-27 | Always CC `liaison-coordination@iab.org` |
| ISSUE-28 | State-change notifications for outgoing statements |
| ISSUE-29 | Deadline reminder emails |
| ISSUE-30 | Email is always sent on posting, with a guarded exception; email preview |
| ISSUE-31 | Liaison aliases per peer group, and role-based address prepopulation |
| ISSUE-32 | Notifications for incoming statements |

**M5 – Presentation**

| Issue | Title |
|-------|-------|
| ISSUE-33 | Replies — only to posted statements, addressed from the original |
| ISSUE-34 | Changing attachments after a statement is created |
| ISSUE-35 | Comments — a standalone action, and comments carried by other actions |
| ISSUE-36 | Statement bodies render exactly as submitted |
| ISSUE-37 | Store and serve attachments from the datatracker blobstore |
| ISSUE-38 | Form input survives validation errors |

**M6 – Migration and cutover**

| Issue | Title |
|-------|-------|
| ISSUE-39 | Migrate existing liaison data into the reimplemented schema |

---

## Attribution Index

| Origin | Referenced in |
|--------|--------------|
| liaison-tooling-2025a #1 | ISSUE-4, ISSUE-26 |
| liaison-tooling-2025a #2 | ISSUE-9 |
| liaison-tooling-2025a #4 | ISSUE-21 |
| liaison-tooling-2025a #8 | ISSUE-20, ISSUE-21 |
| liaison-tooling-2025a #9 | ISSUE-16 |
| liaison-tooling-2025a #10 | ISSUE-27 |
| liaison-tooling-2025a #14 | ISSUE-10, ISSUE-30 |
| liaison-tooling-2025a #16 | ISSUE-16 |
| liaison-tooling-2025a #17 | ISSUE-29 |
| liaison-tooling-2025a #18 | ISSUE-28, ISSUE-32 |
| liaison-tooling-2025a #28 | ISSUE-20, ISSUE-21 |
| liaison-tooling-2025a #29 | ISSUE-20 |
| liaison-tooling-2025a #31 | ISSUE-22 |
| liaison-tooling-2025a #34 | ISSUE-5, ISSUE-11 |
| liaison-tooling-2025a #35 | ISSUE-17 |
| liaison-tooling-2025a #36 | ISSUE-33 |
| liaison-tooling-2025a #37 | ISSUE-15 |
| liaison-tooling-2025a #38 | ISSUE-26 |
| liaison-tooling-2025a #41 | ISSUE-7, ISSUE-9 |
| liaison-tooling-2025a #42 | ISSUE-27 |
| liaison-tooling-2025a #43 | ISSUE-34 |
| liaison-tooling-2025a #45 | ISSUE-17 |
| liaison-tooling-2025a README | ISSUE-5, ISSUE-6, ISSUE-7, ISSUE-8, ISSUE-10, ISSUE-11, ISSUE-30 |
| RFC 4052bis §2 (formal relationships) | ISSUE-1 |
| RFC 4052bis §3 (liaison manager responsibilities) | ISSUE-1 |
| RFC 4053bis §2.1 (contact model) | ISSUE-20, ISSUE-25, ISSUE-26, ISSUE-33 |
| RFC 4053bis §2.2 (purpose values and deadlines) | ISSUE-23 |
| RFC 4053bis §2.3 (attachment formats) | ISSUE-37 |
| RFC 4053bis §3 (value of the public record) | ISSUE-17, ISSUE-39 |
| RFC 4053bis §4.1 (approval authority) | ISSUE-4, ISSUE-9, ISSUE-13 |
| RFC 4053bis §3.1 (follow-up responsibility, assignee) | ISSUE-24 |
| Current implementation: prior-approval bypass | ISSUE-4 |
| Current implementation: incoming auto-post | ISSUE-8 |
| Current implementation: From-Contact role restrictions | ISSUE-21 |
| Current implementation: reply allowed in any state | ISSUE-33 |
| Current implementation: action-taken authority | ISSUE-16 |
| Current implementation: `technical_contacts` | ISSUE-20 |
| Current implementation: `action_holder_contacts` | ISSUE-24 |
| Current implementation: "Authorized Individual" entry role | ISSUE-3 |
| Current implementation: attachment removal disabled in UI | ISSUE-34 |
| Current implementation: comment view restricted to Secretariat | ISSUE-35 |
| Current implementation: SDOs modelled as `Group`, managers as `Role` | ISSUE-1, ISSUE-2, ISSUE-39 |
| ietf/datatracker #5340 | ISSUE-38 |
| ietf/datatracker #6910 | ISSUE-37 |
| ietf/datatracker #7087 | ISSUE-37 |
| ietf/datatracker #8271 | ISSUE-28 |
| ietf/datatracker #9387 | ISSUE-33 |
| ietf/datatracker #9438 | ISSUE-16 |
| ietf/datatracker #9457 | ISSUE-9 |
| ietf/datatracker #10129 | ISSUE-36 |

