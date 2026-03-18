---
description: "Manually add a job to the candidate's ledger from a URL, score it, and generate a full application package"
arguments:
  - name: args
    description: |
      **Required:**
      --candidate <slug>   Candidate folder name (e.g., dom-eloe)
      --url <url>          Direct URL to the job posting

      **Optional:**
      --company <name>     Company name (auto-detected if not provided)
      --role <title>       Role title (auto-detected if not provided)
      --comp <range>       Compensation range (e.g., "$150k-$200k")
      --note <text>        Note to attach (e.g., "Found via LinkedIn, referral from John")

      **Examples:**
      /add-job --candidate dom-eloe --url https://boards.greenhouse.io/company/jobs/12345
      /add-job --candidate dom-eloe --url https://linkedin.com/jobs/view/12345 --note "Recruiter reached out"
      /add-job --candidate dom-eloe --url https://company.com/careers/sr-eng --company Acme --role "Senior Engineer" --comp "$160k-$200k"
    required: true
tools: [Read, Write, Glob, Bash, WebFetch, WebSearch]
---

<!--
TABLE OF CONTENTS
=================
1. Parse Arguments .............. line ~30
2. Load Candidate Context ....... line ~47
3. Fetch Job Posting ............ line ~58
4. Dedup Check .................. line ~79
5. Score the Job ................ line ~89
6. Determine Run Folder ......... line ~107
7. Generate Job Package ......... line ~113
8. Copy Tailored Resume ......... line ~187
9. Update the Ledger ............ line ~192
10. Confirm ..................... line ~219
-->

<!--
SUMMARY: Manually adds a job from a URL to the candidate's ledger, scores it, and generates a full application package.
READS: resume-base.md, candidate-profile.json, search-config.yaml, found-jobs.json, job posting URL
PRODUCES: job-details.md, instructions.md, resume-tailored.md, updated found-jobs.json, copy in tailored-resumes/
-->

# Add Job Manually

You add a manually-found job to the candidate's ledger and generate a full application package — the same output that /find-me-a-job produces per job.

**CRITICAL:** All long-form output (job details, instructions, cover letter, resume) must be written to FILES, never to the terminal. Terminal output should only be short status messages.

## 1. PARSE ARGUMENTS

Extract from `$ARGUMENTS`:

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--candidate` | YES | — | Candidate slug |
| `--url` | YES | — | URL to the job posting |
| `--company` | NO | auto-detect | Company name |
| `--role` | NO | auto-detect | Role title |
| `--comp` | NO | auto-detect or "Not listed" | Compensation range |
| `--note` | NO | null | Freeform note |

**Validation:**
- If `--candidate` is missing: ERROR `"--candidate is required."`
- If `--url` is missing: ERROR `"--url is required."`

## 2. LOAD CANDIDATE CONTEXT

Read these files from `~/job-search-candidates/{candidate}/`:

1. **resume-base.md** — Full resume
2. **candidate-profile.json** — Structured profile with stack, experience, target titles
3. **search-config.yaml** — Preferences, avoid rules, salary minimums
4. **found-jobs.json** — Existing ledger (for dedup check)

If candidate-profile.json or found-jobs.json don't exist: ERROR `"Candidate workspace not initialized. Run /setup-candidate first."`

## 3. FETCH THE JOB POSTING

1. Use WebFetch to retrieve the job posting at `--url`
2. Extract from the page:
   - Company name (use `--company` override if provided)
   - Role title (use `--role` override if provided)
   - Full job description
   - Requirements and qualifications
   - Nice-to-haves
   - Compensation range (use `--comp` override if provided, otherwise extract or "Not listed")
   - Location / remote policy
   - Company info (size, industry, funding if available)
   - How to apply / application method
   - Posting date if visible

3. If WebFetch fails or returns insufficient data:
   - Try WebSearch for `"{company}" "{role}" job` to find alternate listings or details
   - If `--company` and `--role` were provided, work with those plus whatever you can gather
   - If you still have nothing: ERROR `"Could not fetch job details from URL. Try providing --company and --role flags."`

## 4. DEDUP CHECK

Generate a job ID: `{company-slug}_{role-slug}` (lowercase, hyphens, no special chars, truncate to 50 chars).

Check found-jobs.json for:
- Exact ID match
- Same company + similar role title
- Same URL

If duplicate found: WARN `"This job may already be in the ledger as '{existing_id}' (status: {status}). Proceeding anyway."` — don't block, just warn.

## 5. SCORE THE JOB

Score using the same rubric as /find-me-a-job:

| Dimension | Weight | Max Points | How to Score |
|-----------|--------|------------|--------------|
| Stack Match | 30% | 30 | 30=perfect match, 20=most stack matches, 10=partial, 0=different stack |
| Experience Fit | 20% | 20 | 20=exact level match, 15=close, 10=stretch, 5=significant mismatch |
| Company Stability | 15% | 15 | 15=established (500+), 12=mid (100-500), 8=small (50-100), 3=startup |
| Comp Range | 15% | 15 | 15=above target, 12=meets target, 8=slightly below, 0=way below or unlisted |
| Remote/Location | 10% | 10 | 10=confirmed remote + state eligible, 5=remote but state unknown, 0=onsite/hybrid |
| Role Type | 10% | 10 | 10=candidate's primary role type (from search-config.yaml prioritize.role_types or candidate-profile.json), 7=secondary, 5=tertiary, 3=unrelated |

Use the candidate-profile.json and search-config.yaml to inform scoring (primary_stack, min_salary, remote_only, avoid rules, etc.).

Check search-config.yaml avoid rules — if the job matches a hard-skip rule (background.avoid_keywords, background.avoid_industries), warn but still add it since the user explicitly provided this job.

## 6. DETERMINE RUN FOLDER

Find the most recent run folder in `~/job-search-candidates/{candidate}/runs/` (sort by name, take the latest). If no runs exist, create one: `runs/run-{YYYY-MM-DD-HH-MM}/`.

Create the job subfolder: `{run-folder}/{job-id}/`

## 7. GENERATE JOB PACKAGE

Write these files to `{run-folder}/{job-id}/`:

### job-details.md

```
# {Company} — {Role Title}

## Job Description
{Full description}

## Requirements
{Listed requirements}

## Nice to Haves
{If listed}

## Compensation
{Range or "Not listed"}

## Location
{Remote/hybrid/onsite, restrictions}

## Company Info
{Size, industry, funding, notable details}

## Source
- **Found on:** Manual add
- **URL:** {url}
- **Date found:** {today's date via `date -u +%Y-%m-%dT%H:%M:%SZ`}
```

### instructions.md

```
# Application Instructions: {Company} — {Role Title}

## Quick Facts
- **Company:** {name} ({size/industry})
- **Role:** {title}
- **Compensation:** {range or "Not listed"}
- **Location:** {location}
- **Match Score:** {score}/100

## How to Apply
- **Application URL:** {url}
- **Application Method:** {detected method}
- **Resume to use:** `resume-tailored.md` in this folder

## Application Tips
- {3-5 specific tips based on job requirements vs candidate strengths}

## About the Company
- {Key company details}

## Concerns / Flags
- {Any mismatches, unknowns, or yellow flags}

## Why This Matched
- {Score breakdown with notes per dimension}
```

### resume-tailored.md

Generate a tailored resume following these rules:
- **CRITICAL: Preserve the candidate's EXACT contact header** from resume-base.md (name, location, email, phone, LinkedIn). Copy verbatim — never use placeholders.
- Rewrite the Summary to mirror the job's language and priorities
- Reorder skills to lead with what the job asks for
- Adjust experience bullets to emphasize the most relevant accomplishments
- Keep roughly the same length as the original
- NEVER fabricate experience, skills, or achievements
- Do NOT add any footer, watermark, or metadata (no "Tailored for..." lines)

## 8. COPY TAILORED RESUME

Copy the tailored resume to the central folder:
`~/job-search-candidates/{candidate}/tailored-resumes/resume-{job-id}.md`

## 9. UPDATE THE LEDGER

Get the current timestamp via `date -u +%Y-%m-%dT%H:%M:%SZ`.

Add a new entry to found-jobs.json `jobs[]` array:

```json
{
  "id": "{job-id}",
  "url": "{url}",
  "company": "{company}",
  "role": "{role}",
  "comp_range": "{comp_range}",
  "location": "{location}",
  "score": {score},
  "score_breakdown": {breakdown},
  "status": "{qualified|near_miss based on score vs threshold (default 70)}",
  "source": "manual",
  "found_at": "{timestamp}",
  "run": "{run-id}",
  "application_status": "new",
  "submitted_at": null,
  "notes": "{--note value or null}"
}
```

## 10. CONFIRM

Display:

```
Added: {Company} — {Role}
Score: {score}/100
Status: {qualified|near_miss}
Package: {path to job folder}
Resume: {path to tailored resume}
```
