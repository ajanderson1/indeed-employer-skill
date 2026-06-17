# Write procedures — mutations on the live Indeed account

**Read this whole file before firing any mutation.**

A "write" is any GraphQL operation whose body opens with `mutation`. These take
**real, hard-to-reverse actions on a live Indeed employer account** — changing an
applicant's status, rejecting them, sending a message a real person receives. There
is no sandbox and no undo. The MCP server **audits every mutation loudly** server-side;
that audit log is the only write-safety guardrail. Treat writes as a big deal.

## Non-negotiable rules

1. **Never fire a mutation without explicit, specific user confirmation** naming the
   exact action and the exact target (which candidate, which job, which status, what
   message text). "Yes do it" to a vague proposal is not enough — restate specifics.
2. **Default to read.** If the user's goal can be met by reading, do that. Drafting a
   reply is a read/compose step; *sending* it is the write — keep them separate.
3. **Confirm the op name before trusting it.** Only `UpdateCandidateStatus` has a
   verified shape (below). Reject/shortlist/reply/note op names are NOT confirmed —
   verify via the audit log on first live use, never guess-and-fire blindly.
4. **One at a time.** No batch/loop mutations. No "reject everyone who…" sweeps.
5. **Stop and escalate** on any ambiguity, any error, or low confidence. This project
   runs at conditional autonomy; writes are exactly the kind of hard-to-reverse blast
   radius that requires a human in the loop.

## Confirm-first checklist (every write)

- [ ] `indeed_session_status` shows the session alive.
- [ ] The user named the exact target and action in this turn.
- [ ] You restated it back and got an explicit yes (e.g. "Move candidate <id> on job
      <id> to REJECTED — confirm?").
- [ ] For a message: the full text is shown to the user and approved verbatim.
- [ ] You ran the read first to confirm the current state (e.g. the candidate's
      present milestone) so the change is intentional, not a re-fire.
- [ ] After firing: report the result plainly and note it will appear in the audit log.

## The one confirmed mutation: `UpdateCandidateStatus`

Moves an applicant to a hiring milestone (status). Captured shape (sanitized):

```
mutation UpdateCandidateStatus(statusInput) {
  statusInput.move.milestoneId                                  # target status
  statusInput.move.candidateSubmissionEmployerJobIdPairs[] {
    candidateSubmissionId                                        # the applicant
    jobId                                                        # the job
  }
}
```

Reject is very likely a `UpdateCandidateStatus` move to a REJECTED milestone (or a
sentiment op) — confirm which by inspecting the audit log after a deliberate first use,
NOT by guessing the milestone string.

### Guarded recipe (only after the checklist passes)

1. `indeed_session_status` → alive.
2. Read current state: `indeed_candidate_detail(submission_id)` → confirm present
   milestone and that this is the right person.
3. Restate to the user: target candidate id, job id, and the destination milestone.
   Get an explicit yes **in this turn**.
4. Resolve the destination `milestoneId` (e.g. from `indeed_candidate_counts` /
   `indeed_candidate_filter_options`, which enumerate milestone ids). Do not invent it.
5. Fire via `indeed_graphql` with `operation_name="UpdateCandidateStatus"`, the
   `mutation` body above, and variables:
   `{"statusInput": {"move": {"milestoneId": "<resolved>",
     "candidateSubmissionEmployerJobIdPairs": [{"candidateSubmissionId": "<id>",
     "jobId": "<id>"}]}}}`.
   (If/when the server catalogues it, prefer `indeed_run_operation` instead.)
6. Report the outcome and confirm it landed in the audit log.

## Writes located but NOT confirmed (document only)

Mapped in the portal UI but never fired; op names are TBC via the audit log on first
deliberate use. Do not invent these op names.

| Action | Where it lives in the portal | Likely mechanism |
|---|---|---|
| Reject / decline | ✕ on a candidate row + on detail | `UpdateCandidateStatus` → REJECTED, or a sentiment op |
| Shortlist / interested | ✓ on a candidate row | sentiment mutation (`sentiments:[…]`) or status move |
| Undecided | ? on a candidate row | sentiment mutation |
| Reply / send message | compose box + **Send** in a thread | message-send mutation (name TBC) |
| Save private note | "Write private note" + **Save note** on detail | note mutation (name TBC) |
| Set up interview | **Set up interview** on detail | interview-scheduling mutation (name TBC) |

To confirm one: with the user's explicit go-ahead for a single real action, perform it
once, then read the server audit log to learn the exact op name and variable shape;
record that shape (sanitized) so it can be catalogued. This is the only safe way to
turn a located control into a documented, reusable write.

## Telemetry mutations (ignore — they auto-fire on read)

These fire on page load / list render and are harmless presence/seen-tracking, not
meaningful writes: `ReceiveOnlineStatusSignal`, `registerOnlineStatusListeners`,
`LogRcpCandidateSeen`. The MCP read tools are direct GraphQL reads and do not need them.
