---
description: "Mark a job application as submitted in the candidate's found-jobs.json ledger"
arguments:
  - name: args
    description: |
      **Required:**
      --candidate <slug>   Candidate folder name (e.g., dom-eloe)
      --job <id>           Job ID or partial match (e.g., cresta, rocket-money)

      **Optional:**
      --status <status>    Override status (default: submitted). Options: new, submitted, interviewing, offered, rejected, withdrawn
      --note <text>        Add a note (e.g., "Applied via Greenhouse, attached cover letter")

      **Examples:**
      /mark-submitted --candidate dom-eloe --job cresta
      /mark-submitted --candidate dom-eloe --job rocket-money --note "Applied with cover letter"
      /mark-submitted --candidate dom-eloe --job lithic --status interviewing --note "Phone screen scheduled 3/10"
    required: true
tools: [Read, Write, Glob, Bash]
---

<!--
TABLE OF CONTENTS
=================
1. Parse Arguments .............. line ~27
2. Find the Candidate Ledger .... line ~42
3. Match the Job ................ line ~49
4. Update the Job Entry ......... line ~62
5. Write Back ................... line ~83
6. Confirm ...................... line ~87
-->

<!--
SUMMARY: Updates the application_status field for a job in the candidate's found-jobs.json ledger.
READS: found-jobs.json
PRODUCES: updated found-jobs.json with new status, status_history entry, and optional note
-->

# Mark Application Status

You update the `application_status` field for a job in the candidate's `found-jobs.json` ledger.

## 1. PARSE ARGUMENTS

Extract from `$ARGUMENTS`:

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--candidate` | YES | — | Candidate slug (folder name under `~/job-search-candidates/`) |
| `--job` | YES | — | Job ID or partial match (company name or slug fragment) |
| `--status` | NO | `submitted` | One of: `new`, `submitted`, `interviewing`, `offered`, `rejected`, `withdrawn` |
| `--note` | NO | null | Freeform note to attach |

**Validation:**
- If `--candidate` is missing: ERROR `"--candidate is required."`
- If `--job` is missing: ERROR `"--job is required."`
- If `--status` is not one of the valid values: ERROR `"Invalid status. Must be one of: new, submitted, interviewing, offered, rejected, withdrawn"`

## 2. FIND THE CANDIDATE LEDGER

Read `~/job-search-candidates/{candidate}/found-jobs.json`.

If the file doesn't exist: ERROR `"No found-jobs.json for candidate '{candidate}'. Run /find-me-a-job first."`

## 3. MATCH THE JOB

Search the `jobs[]` array for a match against `--job`:

1. **Exact ID match** — `job.id === --job`
2. **Partial ID match** — `job.id.includes(--job)`
3. **Company name match** (case-insensitive) — `job.company.toLowerCase().includes(--job.toLowerCase())`

If **zero matches**: ERROR `"No job matching '{--job}' found in ledger."`

If **multiple matches**: List all matches with their ID, company, and role, then ask the user to be more specific.

If **one match**: Proceed.

## 4. UPDATE THE JOB ENTRY

On the matched job entry, get the current time by running `date -u +%Y-%m-%dT%H:%M:%SZ` via Bash, then:

- Set `application_status` to the `--status` value (default: `submitted`)
- **Append to `status_history`** — add an entry to the array:
  ```json
  { "status": "{new_status}", "at": "{timestamp}" }
  ```
  If `--note` is provided, include it in the history entry:
  ```json
  { "status": "{new_status}", "at": "{timestamp}", "note": "{note}" }
  ```
- **Legacy field `submitted_at`** — if status is `submitted` and `submitted_at` is currently null, also set `submitted_at` to the timestamp (for backwards compatibility)
- If `--note` is provided: set `notes` to the note value. If `notes` already has a value, append the new note on a new line with a timestamp prefix: `[YYYY-MM-DD] new note`

**If the job doesn't have tracking fields yet** (from an older run before tracking was added):
- Add `application_status`, `submitted_at`, `notes`, and `status_history` fields
- Initialize `status_history` as an array. If `submitted_at` already has a value (was set before status_history existed), seed the array with that original submission entry first, then append the new status change.

## 5. WRITE BACK

Write the updated JSON back to `found-jobs.json`. Preserve all other data exactly as-is.

## 6. CONFIRM

Display:

```
Updated: {company} — {role}
Status:  {old_status} → {new_status}
Score:   {score}/100
URL:     {url}
Note:    {note or "—"}

History:
  {status_history entries, one per line, formatted as: YYYY-MM-DD  status  (note if present)}
```
