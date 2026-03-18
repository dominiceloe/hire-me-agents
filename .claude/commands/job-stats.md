---
description: "Show job search stats — pipeline summary, weekly activity, and unemployment reporting data"
arguments:
  - name: args
    description: |
      **Required:**
      --candidate <slug>   Candidate folder name (e.g., dom-eloe)

      **Optional:**
      --week <date>        Show stats for a specific week (YYYY-MM-DD, any day in that Sun-Sat week). Default: current week
      --all                Show all-time stats instead of weekly
      --export             Write report to a file instead of terminal

      **Examples:**
      /job-stats --candidate dom-eloe
      /job-stats --candidate dom-eloe --all
      /job-stats --candidate dom-eloe --week 2026-03-03
      /job-stats --candidate dom-eloe --all --export
    required: true
tools: [Read, Write, Glob, Bash, WebSearch, WebFetch]
---

<!--
TABLE OF CONTENTS
=================
1. Parse Arguments .............. line ~27
2. Load Data .................... line ~41
3. Calculate Stats .............. line ~55
4. Format Output ................ line ~121
5. Confirm ...................... line ~219
-->

<!--
SUMMARY: Generates a job search activity summary with pipeline stats, weekly activity, and unemployment reporting data.
READS: found-jobs.json, company info via WebSearch
PRODUCES: terminal stats report, or exported .txt file in reports/ directory
-->

# Job Search Stats

You generate a job search activity summary from the candidate's found-jobs.json ledger. This data is used for personal tracking and for unemployment reporting, so accuracy matters.

## 1. PARSE ARGUMENTS

Extract from `$ARGUMENTS`:

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--candidate` | YES | — | Candidate slug |
| `--week` | NO | current week | A date within the target week |
| `--all` | NO | false | Show all-time stats |
| `--export` | NO | false | Write to file instead of terminal |

Get today's date via `date -u +%Y-%m-%d`.

## 2. LOAD DATA

Read `~/job-search-candidates/{candidate}/found-jobs.json`.

If missing: ERROR `"No found-jobs.json for candidate '{candidate}'."`

Parse all jobs. For each job, use these fields:
- `application_status`: new, submitted, interviewing, offered, rejected, withdrawn
- `submitted_at`: ISO timestamp (null if not submitted)
- `found_at`: ISO timestamp
- `status`: qualified, near_miss
- `score`: match score
- `company`, `role`, `comp_range`, `url`

## 3. CALCULATE STATS

### Pipeline Summary

Count jobs by `application_status`:
- **New** (not yet applied)
- **Submitted** (application sent, waiting)
- **Interviewing** (in interview process)
- **Offered** (received offer)
- **Rejected** (company declined)
- **Withdrawn** (candidate withdrew)

Also calculate:
- Total jobs in ledger
- Total qualified vs near_miss
- Response rate: (rejected + interviewing + offered) / submitted — as a percentage
- Average score of submitted jobs
- Average score of all qualified jobs

### Weekly Activity (for unemployment reporting)

If `--all` is NOT set, filter to the target week (Sunday–Saturday).

Week start = most recent Sunday on or before the target date.
Week end = Saturday after week start.

For the target week, gather:
- Jobs found (by `found_at`)
- Applications submitted (by `submitted_at`)
- Status changes (any job whose status changed — approximate by checking `submitted_at` within the week)

### Application Log

Build a list of all submitted/active applications sorted by `submitted_at` descending:
- Company, Role, Date Submitted, Current Status, Score

### Unemployment Reporting Block

Build a structured block specifically for unemployment claims:
- Week ending date (Saturday)
- Number of employer contacts (applications submitted that week)
- For each contact: Company Name, Date, Method (online application), Result (pending/rejected/interview)

### Certification Form Data (auto-lookup)

When generating the CERTIFICATION FORM DATA block, for each submitted job in the target week:

1. **Company details lookup**: Use WebSearch to search `"{company name}" headquarters address` and WebFetch on the top result to extract:
   - Company website URL (just the domain, e.g. `https://example.com`)
   - Street address (or `"See company website"` if not found)
   - City, State, ZIP (leave blank if not found)
   - If multiple jobs share the same company, reuse the lookup result — don't search again.

2. **Job number extraction**: Inspect the job's `url` field. Common patterns:
   - Greenhouse: `/jobs/123456` → job number `123456`
   - Lever: `/postings/{id}` → use the id
   - Workday: job URL path often contains a numeric ID
   - If no recognizable pattern, leave Job Number blank.

3. **Next Step derivation** from `application_status`:
   - `submitted` → `"Waiting to hear back"`
   - `rejected` → `"Not selected"`
   - `interviewing` → `"Interview scheduled"`
   - `offered` → `"Received offer"`
   - `withdrawn` → `"Withdrew application"`
   - Any other → `"Waiting to hear back"`

## 4. FORMAT OUTPUT

### If --export flag is set:

Write the report to: `~/job-search-candidates/{candidate}/reports/stats-{date}.txt`

Create the `reports/` directory if it doesn't exist.

Use plain text formatting (no markdown) so it copies cleanly.

### If --export is NOT set:

Output directly to terminal using compact formatting.

### Report Format:

```
JOB SEARCH STATS — {Candidate Name}
Report Date: {today}
================================================================

PIPELINE
----------------------------------------------------------------
New (ready to apply)    {n}
Submitted (waiting)     {n}
Interviewing            {n}
Offered                 {n}
Rejected                {n}
Withdrawn               {n}
----------------------------------------------------------------
Total in ledger         {n}
Qualified               {n}
Near Misses             {n}

METRICS
----------------------------------------------------------------
Response rate           {n}% ({responses}/{submitted})
Avg score (submitted)   {n}/100
Avg score (qualified)   {n}/100
Highest score           {n}/100 — {company}
Comp range (submitted)  {lowest}–{highest}

APPLICATIONS
----------------------------------------------------------------
Company              Role                        Submitted   Status
----------------------------------------------------------------------
{company}            {role}                      {date}      {status}
{company}            {role}                      {date}      {status}
...

WEEKLY ACTIVITY — Week of {Sunday date} to {Saturday date}
----------------------------------------------------------------
Jobs found this week          {n}
Applications submitted        {n}
Responses received            {n}

UNEMPLOYMENT REPORT — Week Ending {Saturday date}
----------------------------------------------------------------
Employer contacts this week: {n}

  1. {Company Name}
     Date: {submitted_at date}
     Method: Online application
     Position: {role title}
     Result: {Pending / Not selected / Interview scheduled}

  2. {Company Name}
     ...
----------------------------------------------------------------
I certify that I made {n} employer contact(s) during this week.

CERTIFICATION FORM DATA — Week of {Sunday date} to {Saturday date}
For each employer contact, fill into the online certification form:
================================================================

--- Application 1 of {n} ---
Action Date:          {submitted_at date, MM/DD/YYYY}
Contact Name:         [leave blank]
Contact Phone:        [leave blank]
Confirmation #:       [leave blank]
Company Name:         {company}
Company Address:      {looked up or "See company website"}
City:                 {looked up or blank}
State:                {looked up or blank}
ZIP Code:             {looked up or blank}
Company Website:      {looked up from web search}
Company Email:        [leave blank]
Company Fax:          [leave blank]
Type of Work:         {role title}
Job Number:           {extracted from URL if possible, else blank}
Application Submitted: Yes
Next Step:            {derived from application_status — see §3 Certification Form Data}

--- Application 2 of {n} ---
...
```

## 5. CONFIRM

If `--export` was used:
```
Stats written to: {file path}
```

If not, the terminal output IS the confirmation — don't add extra messaging.
