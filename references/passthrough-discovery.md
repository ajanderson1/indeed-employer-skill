# Passthrough discovery — finding and adding Indeed operations

How to extend the skill's reach when no typed `indeed_*` read tool covers what the
user wants. All operations hit one GraphQL endpoint the server replays:
`https://apis.indeed.com/graphql` (operationName-keyed POSTs, full query text sent).

The operation names and variable **key shapes** below were captured live (sanitized:
keys only, every value redacted). They are a map, not a guarantee — confirm the
current shape with a small run before relying on it.

## The three passthrough tools (prefer the safest)

1. **`indeed_list_operations`** — enumerate what's already catalogued server-side.
   Start here; the op you want may already be a one-call catalogued read.
2. **`indeed_run_operation(operation_name, variables)`** — run a catalogued op. The
   query body comes from the trusted server catalogue, so you supply only variables.
   **Prefer this** over raw GraphQL whenever the op is catalogued.
3. **`indeed_graphql(operation_name, query, variables)`** — raw passthrough; you
   supply the full query text. Reads AND mutations. Use only for an op the catalogue
   does not have. For a mutation, see write-procedures.md first.

The MCP resource `indeed://operations` returns the full catalogue.

## Workflow to add a new capability

1. `indeed_list_operations` — is it already there? If yes, `indeed_run_operation`.
2. If not, find the operation name + variable shape below (or capture a fresh one —
   see § Capturing a new shape).
3. Run it read-only via `indeed_graphql` with a minimal query selecting a few fields.
4. Once proven, propose it to the user as a candidate for the server catalogue (a
   new catalogued op) or a typed tool — that is the "compound the capability" step.
   The server source lives at `mcp_indeed_employer` (`src/indeed_employer_mcp/`).

## Read surface map (operation name → variable key shape)

Values shown as `<…>` are redacted placeholders; supply real values at call time.

### Jobs
- `FindEmployerJobs` — `input.{limit, filter.allOf[…createdOnIndeed/isEditable]}`
- `FindEstimatedEmployerJobCount` — `input.filter.hostedJobStatus[]`
- `GetJobsData` — `jobDataInput.jobKeys[]`

### Candidates (list + counts)
- `FindRCPMatches` — the candidate list. `input.{clientSurfaceName, limit, offset,
  context.surfaceContext[], identifiers.jobIdentifiers}`. With **no** job filter,
  `identifiers.jobIdentifiers` is `{}` (empty).
- `FindGroupedCandidateSubmissionFilterOptions` — status-tab counts + facets; four
  parallel filter inputs, each `filter.{submissionType, created.createdAfter,
  jobs.hostedJobPostStatuses[], hiringMilestones.milestoneIds[], sentiments.sentiments[]}`.
- `CandidateListIds` / `CandidateListTotalCount` —
  `input.filter.{jobs.employerJobIds[], hiringMilestones.milestoneIds[], submissionType}` (+ `first`).

### Narrow candidates by job (the documented extension)
The typed tools can't do this yet. Use the passthrough:
- Key is **`employerJobIds`** (plural) / **`employerJobId`** (singular), NOT `jobIds`.
- Value is **base64( `iri://apis.indeed.com/EmployerJob/<job-uuid>` )** — the same
  token Indeed puts in the portal URL's `selectedJobs` param. NOT the raw UUID.
- `FindRCPMatches` narrows via `input.identifiers.jobIdentifiers.employerJobId`.
- `CandidateListIds` / filter-options narrow via `…filter.jobs.employerJobIds[]`.

### Candidate detail
- `GetCandidateSubmission` — `input.legacyIds[]` (the core detail read)
- `CRP_CandidateSubmissions`, `CandidateDetailsIQP` — `input.legacyIds[]` + feature flags
- `OriginalApplicationData` — `input.submissionUuid`
- `Nex_SmartScreeningSummary_Applicant` — `{scoutApplicationId, candidateSubmissionId, …}` (screener answers)
- `FindEmployerInterviews` — `input.byCandidateSubmission.candidateSubmissionId`

#### Inline resume fetch (when typed tools fail)
The catalogued `indeed_candidate_resume` / `indeed_candidate_detail` do a paged scan
and raise if the target is not in the first 5 pages. Use `findRCPMatches` with an
explicit `candidateSubmissionUuids` filter and inline the `resume` fragment:

```graphql
query ListCandidatesWithResume($input: OrchestrationMatchesInput!) {
  findRCPMatches(input: $input) {
    matchConnection {
      matches {
        candidateSubmission {
          id
          data {
            profile { name { displayName } }
            resume {
              __typename
              ... on CandidatePdfResume {
                id
                downloadUrl
                txtDownloadUrl
              }
              ... on CandidateHtmlFile {
                id
                downloadUrl
                body
              }
              ... on CandidateTxtFile {
                id
                downloadUrl
                body
              }
              ... on CandidateUnrenderableFile {
                id
                downloadUrl
              }
            }
          }
        }
      }
    }
  }
}
```

Variables shape:
```json
{
  "input": {
    "clientSurfaceName": "candidate-list-page",
    "defaultStrategyId": "U20GF",
    "limit": 25,
    "offset": 0,
    "context": {
      "surfaceContext": [
        {"contextKey": "HOSTED_JOB_POST_STATUS", "contextPayload": "ACTIVE"},
        {"contextKey": "HOSTED_JOB_POST_STATUS", "contextPayload": "PAUSED"}
      ]
    },
    "identifiers": {
      "candidateSubmissionUuids": ["<submission-uuid>"]
    }
  }
}
```

**Caveat:** `candidateSubmissionUuids` plus other restrictive filters can trigger a
`RankerException`. Keep the input minimal — only surface context + the UUID array.

### Messages (no typed tool yet — passthrough only)
- `GetConversations` — inbox list: `{last, contexts[], scope:[{key,value}]}`
- `GetConversationAndEvents` — one thread: `{conversationId, timelineModuleInput.{atk, telVersionUpperBound}}`
- `GetUnreadConversationCount` — `contexts[]`
- `FindConversationsByCandidateKey` — `{advertiserKey, candidateKey}`
- `FetchTemplates` — saved reply templates: `{first, input.includeShared}`

### Analytics (no typed tool yet — passthrough only)
- `SpendSnapshot` — per-product spend; `{ba/cs/jc/ie}Options.{advertiserSet[],
  granularity, dateRanges[]}` (ba=Branding Ads, cs=Smart Sourcing, jc=Sponsored
  Jobs, ie=Interviews & Events).
- `SponsoredJobsRoi` — `options.{granularity, jobType, dateRanges[]}`
- `AdvertiserCurrency`, `SavedViewsList` — minimal/no vars

### Account / session
- `CurrentEmployerUserId`, `GetCurrentEmployerUser`, `GetEmployerPermissions`,
  `GetEmployerUserAccounts(userCount)`, `GetEmployerNotifications(first, input)`

## Capturing a new shape (when the map is missing one)

The live portal sits behind Cloudflare; an unauthenticated browser can't reach it.
The proven capture method (used to build this map) is an in-page fetch/XHR hook run
in a Chrome profile **already logged into** `employers.indeed.com`, recording
`operationName` + variable key paths with **all values redacted**. Walk the UI
surface you want, read the hook buffer, save **structure only** to a gitignored file.
NEVER capture or surface cookies, tokens, or response bodies. The reference capture
lives (gitignored) in the server repo: `captured_ops/skill-exploration-*.{json,md}`.

## Why keys, not just field names

R0 unit tests can't catch a wrong variable key — the captures are the only ground
truth. A correct field-leaf selection with a wrong filter key (e.g. `jobIds` instead
of `employerJobIds`) returns a 400, not a helpful error. Always match the key shape
here, and prefer `indeed_run_operation` (catalogued body) when you can.
