# RUNBOOK — [Project Name]

**Spec:** [SDD file path]
**Design reference:** [design-system.md or "none"]
**Target:** OutSystems Developer Cloud (ODC)
**Build method:** ODC Mentor, one wave at a time

> Resolve tenant and app at the start of every session with `auth_status` and
> `app_list`. Never carry a key from a previous session without re-resolving.

| | |
|---|---|
| Tenant | _resolve with `auth_status`_ |
| App name | [app name] |
| App key | _resolve with `app_list`_ |
| Baseline | _measure before the first Mentor turn_ |
| Planning location | Local ← *or* Repo: `owner/repo` (see "Issue tracker" below) |
| Planning mode | Sequential ← *or* Parallel (see corresponding section below) |

---

## Current wave  *(local mode — delete this whole section in repo mode; use "Issue tracker" below instead. Sequential-only within local mode; use "Tracks" instead if also parallel)*

**→ W___ — ___**  *(update this line each time you start a new wave)*

This is the resumption pointer. When returning to an interrupted session:
1. Read this line to know where to pick up.
2. Check `execution-log.md` for the last completed entry.
3. Re-resolve tenant and app key with `auth_status` + `app_list` before firing anything.

---

## Tracks  *(local + parallel mode only — delete this whole section in sequential mode, or in repo mode; repo mode tracks live as issue labels instead, see "Issue tracker")*

| Track | Owner | Scope | Depends on |
|---|---|---|---|
| Foundation | — | entities, theme, shell — closes the data model | — |
| A | [dev name] | [module / screen group] | Foundation |
| B | [dev name] | [module / screen group] | Foundation |

**Rule:** no two tracks create or alter the same entity, action, or screen.
A wave that needs something another track owns names it explicitly under
that wave's spec "Consumes (must not alter)" section, including which
track owns it (see `spec-wave.md`'s Track field).

### Current wave per track

This replaces the single resumption pointer sequential mode uses above —
each track has its own next wave, but **only one wave fires against Mentor
at a time, across every track.** See the Claim protocol below.

| Track | → Current wave | Claimed by | Status |
|---|---|---|---|
| Foundation | W___ — ___ | — | — |
| A | A___ — ___ | — | — |
| B | B___ — ___ | — | — |

---

## Claim protocol  *(local + parallel mode only — delete this whole section in sequential mode, or in repo mode; repo mode claims a wave via its issue assignee instead, see "Issue tracker")*

The app, its live revision, and the Mentor session chain are **shared
across every track** — there is one app, one live revision, and one
`mentor_session_id`/`mentor_session_token` chain, no matter how many
tracks are being planned or tested in parallel. Only one Mentor run can be
in flight at all (`run_already_in_flight` is the platform enforcing this),
and the session/token to resume from is whichever wave *most recently
reached a terminal state* — not necessarily this track's own last wave.

Before firing **any** wave, in **any** track:

1. `git pull` — get every track's latest `RUNBOOK.md` and
   `execution-log.md`, not just this track's own history.
2. Re-resolve `app_list` / `env_app` and confirm the live revision matches
   what `execution-log.md`'s most recent entry (from any track) recorded.
   A mismatch means something published since your last pull — stop and
   re-read that entry before doing anything else.
3. Resume the Mentor session from **that** entry's `mentor_session_id` /
   `mentor_session_token` — the most recent one in the whole log, never
   assumed from this track's own last wave.
4. Claim the wave: fill in "Claimed by" for this track's row above, commit,
   push — **before firing**. A push rejected here means another track
   claimed first; pull and recheck steps 2–3, then either wait or move to
   a different ready wave in your own track.
5. Run the normal six-step wave cycle (fire → poll → gate → publish →
   compare → test) for this one wave.
6. Append the `execution-log.md` entry — the Session line (final
   `mentor_session_id`/`mentor_session_token`) is **mandatory**, not
   optional, in parallel mode. Clear "Claimed by" back to `—`, commit, push
   immediately. The next track's fire depends on seeing this pushed, not on
   hearing about it out of band.

**This makes git the lock, not a side effect of it.** The commit/push
around firing is the actual mechanism serializing the team onto one shared
Mentor session chain, in whatever order tracks are actually ready — not
the fixed order a sequential plan would impose.

**Never fire with a session or token older than the latest
`execution-log.md` entry, from any track.** A wave built against a stale
session can diverge from what's actually live, and there is no merge step
to reconcile that afterward — the fix is always "re-fire from the real
latest state," never "patch the difference in later."

---

## Issue tracker  *(repo mode only — delete this whole section in local mode; replaces "Current wave," "Tracks," "Claim protocol," and "Wave plan" above)*

**Repo:** `owner/repo`
**Labels:** `outsystems-plan` (every wave issue) · `track:<name>` (parallel mode only — one per track, e.g. `track:foundation`, `track:a`) · `status:deferred` (deferred waves — a plan-time scheduling flag, unrelated to the Status field below)
**Milestone:** `[Project Name]` — groups every wave issue under this plan
**Project board:** `[Project Name]` — number `<project-number>` / id `<project-id>`
**Status field:** id `<status-field-id>` — options: Todo `<option-id>` · In Progress `<option-id>` · In Review `<option-id>` · Done `<option-id>`

One-time setup (idempotent — safe to re-run):
```bash
gh auth status
gh auth status | grep -q "'project'" || gh auth refresh -s project
gh repo view owner/repo
gh label create outsystems-plan --repo owner/repo --color BFD4F2 --description "Wave from outsystems-plan skill" || true
gh api repos/owner/repo/milestones -f title="[Project Name]" || true
gh project list --owner owner --format json --jq '.projects[] | select(.title=="[Project Name]") | .number'
gh project create --owner owner --title "[Project Name]" --format json --jq '.number'   # only if the above printed nothing
gh project field-list <project-number> --owner owner --format json --jq '.fields[] | select(.name=="Status")'
```
A newly created board's Status field ships with Todo/In Progress/Done only —
add an **In Review** option once via the Project UI if the team wants that
stage, then re-run the `field-list` command and update the ids above.

**Find the next wave to fire** (Foundation before any track; lowest unclaimed wave ID within a track):
```bash
gh issue list --repo owner/repo --label outsystems-plan --state open
```

**Resume the Mentor session** (most recently *closed* issue, by `closedAt` —
not by wave number, since another track may have closed a later wave more
recently):
```bash
gh issue list --repo owner/repo --label outsystems-plan --state closed \
  --json number,closedAt --jq 'sort_by(.closedAt) | last | .number'
gh issue view <that number> --repo owner/repo --json comments \
  --jq '.comments[-1].body'
```
Parse the `Session:` line out of that comment.

**Claim a wave** *(parallel mode only — skip entirely in sequential mode)*:
```bash
gh issue view <number> --repo owner/repo --json assignees   # must be empty
gh issue edit <number> --repo owner/repo --add-assignee @me
```
If already assigned, someone else claimed it first — pick a different ready wave.

**Move a wave's status** ("move/mark/set it to In Progress/In Review/…" means
this — a Project field edit, never a label):
```bash
# Add the issue to the board once, right after creating it:
gh project item-add <project-number> --owner owner --url <issue-url>
# Resolve its item id (once; reuse afterward):
gh project item-list <project-number> --owner owner --format json \
  --jq --arg n <issue-number> '.items[] | select(.content.number==($n|tonumber)) | .id'
# Set Status to the matching option id from the table above:
gh project item-edit --project-id <project-id> --id <item-id> \
  --field-id <status-field-id> --single-select-option-id <option-id>
```
If the requested status has no matching option, stop and say so — add the
option via the Project UI first, don't substitute a label or the closest
existing option.

**Close a wave** (replaces an `execution-log.md` entry — same format, as an
issue comment, Session line mandatory):
```bash
gh issue comment <number> --repo owner/repo --body-file <comment file>
gh issue close <number> --repo owner/repo
```
Closing also sets the Project item to its "Done"-equivalent status
automatically on most boards — verify it landed there rather than assuming.

---

## Wave plan  *(local mode only — delete this whole section in repo mode; use "Issue tracker" above instead)*

Sequential mode:

| Wave | Name | Proves | Scope | Status |
|---|---|---|---|---|
| W1 | Foundation | Shell renders, seed data exists | entities, theme, 1 screen | — |
| W2 | … | … | … | — |
| — | — | **committed scope ends here** | — | — |
| W… | Admin screens | CRUD for reference data | +2 screens | DEFERRED |
| W… | Reporting | Dashboard and drilldown | +2 screens | DEFERRED |

Parallel mode — add a `Track` column, grouped by track:

| Track | Wave | Name | Proves | Scope | Status |
|---|---|---|---|---|---|
| Foundation | W1 | Foundation | Shell renders, seed data exists | entities, theme, 1 screen | — |
| Foundation | W2 | … | … | … | — |
| A | A1 | … | … | … | — |
| A | A2 | … | … | … | — |
| B | B1 | … | … | … | — |
| B | B2 | … | … | … | DEFERRED |

---

## Per-wave procedure

For each wave, in order. **Local + parallel mode:** run the Claim protocol
section above in place of step 0. **Repo mode:** run the Issue tracker
section above in place of step 0 (and in place of step 5's `execution-log.md`
append — close the issue instead).

**0. Update resumption pointer and record wave start**  *(local + sequential mode)*

Update the `## Current wave` line at the top of this file to the wave you
are about to start. This is the first thing a new session reads to resume.

```
**→ W<N> — <name>**
```

Then record the start time:

```bash
date "+%H:%M:%S"
```

Store this as the wave's `started` time. Never estimate it later.

**1. Fire Mentor**

For the first wave (new app):
```
app_create(name: "[app name]")  →  app_key
mentor_start(app_key, prompt: <wave spec + SDD + design-system>)
```

For subsequent waves (resume session):
```
mentor_start(
  mentor_session_id:    <from previous wave>,
  mentor_session_token: <from previous wave's terminal result>,
  fresh_context: true,
  prompt: <wave spec + SDD + design-system>
)
```

Always prepend these guardrails to the prompt:
- No hex literals — every color is a theme variable.
- **Declaring a CSS variable is not the same as applying it.** The theme
  stylesheet must also set `background-color`, `color`, and `border-color`
  on `.form-control`, `.dropdown-display`, `input`, `select`, `textarea`,
  and the Upload widget's control — with enough CSS specificity to override
  browser UA defaults. Without this, every form input renders white on a
  dark theme even when the token is correctly defined.
- **Forms use a multi-column grid. Never one field per line.** Fields that
  belong together share a row. Use `Columns2` / `Columns3` /
  `ColumnsSmallLeft` / `ColumnsSmallRight` blocks. The default vertical
  stack from the Form widget is not acceptable.
- **Action flows must be readable top-to-bottom without zooming.** Do not
  stack assignments, conditions, or calls on the same coordinates. Space
  elements so each step is individually selectable and its label is fully
  visible. Overlapping nodes are a defect.
- Use verified OutSystems UI block names only — no bare HTML.
- ODC terminology only (no "Service Studio", no "eSpace").
- Section "Out of scope" is absolute.
- Never call `eSpace.AddDependency`.
- Add every `data-test` attribute listed in the spec, spelled exactly.

**2. Poll**

Poll every **60 seconds minimum**, regardless of what `pollAfterMs` suggests.
Shorter intervals generate unnecessary calls without producing new information.
Stop on `succeeded`, `failed`, or `cancelled`.
A run still active past ~30 minutes is stuck — cancel and split at the wave's designated split point.

**Never delegate the fire-and-poll loop to a subagent.** Subagents have no real clock and will fabricate durations. Fire and poll in the orchestrating session.

**3. Static gate**

Before publishing, verify with `context_entities` / `context_actions` / `context_screens`:
- Entity count matches the wave's expected total
- Action count matches
- Screen count matches
- Zero hex literals in any screen or block
- No role created that the wave spec did not authorize

Check row `timestamp` — rows older than the Mentor turn are stale and prove nothing.

**4. Publish**

Publishing is a human decision. Prepare the call, report it, wait for confirmation.

After publish, verify with `env_app` that the new revision is live.

**5. Record wave end and ask about tests**

```bash
date "+%H:%M:%S"
```

Store this as the wave's `finished` time. Then ask:

Report: "Wave N is published. Do you want to run the E2E tests now?"

If yes: `npx playwright test tests/wN.spec.ts`
Run the full suite (all previous waves too): `npx playwright test`
A red test means the wave is not done. Do not proceed to the next wave.

If no: record the decision in the timing log and proceed.

---

## Gate: what "done" means for a wave

A wave is done when:
1. Static gate passed
2. Published (new revision confirmed)
3. E2E tests green (or explicitly deferred by user decision)
4. User has clicked the feature in the running app

**`change_applied: true` is necessary but not sufficient.** Read the model back.
A rename can report success with a fresh revision and still have the old name deployed. Check the timestamp on returned rows.

---

## Failure playbook

| Symptom | Action |
|---|---|
| `change_applied: false` on a `succeeded` run | Re-fire once with `fresh_context: true`. If it repeats, escalate to a stronger model. |
| `change_applied: true` but `context_*` shows the old state | Check the row timestamp. If newer than the turn, it silently did not land — re-fire and ask Mentor to verify per item. |
| Run stuck past ~30 min in `applyModelApiCode` | Cancel. Split the wave at the designated split point in its spec. Fire the halves separately. |
| `publish_status` 404 | The record expired. Check `app_revisions` — do NOT re-publish to find out. |
| Count is off by one | Something extra was created. Find it with `context_*` and remove it. Extra artifacts compound across waves. |
| A named block does not exist | The spec is wrong, not Mentor. Verify the real name with `context_search`, fix the spec, re-fire. |
| Feature built but broken when clicked | This is what the whole plan exists to catch. Diagnose with `app_logs` / `db_query`, then fire a fix turn against the same wave. Do not proceed to the next wave. |
| `run_already_in_flight` | Wait or cancel. Never fire a parallel run. |
| (Parallel mode) `git push` rejected on the claim commit | Another track claimed first. `git pull`, re-check the claim protocol's steps 2–3 (live revision, latest session/token), then either wait for that track to finish or move to a different ready wave in your own track. |
| (Parallel mode) Live revision doesn't match the latest `execution-log.md` entry | Something published since your last pull and the log is behind, or you're reading a stale local file. `git pull` again; if it still doesn't match, stop and reconcile before firing — do not guess which session/token is actually current. |
| (Repo + parallel mode) An issue is assigned but its wave was never actually fired | A claim that got abandoned (dev switched tasks, session died mid-fire). Check with the assignee before unassigning (`gh issue edit <n> --remove-assignee <user>`) and re-claiming — don't assume a stale claim is free just because no comment followed it. |
| (Repo mode) `gh issue create`/`comment`/`close` fails with a permissions error | `gh auth status` has drifted, or the token lacks `repo` scope. Re-authenticate (`gh auth login`) before continuing — do not fall back to writing a local file as a workaround, the plan's source of truth would then be split across two places. |
| (Repo mode) `gh project` commands fail with a permissions/scope error | The token lacks the `project` scope (separate from `repo`) — run `gh auth refresh -s project`, then retry. |

**Escalate to a stronger model when:** a wave fails twice; a gate result is ambiguous; a fix requires amending a spec; an invariant rule is in question.

---

## Execution log  *(local mode only — repo mode uses the same format as the closing comment on each wave's issue instead; see "Close a wave" in the Issue tracker section)*

All execution details live in `execution-log.md` (created alongside this file).

After each wave: append one entry. Format:

```
## W<N> — <name>  |  <started> → <finished>
- Turn <n>: <runId short>, <HH:MM>→<HH:MM> (<Xm>), retries=<N> → applied
- [Deviation: <what> → <how resolved>]
- [Fix turn: <runId short>, <Xm>, retries=<N> → <what was fixed>]
- Publish: rev <N>
- Gate: PASS
- Tests: <N>/<N> pass
- Status: DONE
- [Parallel mode only, mandatory] Session: <mentor_session_id> / <mentor_session_token>
```

Include deviation/fix lines only when something actually happened.
Every timestamp from `date "+%H:%M:%S"` — never estimated.
Write the entry once per wave, when the wave closes.
**Parallel mode:** the Session line is not optional — it's what the next
wave, in any track, resumes from (see the Claim protocol above). Commit
and push this file as part of closing the wave.

---

## Never

- Auto-publish. Publishing is a human decision every time.
- Report a wave complete on `change_applied` or a clean publish alone. Read the model back. Click the feature.
- Blame Context Service lag without checking the row timestamp first.
- Fire the next wave on a failed gate or red tests.
- Loosen a test to make a wave pass — fix the app or amend the spec and the test together.
- Poll faster than every 60 seconds — each poll consumes a model turn.
- Call `eSpace.AddDependency` — known broken. Surface needed references for manual wiring.
- Create placeholder screens for a future wave's links.
- Put a hex literal in a screen or block.
- Re-publish on a 404 `publish_status` — check `app_revisions` first.
- Delegate fire-and-poll to a subagent. Subagents fabricate durations.
- Trust a zero-artifact count on a seconds-old app — the Context Service indexes asynchronously.
- **Parallel mode:** fire with a session/token older than the latest `execution-log.md` entry from *any* track — resume from the actual latest, never from this track's own last wave.
- **Parallel mode:** skip the claim commit/push before firing — it's the only thing serializing the team onto one Mentor session chain.
- **Parallel mode:** let two tracks' specs create or alter the same entity, action, or screen — fix the scoping before either fires, not after.
- **Repo mode:** close a wave's issue without the Session line in the closing comment — it's the only durable record of what the next wave resumes from.
- **Repo mode:** maintain a local `execution-log.md`/wave-table alongside the issue tracker "just in case" — pick one source of truth; a second one left around only drifts out of sync and gets trusted by mistake.
- **Repo mode:** treat "move it to In Progress/In Review/…" as a label edit — it's the Project item's Status field; see "Move a wave's status" in the Issue tracker section.

---

## Pre-production checklist

Work through this before any promotion beyond Development:

- [ ] `DemoModeEnabled` (if used) defaults to `False` — stubs must fail loudly, not return synthetic data
- [ ] Every row where `IsSyntheticData = True` has been reviewed and removed or marked
- [ ] All external boundaries (AI, document extraction) have real implementations, not stubs
- [ ] No screen is anonymous/public if the app requires authentication
- [ ] Theme contrast ratios verified (secondary text on surface background ≥ 4.5:1)
- [ ] No `data-test` attributes left that exist only for testing
- [ ] `.env`, credentials, and test auth files are git-ignored

---

## Retrospective (fill after last committed wave)

Answer from the timing log, not from memory:

1. Which waves needed more than one turn, and why? (spec too large, too vague, or wrong?)
2. Which gate caught a real defect that looked correct on screen?
3. Which Playwright tests were flaky, and what made them flaky?
4. What did Mentor get wrong repeatedly? (Add each to the wave prompt guardrails.)
5. What did this RUNBOOK say that turned out to be wrong? (Fix it before the next project.)

Then: produce the next version of the plan in a new folder. Carry forward what held. Drop what did not.
