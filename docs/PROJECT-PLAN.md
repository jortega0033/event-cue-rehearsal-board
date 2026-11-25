# Event Cue Rehearsal Board

An offline browser workbench for creating and rehearsing a community event rundown. It gives the caller one visible next cue, explicit holds, and a record that can be replayed after an interruption.

## Product requirements

**Proposed users.** A volunteer caller rehearses transitions, learns which cues are blocked, and briefs a helper from a read-only board. The main problem is losing the next action when the plan changes or a rehearsal stops. All demonstrations use an original fictional library reading.

**First durable milestone.** In a clean local workspace, edit a three-cue draft, publish it and start a rehearsal pinned to that version. Mark the first cue ready, fire it and reopen the browser. Show the recorded first cue and the same pending order. Later increments prove skipped dependencies, interruption handling, version isolation, and independent replay.

| ID | User need | Observable success |
| --- | --- | --- |
| PR-01 | Revise the rundown without moving a cue in an ongoing run. | A new version shows its new order and titles. An existing run keeps its pinned snapshot. |
| PR-02 | Know the next action when prerequisites are incomplete. | The board describes readiness, clock and dependency blockers. Cues sharing a time follow authored order. |
| PR-03 | Hold the rehearsal until deliberate resumption. | A hold freezes clock and cues. Its release enters Paused and retains the reason. Restart while Running also enters Paused. |
| PR-04 | Let a helper read the run without operating it. | A second local tab shows the committed state after its next successful refresh. Older responses never replace newer ones. |
| PR-05 | Explain a skip and reconstruct the run after interruption. | The event log shows accepted transitions once. Replay yields the same dispositions, time, pending order and run state. |

**Scope.** Support one local rehearsal operator, multiple tabs and one active run per workspace. Use whole-second clock steps requested in the operator board. Hardware cues, lighting, audio, video playback, live feeds and collaboration over a network are outside scope. Provide keyboard operation, named controls, text statuses alongside colors, and an announcement when the cue or hold state changes.

**Proposed choices.** Start with fixed authored order and per-cue readiness. Branching and per-role readiness are later decisions. The workflow and examples use fictional materials.

## Technical design

### Local runtime and data

Use Python 3 standard library HTTP and SQLite with bundled HTML, CSS and JavaScript. Initialize a dedicated data directory with the fictional fixture, then serve it on 127.0.0.1 for Chrome. Restart with the same directory to recover. Refuse to overwrite an existing workspace. Allow one process per database directory using an exclusive Linux file lock held for the service lifetime. Preflight Python, SQLite and Chrome on the target host. Bound JSON requests, validate same-origin mutations and render cue labels and reasons as text. The local pages share one trust boundary.

Keep drafts separate from immutable published snapshots. Store an editable draft revision for each rundown. A snapshot carries a version number and a sequence of cues with stable ID, title, ordinal, earliest elapsed second, required flag and prerequisite IDs. Publish completed snapshots in a transaction and clone them for later editing. Reject stale draft edits and offer a reload. A new version never changes an existing run.

A publishable version has at most 80 unique cues, nonempty titles up to 80 characters and whole seconds from 0 through 7,200 in nondecreasing authored order. Allow at most eight distinct prerequisite IDs per cue, each earlier in this snapshot. Required cues can depend only on required cues, so a legitimate optional skip cannot strand a required cue. A failed publish keeps the editable draft and row-level errors.

Each run pins one published version and begins at sequence zero, Paused at zero seconds, with all cues Pending and unready. Persist the run and materialized cue state, append-only events, and accepted command IDs. Events contain consecutive sequence, kind, cue when applicable, logical time and transition data. Reasons are at most 120 characters. Fired and Skipped are terminal dispositions for a cue in its run. Rehearse again in a fresh run.

### Cue progression

The frontier is the earliest Pending cue by authored ordinal. Advancing time can make it due. Fire requires Running, frontier identity, due time, readiness, and all direct prerequisites Fired. A Skipped prerequisite does not satisfy this guard. Skip requires Running, an optional frontier, and a reason. It can occur before the cue is due. A later cue stays Pending until earlier cues resolve. Explain all relevant blockers in the board.

| Command | Accepted in | Effect |
| --- | --- | --- |
| Resume | Paused | Enter Running at the same time. |
| Pause | Running | Enter Paused at the same time. |
| Hold | Running | Enter Held and record a reason. |
| Release hold | Held | Enter Paused and retain the reason in history. |
| Change readiness | Paused, Running or Held | Set a different ready value for a Pending cue. This can prepare a cue during a hold. |
| Advance | Running | Add 1 through 600 seconds without exceeding 7,200. No cue resolves automatically. |
| Fire or Skip | Running | Resolve the frontier under the rules above. |
| Finish | Paused or Running with no Pending cues | Enter Completed. |
| Abandon | Paused, Running or Held | Enter Abandoned with a reason and preserve history. |

Commands carry a base sequence and an ID. Validate syntax and origin, then begin a short write transaction. If the ID exists for this run and its base and payload match, return the original accepted result without appending. A reused ID with different data is a conflict. Otherwise require the current sequence and apply the transition. Commit the updated view, event and accepted-command record together. Rejected commands leave all three unchanged. A stale operator tab keeps the input and refreshes the state before another attempt. A duplicate result must never replace a newer view.

### Restart, replay and views

Use a pure reducer to build a canonical view from the pinned snapshot plus sequenced events. On startup compare a reconstruction of the active run against its stored view. Block writes and identify the run and last replayed sequence if they disagree. A consistent Running run gets one committed recovery pause before commands become available. This preserves time, readiness and Pending order. A Paused or Held run gains no restart event. Browser reload reads the state and does not create an event. After hold release or recovery pause the operator resumes explicitly.

Serve routes to manage drafts and versions, create and read runs, submit commands and read paged events with at most 100 events per result. The operator sees named controls, elapsed time, readiness, frontier, blocker reasons and a short history. A read-only observer page shows status, time, last Fired cue, frontier and counts. It refreshes with read requests and paints only increasing sequences for the chosen run. Unavailable data keeps the last snapshot visible with a stale indicator. Updates leave the active control focused. This page has no write controls and is intended for the same local machine.

## Original verification fixtures

These are expected results for a fictional rehearsal. Application and browser tests remain planned.

### Reading Night

| ID | Ordinal | Cue | Earliest second | Required | Prerequisite |
| --- | ---: | --- | ---: | --- | --- |
| C01 | 1 | Open the room | 0 | Yes | None |
| C02 | 2 | Introduce the reading | 30 | Yes | C01 |
| C03 | 3 | Show a welcome card | 45 | No | C02 |
| C04 | 4 | Show a second card | 45 | No | C03 |
| C05 | 5 | Announce the close | 60 | Yes | C02 |

At sequence zero the run is Paused, elapsed is zero and the Pending order is C01, C02, C03, C04, C05. Apply this fixture and record sequence, state, elapsed seconds, and next cue.

| Sequence | Accepted action | Second | State | Next |
| ---: | --- | ---: | --- | --- |
| 1 | Resume | 0 | Running | C01 |
| 2 | Mark C01 ready | 0 | Running | C01 |
| 3 | Fire C01 | 0 | Running | C02 |
| 4 | Advance 20 | 20 | Running | C02 |
| 5 | Hold with reason “Check the reading order” | 20 | Held | C02 |
| 6 | Release hold | 20 | Paused | C02 |
| 7 | Resume | 20 | Running | C02 |
| 8 | Advance 10 | 30 | Running | C02 |
| 9 | Mark C02 ready | 30 | Running | C02 |
| 10 | Fire C02 with ID demo-fire-02 | 30 | Running | C03 |
| 11 | Recover on server restart | 30 | Paused | C03 |
| 12 | Resume | 30 | Running | C03 |
| 13 | Advance 15 | 45 | Running | C03 |
| 14 | Skip C03 with reason “Use the opening text only” | 45 | Running | C04 |
| 15 | Mark C04 ready | 45 | Running | C04 |
| 16 | Skip C04 with reason “Welcome card was skipped” | 45 | Running | C05 |
| 17 | Advance 15 | 60 | Running | C05 |
| 18 | Mark C05 ready | 60 | Running | C05 |
| 19 | Fire C05 | 60 | Running | None |
| 20 | Finish | 60 | Completed | None |

- After sequence 5, Advance and Skip are rejected without altering the sequence or time. After sequence 6, Fire fails until Resume. When sequence 7 is current, C02 is early. After sequence 8 it is due and still unready. Advance itself never fires it.
- At sequence 11, resending demo-fire-02 with the original base sequence 9 returns the accepted result from sequence 10. The current view stays at 11. A second restart while Paused adds nothing. Restart from a separately Held copy of the fixture preserves the hold and gains no event.
- At second 45, C03 comes before C04. Before sequence 14, Fire C04 is out of order. At sequence 15, C04 is due and ready but cannot fire because C03 was skipped. It needs its own skip and reason.
- The finish has three Fired cues C01, C02 and C05, two Skipped cues C03 and C04, zero Pending, 60 seconds and exactly 20 events. Compare with the manually authored expected result and with late replay from the pinned version.

### Version and browser scenarios

- Publish another version changing C05's title. The in-progress run retains the original title. After the original run ends, a new run uses the latest version when explicitly selected. Reject forward or missing prerequisite references. Separately reject a required cue depending on an optional cue.
- Leave an operator tab at sequence 9 while another commits sequence 10. A new stale command must fail without appending. Refresh the stale tab and verify both show sequence 10. Duplicate responses and delayed observer refreshes must not replace a newer sequence.
- In a clean Chrome profile with internet blocked, use keyboard controls to finish a rehearsal and read the hold and final counts without relying on color. Verify the observer tab shows the same finished state after refresh. Capture exact UI text or labeled screen evidence from the planned run.

## Epic · Rehearse a versioned event rundown and recover the next cue offline

Deliver a complete rehearsal with pinned versions, holds and optional skips, replay and a second local display. Total estimated implementation and verification effort is **14–23 hours**. Start EC-01, then build draft publication and the cue reducer in parallel. EC-04 wires them into durable commands and a thin normal-path rehearsal screen. EC-05 adds the full operator and observer paths. EC-06 completes release verification. The short runnable rehearsal is an initial milestone. The remaining work proves blocked cues, interruption, replay, and offline operation from a clean workspace.

### EC-01 · Initialize the local workbench and contracts

**Story.** As a caller, I want a reproducible workspace so rehearsal starts from a known state.

**Estimate.** 2–3 hours. **Dependencies.** None.

- **EC-01-A.** Open a clean setup from the original Reading Night fixture and show its exact labels and initial version in Chrome.
- **EC-01-B.** Reject a second initialization over the same directory and a second writer process. Preserve the original data.
- **EC-01-C.** Establish draft, cue, run and command contracts for both parallel tickets. Reject a malformed JSON request without changing the workspace.

### EC-02 · Edit drafts and publish immutable rundown versions

**Story.** As a caller, I want to adjust a future rundown independently of the one I am rehearsing.

**Estimate.** 3–4 hours. **Dependencies.** EC-01.

- **EC-02-A.** Clone the fixture, edit C05 and publish. The changed title appears only in the new snapshot.
- **EC-02-B.** Reject a missing or forward prerequisite, repeated IDs, and a required cue with an optional prerequisite. Preserve the invalid draft and identify its rows.
- **EC-02-C.** Edit the draft in two tabs. The second stale save fails, leaving the pinned version and first save intact.

### EC-03 · Evaluate cue readiness and holds in a deterministic reducer

**Story.** As a caller, I want each cue and clock action to have a predictable outcome.

**Estimate.** 3–5 hours. **Dependencies.** EC-01.

- **EC-03-A.** Apply the fixture actions, including a modeled recovery pause, and compare each time, state and frontier to the expected trace.
- **EC-03-B.** Reject Advance and Skip while Held, Fire while Paused, early or unready Fire, and Fire or Skip aimed at a later cue. Preserve state on every rejection.
- **EC-03-C.** Verify a skipped prerequisite blocks its dependent despite being ready and due. Require an explicit optional Skip. Check required Skip rejection and completion when all cues are terminal.

### EC-04 · Persist commands and recover after interruption

**Story.** As a caller, I want to reopen a rehearsal at the same cue without repeating an action.

**Estimate.** 2–4 hours. **Dependencies.** EC-02 and EC-03.

- **EC-04-A.** Commit or roll back the view, event and accepted ID together. Retry demo-fire-02 exactly and return sequence 10 without a second event. Reject the same ID with changed data.
- **EC-04-B.** Stop the server after sequence 10. Restart and verify one pause event at 11, the same time and the same pending order. Verify a second Paused restart and a separate Held restart add no events.
- **EC-04-C.** Complete the thin first milestone and reopen its browser tab. Compare a reconstructed view with the stored view. A deliberately mismatched synthetic view blocks further writes and identifies the run and sequence.

### EC-05 · Build usable operator and observer boards

**Story.** As a caller or helper, I want to understand the rehearsal state as it changes.

**Estimate.** 2–4 hours. **Dependencies.** EC-04.

- **EC-05-A.** With only the keyboard, use named controls and read ready, held and blocked statuses without relying on color. A refresh keeps focus on the active control.
- **EC-05-B.** Update the observer view after its next successful refresh. Show no mutation controls. Reject painting a response with a lower sequence for the same run.
- **EC-05-C.** Surface stale, duplicate, busy and prerequisite outcomes without clearing the reason input or hiding the latest committed state.

### EC-06 · Verify the complete offline rehearsal

**Story.** As a maintainer, I want evidence that another caller can repeat the rehearsal from an empty workspace.

**Estimate.** 2–3 hours. **Dependencies.** EC-05.

- **EC-06-A.** From a clean workspace, publish a version, rehearse in Chrome with a hold and explicit resume, restart at the prescribed event, and finish from the pinned version.
- **EC-06-B.** Follow the trace through sequence 20 and compare every event, the final counts and observer state with expected values. Verify replay and the materialized view agree.
- **EC-06-C.** Document the offline launch, fixture steps, actual check outputs and any host limitations. Confirm the rehearsal only needs loopback requests.
