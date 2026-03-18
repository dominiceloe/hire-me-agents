---
description: "Launch parallel AI agents to search for jobs, score matches, tailor resumes, and generate application packages"
arguments:
  - name: args
    description: |
      **Required:**
      --resume <path>    Path to a markdown resume file (source of truth)

      **Optional:**
      --duration <time>  How long to run (e.g., 30m, 2h, 8h). Default: 1h
      --agents <n>       Number of parallel search agents (3-5). Default: 3
      --interval <time>  Sleep between search cycles (e.g., 5m, 15m). Default: 10m
      --threshold <n>    Minimum match score 0-100. Default: 70
      --output-dir <path> Where to write output. Default: ~/job-search-candidates/
      --config <path>    Optional YAML/JSON config with search preferences

      **Examples:**
      /find-me-a-job --resume ~/resumes/dom.md
      /find-me-a-job --resume ~/resumes/dom.md --duration 2h --agents 5
      /find-me-a-job --resume ~/resumes/dom.md --config ~/configs/search.yaml --threshold 60
    required: true
tools: [Read, Edit, Write, Glob, Grep, Bash, Task, WebSearch, WebFetch, TodoWrite]
---

<!--
TABLE OF CONTENTS
=================
1. Parse Arguments .................. line ~33
2. Read and Extract Candidate Profile line ~72
3. Merge Search Config .............. line ~134
4. Setup Output Directory ........... line ~157
5. Spawn Search Agents .............. line ~213
6. Monitor and Coordinate ........... line ~574
7. Generate Final Report ............ line ~619
8. Cleanup and Final Output ......... line ~838
9. Critical Rules ................... line ~870
-->

<!--
SUMMARY: Spawns parallel search agents to find jobs, score them, and generate full application packages.
READS: resume-base.md, candidate-profile.json, search-config.yaml, found-jobs.json
PRODUCES: agent results, tailored resumes, cover letters, job-details, instructions, FINAL-REPORT.md, updated found-jobs.json
-->

# Job Search Agent System

You are the **Lead Coordinator** for an automated job search system. You do NOT search for jobs yourself — you orchestrate a team of parallel search agents, coordinate their results, manage deduplication, and produce the final deliverables.

**Your primary tools:** Task (to spawn parallel agents), Read/Write/Edit (file management), WebSearch/WebFetch (only for validation), Bash (directory ops), TodoWrite (progress tracking).

---

<!-- ═══════════════════════════════════════════════════════ -->
<!-- SECTION 1: PARSE ARGUMENTS                             -->
<!-- ═══════════════════════════════════════════════════════ -->

## 1. PARSE ARGUMENTS

Extract parameters from the user's input: `$ARGUMENTS`

Parse these flags (use defaults if not provided):

| Flag | Default | Description |
|------|---------|-------------|
| `--resume` | **REQUIRED** | Path to markdown resume file |
| `--duration` | `1h` | Total run time (e.g., 30m, 2h, 8h) |
| `--agents` | `3` | Number of parallel search agents (3-5) |
| `--interval` | `10m` | Sleep between search cycles per agent |
| `--threshold` | `70` | Minimum match score (0-100) |
| `--output-dir` | `~/job-search-candidates/` | Root output directory |
| `--config` | none | Optional YAML/JSON search preferences file |

**Validation:**
- If `--resume` is missing, ERROR immediately: `"--resume is required. Usage: /find-me-a-job --resume <path>"`
- If `--config` is provided without `--resume`, ERROR: `"--resume is required. The config file supplements the resume but cannot replace it."`
- If the resume file doesn't exist, ERROR: `"Resume file not found: <path>"`
- If the resume file is not `.md` or `.txt`: `"Please provide a markdown (.md) resume file."`
- Clamp `--agents` to 1-5 range
- Clamp `--threshold` to 0-100 range

Display parsed parameters:
```
Job Search System Starting
━━━━━━━━━━━━━━━━━━━━━━━━━━
Resume:    <path>
Duration:  <time>
Agents:    <n>
Interval:  <time>
Threshold: <n>
Output:    <path>
Config:    <path or "none">
```

---

<!-- ═══════════════════════════════════════════════════════ -->
<!-- SECTION 2: READ AND EXTRACT CANDIDATE PROFILE          -->
<!-- ═══════════════════════════════════════════════════════ -->

## 2. READ AND EXTRACT CANDIDATE PROFILE

Read the resume file at the `--resume` path. Extract a structured candidate profile.

### What to Extract

Analyze the full resume content and extract:

1. **name** — Full name (look for the largest heading, or the first line, or a "Name:" field)
2. **location** — City, State, Country (look for location in contact section, header, or summary)
3. **years_experience** — Calculate from work history dates. Count from earliest professional role to present.
4. **experience_level** — Derive from years: 0-2 = "junior", 3-5 = "mid", 5-8 = "senior", 8+ = "staff/principal"
5. **primary_stack** — Languages, frameworks, databases, tools that appear in BOTH skills section AND experience bullet points (i.e., actually used, not just listed)
6. **secondary_stack** — Technologies listed in skills but not heavily featured in experience
7. **job_titles** — All titles held (e.g., "Senior Software Developer", "Technical Support Engineer")
8. **target_titles** — Generate appropriate job search titles based on most recent title and experience level. Include variations (e.g., for "Senior Software Developer" with React/Python: ["Senior Frontend Engineer", "Senior Full Stack Engineer", "Senior Software Engineer", "Frontend Developer", "Full Stack Developer"])
9. **industries** — Industries worked in (SaaS, edtech, etc.)
10. **achievements** — Notable accomplishments (patents, scale numbers, acquisitions, open source, etc.)
11. **education** — Schools and degrees
12. **remote_preference** — Look for "Remote", "Open to Remote", location cues
13. **summary** — The candidate's own summary/objective section verbatim

### Generate Search Keywords

From the extracted profile, generate:
- **title_queries** — 5-8 job title strings to search for (e.g., "Senior Frontend Engineer", "React Developer", "Full Stack Python")
- **stack_queries** — 3-5 technology combination strings (e.g., "React TypeScript", "Python Django", "WebRTC real-time")
- **combo_queries** — 3-5 combined title+tech strings for Google search (e.g., `"senior frontend" "react" "typescript" remote`)

### Write candidate-profile.json

Generate a slug from the candidate name: lowercase, hyphens, no special chars (e.g., "Dominic Eloe" → "dom-eloe", "Jane Smith" → "jane-smith"). If name extraction fails, use the resume filename.

Write the profile to `{output-dir}/{candidate-slug}/candidate-profile.json`:

```json
{
  "name": "Full Name",
  "slug": "full-name",
  "location": { "city": "...", "state": "...", "country": "US" },
  "years_experience": 5,
  "experience_level": "senior",
  "primary_stack": ["React", "TypeScript", "Python", "Django", "PostgreSQL"],
  "secondary_stack": ["Docker", "AWS", "Redis", "Celery"],
  "job_titles": ["Senior Software Developer"],
  "target_titles": ["Senior Frontend Engineer", "Senior Full Stack Engineer", "..."],
  "industries": ["SaaS", "Community Platforms"],
  "achievements": ["Built product from scratch through acquisition", "..."],
  "education": [{"school": "...", "degree": "...", "status": "attended"}],
  "remote_preference": "remote",
  "summary": "...",
  "search_queries": {
    "title_queries": ["..."],
    "stack_queries": ["..."],
    "combo_queries": ["..."]
  },
  "extracted_at": "2026-03-02T14:30:00Z"
}
```

---

<!-- ═══════════════════════════════════════════════════════ -->
<!-- SECTION 3: MERGE SEARCH CONFIG                         -->
<!-- ═══════════════════════════════════════════════════════ -->

## 3. MERGE SEARCH CONFIG (if --config provided)

If a `--config` path was provided:

1. Read the file (support `.yaml`, `.yml`, and `.json`)
2. Parse using bash: for JSON use `cat`, for YAML read it as text and interpret the structure
3. Merge with auto-detected profile:
   - Config values **override** auto-detected values where they conflict
   - Config `avoid.industries`, `avoid.keywords`, `avoid.companies` ADD to skip rules
   - Config `prioritize.industries`, `prioritize.keywords` ADD to scoring bonuses
   - Config `preferences.remote_only`, `preferences.min_salary`, `preferences.company_size_min/max` SET hard filters
   - Config `background.avoid_keywords`, `background.avoid_industries` ADD to auto-skip rules (these are HARD skips, not score reductions)
4. Copy the config file to `{output-dir}/{candidate-slug}/search-config.yaml`

If no config: proceed with auto-detected values only. Use sensible defaults:
- Remote preferred but not required
- No salary floor (score by market rate)
- No industry restrictions
- No company blocklist

---

<!-- ═══════════════════════════════════════════════════════ -->
<!-- SECTION 4: SETUP OUTPUT DIRECTORY                      -->
<!-- ═══════════════════════════════════════════════════════ -->

## 4. SETUP OUTPUT DIRECTORY

Create the full directory structure:

```bash
mkdir -p {output-dir}/{candidate-slug}/runs/run-{YYYY-MM-DD-HH-MM}/
```

### Initialize Files

**A. Check/create cumulative found-jobs.json** at `{output-dir}/{candidate-slug}/found-jobs.json`:
- If it exists: read it (this is a subsequent run — all previous jobs are loaded for dedup)
- If it doesn't exist: create it with `{ "jobs": [], "metadata": { "created": "<timestamp>", "candidate": "<name>" } }`

**B. Create run-meta.json** at `{run-folder}/run-meta.json`:
```json
{
  "run_id": "run-2026-03-02-14-30",
  "candidate": "Full Name",
  "started_at": "2026-03-02T14:30:00Z",
  "ended_at": null,
  "duration_configured": "1h",
  "agents_configured": 3,
  "interval_configured": "10m",
  "threshold": 70,
  "sources_searched": [],
  "total_jobs_scanned": 0,
  "total_qualified": 0,
  "total_skipped": 0,
  "total_duplicates": 0,
  "previous_runs": 0
}
```

**C. Snapshot candidate-profile.json** into the run folder.

**D. Snapshot search config** (if provided) as `search-config-snapshot.yaml` in the run folder.

**E. Initialize search-log.json** in the run folder: `{ "events": [] }`

**F. Initialize summary.md** in the run folder:
```markdown
# Job Search Results — {Candidate Name}
**Run:** {run-id} | **Date:** {date} | **Threshold:** {threshold}

| # | Company | Role | Score | Comp Range | Location | Link | Status |
|---|---------|------|-------|------------|----------|------|--------|
```

Display:
```
Output: {output-dir}/{candidate-slug}/runs/{run-folder}/
Previous jobs in ledger: {count}
```

---

<!-- ═══════════════════════════════════════════════════════ -->
<!-- SECTION 5: SPAWN SEARCH AGENTS                         -->
<!-- ═══════════════════════════════════════════════════════ -->

## 5. SPAWN SEARCH AGENTS

Launch parallel search agents using the **Task** tool. Each agent gets a comprehensive prompt with everything it needs to work independently.

### Agent Assignment Strategy

Distribute sources across agents based on `--agents` count:

**3 agents (default):**
- Agent 1: HN Who's Hiring + We Work Remotely + RemoteOK
- Agent 2: Google Jobs aggregation (Greenhouse, Lever, LinkedIn via Google)
- Agent 3: Built In + Arc.dev + Direct company career pages

**4 agents:**
- Agent 1: HN Who's Hiring + We Work Remotely
- Agent 2: RemoteOK + Arc.dev
- Agent 3: Google Jobs aggregation (Greenhouse, Lever, LinkedIn via Google)
- Agent 4: Built In + Direct company career pages

**5 agents:**
- Agent 1: HN Who's Hiring
- Agent 2: We Work Remotely + RemoteOK
- Agent 3: Arc.dev + Built In
- Agent 4: Google Jobs aggregation (Greenhouse, Lever, LinkedIn via Google)
- Agent 5: Direct company career pages + Wellfound

### Agent Output Strategy (Per-Agent Files)

**CRITICAL: Agents do NOT write to shared files.** Each agent writes ONLY to its own output file to eliminate file contention.

Each agent writes its results to:
`{run-folder}/agent-{N}-results.json`

Format:
```json
{
  "agent": "agent-{N}",
  "sources": ["source1", "source2"],
  "completed_at": "ISO timestamp",
  "jobs": [
    {
      "id": "{company-slug}_{role-slug}",
      "url": "...",
      "company": "...",
      "role": "...",
      "comp_range": "...",
      "location": "...",
      "score": 82,
      "score_breakdown": { ... },
      "status": "qualified|near_miss|skipped",
      "source": "...",
      "found_by": "agent-{N}",
      "found_at": "ISO timestamp",
      "skip_reason": null,
      "state_eligible": true,
      "posting_date": null,
      "ats_platform": "greenhouse|ashby|lever|workday|icims|taleo|unknown",
      "applicant_count": null
    }
  ],
  "errors": [],
  "summary": {
    "total_scanned": 0,
    "qualified": 0,
    "near_misses": 0,
    "skipped": 0
  }
}
```

Each agent ALSO creates job subfolders for qualified jobs (these don't conflict since each agent handles different sources):
`{run-folder}/{company-slug}_{role-slug}/` containing job-details.md, instructions.md, resume-tailored.md, and cover-letter.md.

### Agent Prompt Template

Each agent is spawned with the Task tool using `subagent_type: "general-purpose"`. Run agents **in the background** using `run_in_background: true` so they execute in parallel.

**CRITICAL:** Spawn ALL agents in a SINGLE message with multiple Task tool calls so they run in parallel.

Each agent's prompt MUST include ALL of the following (copy the full content — agents cannot read files created by the lead):

```
You are Job Search Agent {N} for candidate {NAME}.

## Your Search Sources
{List of assigned sources with URLs and search strategies}

## Candidate Profile
{PASTE the full candidate-profile.json content here}

## Resume Content
{PASTE the full resume markdown content here}

## Search Config
{PASTE the merged search preferences — avoid lists, prioritize lists, salary min, etc.}

## Search Queries to Use
{PASTE the generated title_queries, stack_queries, combo_queries}

## Previously Found Jobs (for dedup)
{PASTE the list of job IDs and URLs from found-jobs.json so the agent can skip duplicates without reading shared files}

## Scoring Rubric

Score each job on these dimensions (0-100 weighted total):

| Dimension | Weight | Max Points | How to Score |
|-----------|--------|------------|--------------|
| Stack Match | 30% | 30 | 30=perfect match, 20=most stack matches, 10=partial, 0=different stack |
| Experience Fit | 20% | 20 | 20=exact level match, 15=close, 10=stretch, 5=significant mismatch |
| Company Stability | 15% | 15 | 15=established (500+), 12=mid (100-500), 8=small (50-100), 3=startup |
| Comp Range | 15% | 15 | 15=above target, 12=meets target, 8=slightly below, 0=way below or unlisted |
| Remote/Location | 10% | 10 | 10=confirmed remote + state eligible, 5=remote but state unknown, 0=onsite/hybrid |
| Role Type | 10% | 10 | 10=candidate's primary role type (from prioritize.role_types), 7=secondary, 5=tertiary, 3=unrelated |

### ATS Platform Detection

When processing each job listing, identify which ATS platform hosts it:
- **Greenhouse:** URL contains `greenhouse.io` or `boards.greenhouse.io`
- **Ashby:** URL contains `jobs.ashbyhq.com`
- **Lever:** URL contains `jobs.lever.co`
- **Workday:** URL contains `myworkdayjobs.com` or `workday.com`
- **iCIMS:** URL contains `icims.com` or `careers-*.icims.com`
- **Taleo:** URL contains `taleo.net`
- **Unknown:** All others

Record the ATS platform in the job result's `ats_platform` field. Also record `applicant_count` if visible on the listing page (LinkedIn shows this, some Greenhouse/Ashby listings show it).

## Auto-Skip Rules

Skip these WITHOUT scoring (log as "skipped" with reason):
- Roles requiring technologies not in the candidate's stack as PRIMARY requirement
- Roles requiring more years of experience than the candidate has + 2 (e.g., skip "10+ years required" if candidate has 5)
- Recruiting agency postings for unnamed companies
- Obvious duplicates of jobs already in the "Previously Found Jobs" list
{IF CONFIG HAS AVOID RULES: paste them here}
{IF CONFIG HAS BACKGROUND RULES: paste them here as hard skips}

## Output Instructions

### 1. Write Your Results File
Write ALL results (qualified, near_miss, and skipped) to: `{run-folder}/agent-{N}-results.json`
Use the JSON format specified above. Write this file ONCE at the end of your search, not incrementally.

### 2. For Each Qualifying Job (score >= {threshold})

Create a subfolder and files:

#### Job Subfolder
Create: `{run-folder}/{company-slug}_{role-slug}/`
Slug rules: lowercase, hyphens for spaces, remove special chars, truncate to 50 chars
Example: `pinterest_sr-frontend-engineer/`

#### job-details.md
```markdown
# {Company} — {Role Title}

## Job Description
{Full job description as found}

## Requirements
{Listed requirements}

## Nice to Haves
{If listed}

## Compensation
{Range if available, "Not listed" if not}

## Location
{Remote/hybrid/onsite, state restrictions if mentioned}

## Company Info
{Brief company description, size if findable, industry}

## ATS Platform
{Greenhouse / Ashby / Lever / Workday / iCIMS / Taleo / Unknown}

## Source
- **Found on:** {source name}
- **URL:** {direct link}
- **Date found:** {timestamp}
- **Posting date:** {if available}
- **Applicant count:** {if visible, otherwise "Not available"}
```

#### instructions.md
```markdown
# Application Instructions: {Company} — {Role Title}

## Quick Facts
- **Company:** {name} ({employee count if known}, {industry})
- **Role:** {title}
- **Compensation:** {range or "Not listed"}
- **Location:** {remote/hybrid/onsite, state restrictions}
- **Match Score:** {X}/100
- **Posted:** {date if available}
- **Application Deadline:** {if mentioned, otherwise "Not specified"}
- **ATS Platform:** {platform name}
- **Applicant Count:** {count if known, otherwise "Unknown"}

## How to Apply
- **Application URL:** {direct link to apply}
- **Application Method:** {Easy Apply / Greenhouse form / Email / etc.}
- **Resume to use:** `resume-tailored.md` in this folder (run `/export-resume` to convert to .docx)

## ATS Notes
{Provide ATS-specific guidance based on the detected platform:}
- **Greenhouse / Ashby (modern ATS):** These systems are recruiter-search based, not auto-filter. The tailored resume's keyword optimization helps recruiters find the candidate when searching. Focus on having every relevant technology as a standalone keyword in the skills section.
- **Workday / iCIMS / Taleo (legacy ATS):** These systems are more likely to auto-filter by keyword matching. Exact keyword matching is CRITICAL. Ensure every required technology from the listing appears verbatim in the resume skills section.
- **High applicant volume (500+):** ATS filtering is more likely at this volume. Exact keyword matching becomes more important. The tailored resume should mirror the listing's exact terminology.
- **Low applicant volume (<50):** A human is more likely to read every resume. The tailored resume should optimize for readability and narrative over keyword density.

## Application Tips
- {Key things to emphasize based on the job description}
- {Any specific questions the application asks and suggested answers}
- {Cover letter suggestions if relevant}

## About the Company
- {Brief description}
- {Funding/stage if available}
- {Notable products or clients}

## Concerns / Flags
- {Yellow flags: state eligibility uncertainty, experience stretch, posting age, etc.}
- {Config avoid-list matches if any}
{If posting is older than 30 days: "⚠️ This posting may be stale (posted {date})."}

## Why This Matched
- {Specific skills from resume that align}
- {Score breakdown: Stack {X}/30, Experience {X}/20, Stability {X}/15, Comp {X}/15, Location {X}/10, Role Type {X}/10}
```

#### resume-tailored.md
Generate a tailored version of the candidate's resume for this specific job:

**Core Tailoring Rules:**
- **CRITICAL: Preserve the candidate's EXACT contact header** from the resume (name, location, email, phone, LinkedIn). Copy the header lines verbatim — never replace them with placeholders like `[email]`, `[phone]`, `[name]`, etc. If the resume has `jane@example.com | 555-123-4567`, the tailored resume must have exactly that.
- Rewrite the Summary to mirror the job's language and priorities
- Reorder skills to lead with what the job asks for
- Adjust experience bullets to emphasize the most relevant accomplishments
- Keep roughly the same length as the original
- NEVER fabricate experience, skills, or achievements
- Do NOT add any footer, watermark, or metadata to the resume (no "Tailored for..." lines, no generation timestamps, nothing). The output must be a clean, submission-ready resume.

**ATS Keyword Optimization:**
- When the job listing uses a specific term the candidate has experience with, use THAT EXACT TERM in the tailored resume. For example: if the listing says "Agile" don't write "agile methodology"; if it says "CI/CD pipelines" don't write "continuous deployment"; if it says "REST APIs" don't write "API development."
- Ensure every technology explicitly named in the job requirements appears as a standalone keyword in the Technical Skills section — not buried in a sentence in the experience bullets only. If "PostgreSQL" is in the requirements, it must be in the skills section AND can also appear in bullets.
- If the listing says "Software Engineer" and the candidate's title was "Software Developer", use "Software Engineer" in the Summary while keeping the actual title accurate in the Experience section. This ensures the ATS matches the title the recruiter searches for.
- If the listing names specific cloud services (e.g., "S3", "Lambda", "EC2"), list those individually in the skills section rather than just the umbrella term "AWS."
- If the listing mentions a methodology (e.g., "Agile", "Scrum", "Kanban") and the candidate has that experience, ensure it appears as a keyword in the skills section under a "Process" or "Product & Process" category.

**Skills Section Formatting for ATS:**
- List technologies as comma-separated keywords, not prose sentences
- Group by category with bold labels (Frontend:, Backend:, Cloud & DevOps:, etc.)
- Every technology the candidate has used that appears in the job listing MUST be in the skills section — don't rely on it appearing only in the experience bullets
- Expand umbrella terms when the listing is specific: "AWS" becomes "AWS (S3, EC2, Lambda, ECS, CloudFront)" if the candidate has used those services and the listing names them
- The skills section is the primary ATS keyword block — treat it as the single most important section for ATS parsing

**Smart Emphasis (detect automatically from resume content):**
- If the role is frontend-only and the candidate has fullstack experience: emphasize UI/component/SPA work, de-emphasize backend
- If the role mentions specific technologies the candidate has used: ensure those appear prominently
- If the role involves real-time/video/streaming and the candidate has relevant experience: highlight it
- If the role mentions AI/ML and the candidate has relevant experience: highlight it
- If the role involves payments/billing and the candidate has Stripe experience: highlight it

**Cover Letter Generation:**

Generate a tailored cover letter for each qualifying job.

Format:
- 3-4 short paragraphs, no more than 250 words total
- No generic filler ("I am excited to apply for..." / "I believe I would be a great fit...")
- Conversational but professional tone — sounds like a real person, not a template
- Markdown format saved as `cover-letter.md` in the job subfolder

Content structure:
1. **Opening** — State the role, then immediately connect the candidate's most relevant experience to what the job asks for. Lead with specifics, not enthusiasm.
2. **Middle (1-2 paragraphs)** — Draw direct lines between the candidate's experience and the job's key requirements. Reference specific projects, technologies, or achievements from the resume that map to what the team needs. Mention side projects only if they're directly relevant.
3. **Closing** — Brief, confident. No begging.

Rules:
- Never fabricate or exaggerate — only reference real experience from the resume
- Mirror the job posting's language where natural (if they say "ship to production," use that phrase)
- If the job mentions specific tech the candidate has used, name it explicitly
- Keep it tight — every sentence should earn its place
- If the job listing emphasizes culture, values, or working style, reflect those naturally without being sycophantic

#### Output Format
The .md files (resume-tailored.md, cover-letter.md) are the primary output. Do NOT attempt PDF/DOCX conversion — use the /export-resume skill separately if needed.

## Search Strategy Per Source

### HN "Who is Hiring" (hnhiring.com)
1. WebFetch https://hnhiring.com/ and look for the most recent month's thread
2. WebFetch the thread URL
3. Search through postings for keywords from the candidate's stack_queries
4. HN posts are plaintext — extract company, role, location, comp, requirements, and link from each matching post
5. Many HN posts don't have formal application links — note "Apply via email" or "Apply on company site" in instructions.md

### We Work Remotely (weworkremotely.com)
1. WebSearch for: site:weworkremotely.com {title_query} OR {stack_query}
2. Or WebFetch category pages like https://weworkremotely.com/remote-jobs/search?term={query}
3. Extract job details from listing pages

### RemoteOK (remoteok.com)
1. First try: WebFetch https://remoteok.com/api — returns JSON of all listings
2. If rate-limited or blocked: fall back to WebSearch for `site:remoteok.com {title_query}`
3. If both fail: log error and skip this source (do NOT retry indefinitely)
4. Filter by candidate's tech keywords
5. Extract structured data from JSON response

### Arc.dev (arc.dev/remote-jobs)
1. WebSearch for: site:arc.dev {title_query} remote
2. WebFetch listing pages for details

### Built In (builtin.com)
1. WebSearch for: site:builtin.com/jobs {title_query} remote
2. WebFetch listing pages for details

### Google Jobs Aggregation
1. WebSearch: {combo_query} site:greenhouse.io OR site:lever.co
2. WebSearch: {combo_query} site:linkedin.com/jobs
3. WebSearch: "{title_query}" "{primary_tech}" "remote" job
4. WebFetch individual listing pages for full details

### Direct Company Career Pages
1. Build a target company list based on the candidate's industry and stack (look for well-known companies in their space)
2. WebSearch: "{company name}" careers "{primary_tech}"
3. WebFetch career pages directly

## Deduplication

Before processing any job, check if it already exists in the "Previously Found Jobs" list provided in your prompt:
- Match by URL (exact match)
- Match by company+role combination (fuzzy — same company and very similar title)
- If duplicate: skip processing, include in your results with status "duplicate"

## Error Handling

- If a source URL fails or is rate-limited: log the error, skip that source, continue with others
- If resume tailoring fails for a job: log the error, still create job-details.md and instructions.md
- Never crash the entire agent over a single source failure
- Do NOT install system packages (brew, apt, etc.) — log the missing dependency and continue

## What to Return

When your search cycle is complete, return a summary:
- Total jobs scanned
- Total qualified (above threshold)
- Total near-misses
- Total skipped (with breakdown by reason)
- Total duplicates
- List of qualified jobs with company, role, score
- Any errors encountered
- Any sources that were inaccessible
```

---

<!-- ═══════════════════════════════════════════════════════ -->
<!-- SECTION 6: MONITOR AND COORDINATE                      -->
<!-- ═══════════════════════════════════════════════════════ -->

## 6. MONITOR AND COORDINATE

After spawning all agents, the lead coordinator should:

1. **Track progress** using TodoWrite — create a todo item for each agent
2. **Wait for all agents to complete** — check for agent-{N}-results.json files
3. **Continue coordinating** while agents are running — do NOT attempt to install any external tools.

### Merge Agent Results

After ALL agents complete:

1. **Read each agent-{N}-results.json** file
2. **Deduplicate across agents** — if two agents found the same job (match by URL or company+role), keep the one with the higher score
3. **Merge into found-jobs.json** at the candidate root:
   - Read existing found-jobs.json
   - Append all new jobs (qualified, near_miss, skipped) with `"run": "{run-id}"`
   - Write back the complete file
4. **Update summary.md** — add a row for each qualified job
5. **Write search-log.json** — compile all agent events into a single log

Display a progress update:
```
All Agents Complete
━━━━━━━━━━━━━━━━━━
Jobs found: {n}
Qualified: {n} (above threshold)
Near misses: {n}
Skipped: {n}
Duplicates (cross-agent): {n}
Duplicates (prior runs): {n}
```

### Multi-Cycle Runs

If the configured duration allows for multiple cycles (i.e., duration > interval):
- After the first cycle completes and results are merged, spawn a NEW round of agents for the second cycle
- Update the "Previously Found Jobs" list in each new agent's prompt to include all jobs from cycle 1
- All dedup happens against the cumulative found-jobs.json

Calculate cycles: `total_cycles = floor(duration_minutes / interval_minutes)`
If only 1 cycle fits in the duration, run once and proceed to the final report.

---

<!-- ═══════════════════════════════════════════════════════ -->
<!-- SECTION 7: GENERATE FINAL REPORT                       -->
<!-- ═══════════════════════════════════════════════════════ -->

## 7. GENERATE FINAL REPORT

**This is the crown jewel of the entire system.** When all cycles are complete, generate `FINAL-REPORT.md` in the run folder.

The report should be **500-1000 lines**, polished, well-formatted, and useful enough that someone unfamiliar with the candidate could read it and understand the full picture.

Read the merged agent results and `search-log.json` to compile the report.

### FINAL-REPORT.md Structure

```markdown
# Job Search Report: {Candidate Name}

**Run ID:** {run-id}
**Date:** {date}
**Duration:** {configured duration} (actual: {actual elapsed time})
**Agents:** {n} parallel search agents
**Search Cycles:** {n} completed

---

## Executive Summary

{10-20 lines: who the candidate is, what was searched, high-level results}

---

## Candidate Profile Summary

- **Name:** {name}
- **Location:** {location}
- **Experience:** {years} years ({level})
- **Primary Stack:** {tech list}
- **Target Roles:** {title list}
- **Remote Preference:** {preference}
- **Key Strengths:** {2-3 bullet points from achievements}

---

## Search Parameters

### Auto-Detected (from resume)
- Target titles: {list}
- Stack keywords: {list}
- Experience bracket: {level}
- Location: {location}

### Config Overrides (from config file)
{If config was provided, list overrides. If not: "No config file provided — using auto-detected values only."}

### Sources Searched
{List each source with status: searched successfully / partially accessible / failed / skipped}

---

## Run Context

- **Run number for this candidate:** {n} (e.g., "3rd run" or "first run")
- **Previous runs:** {count}
- **Cumulative jobs in ledger:** {total across all runs}
- **New jobs found this run:** {count}
- **Duplicates from previous runs:** {count}
- **Cross-agent duplicates this run:** {count}

---

## Results Overview

| Metric | Count |
|--------|-------|
| Total jobs scanned | {n} |
| Qualified (score >= {threshold}) | {n} |
| Near misses (score {threshold-20} to {threshold-1}) | {n} |
| Skipped (auto-skip rules) | {n} |
| Duplicates (already in ledger) | {n} |
| Average score (qualified) | {n} |
| Highest score | {n} |
| Lowest qualifying score | {n} |

---

## Top Matches (Ranked by Score)

{For each qualified job, ranked by score descending:}

### {rank}. {Company} — {Role Title} (Score: {X}/100)

| Dimension | Score | Max | Notes |
|-----------|-------|-----|-------|
| Stack Match | {X} | 30 | {brief note} |
| Experience Fit | {X} | 20 | {brief note} |
| Company Stability | {X} | 15 | {brief note} |
| Comp Range | {X} | 15 | {brief note} |
| Remote/Location | {X} | 10 | {brief note} |
| Role Type | {X} | 10 | {brief note} |

- **Compensation:** {range}
- **Location:** {location details}
- **ATS Platform:** {platform} | **Applicants:** {count or "Unknown"}
- **ATS Strategy:** {Brief note: "Modern ATS — keyword optimization helps recruiter search" OR "Legacy ATS — exact keyword matching critical" OR "Low volume — human will likely read every resume"}
- **Why it's a good match:** {2-3 specific alignment points from resume}
- **Concerns:** {any flags — state eligibility, experience stretch, posting age}
- **Apply:** {direct link}
- **Tailored resume:** `{relative path to resume-tailored.md}` (run `/export-resume` to convert to .docx)

---

## Near Misses (Score {threshold-20} to {threshold-1})

{For each near-miss job:}

### {Company} — {Role Title} (Score: {X}/100)
- **Why it fell short:** {specific dimension(s) that scored low}
- **What would improve the match:** {what would need to change}
- **Link:** {url}

---

## Skipped Jobs

{Group by skip reason with counts:}

| Skip Reason | Count | Examples |
|-------------|-------|----------|
| Wrong primary stack | {n} | {1-2 example companies} |
| Experience level mismatch | {n} | {1-2 examples} |
| Not remote / wrong location | {n} | {1-2 examples} |
| Recruiting agency | {n} | {1-2 examples} |
| Config blocklist match | {n} | {1-2 examples} |
| Duplicate from prior run | {n} | — |

---

## ATS Intelligence

### Platform Distribution
{Count of qualified jobs by ATS platform:}

| ATS Platform | Qualified Jobs | Strategy |
|-------------|---------------|----------|
| Greenhouse | {n} | Recruiter-search based. Keyword optimization helps discovery. |
| Ashby | {n} | Modern ATS. Similar to Greenhouse — keyword-friendly, not auto-filter. |
| Lever | {n} | Moderate filtering. Keywords matter but less aggressive than legacy. |
| Workday/iCIMS/Taleo | {n} | Legacy ATS. Exact keyword matching critical — mirror listing terms verbatim. |
| Unknown/Direct | {n} | Likely human-reviewed. Optimize for readability. |

### High-Volume Listings (500+ applicants)
{List any qualified jobs with 500+ applicants and note that ATS filtering is more likely}

### Low-Volume Opportunities (<50 applicants)
{List any qualified jobs with <50 applicants — these are high-probability applications where a human reads every resume}

---

## Market Insights

### Most Requested Skills
{Top 10 skills/technologies seen across all scanned listings, with frequency counts}

### Salary Ranges Observed
{Salary ranges seen for the candidate's experience level, organized by role type}

### Active Companies
{Companies that appeared multiple times across sources}

### Patterns & Observations
- {Pattern 1: e.g., "Most React+TypeScript roles also require Next.js or GraphQL"}
- {Pattern 2: e.g., "3 companies in the qualified list exclude Idaho from eligible states"}
- {Pattern 3: e.g., "Remote-first companies tend to offer $10-20k more than remote-optional"}
- {Any other notable trends}

---

## Recommendations

### Apply First (Priority Order)
1. **{Company} — {Role}** — {why this should be first: best score, deadline, perfect fit, low applicant count, etc.}
2. **{Company} — {Role}** — {reason}
3. **{Company} — {Role}** — {reason}

### Skills Gaps to Address
{Technologies that appeared frequently in matching roles but are missing from the candidate's stack}
- {Skill 1}: appeared in {n} listings — {suggestion for how to gain exposure}
- {Skill 2}: appeared in {n} listings — {suggestion}

### Search Refinements for Next Run
- {Suggestion 1: e.g., "Lower threshold to 60 to catch more near-misses"}
- {Suggestion 2: e.g., "Add 'Next.js' to config prioritize.keywords"}
- {Suggestion 3: e.g., "Remove company X from blocklist — they had a good match"}

### Config Changes Suggested
{If applicable: specific changes to recommend for the config file}
```yaml
# Suggested additions to search-config.yaml
prioritize:
  keywords: ["{new keyword 1}", "{new keyword 2}"]
```

---

## Appendix: All Jobs Found

{Full table of ALL jobs found this run, including near-misses and skipped:}

| # | Company | Role | Score | Status | Source | Agent | ATS | Applicants | Skip Reason | Link |
|---|---------|------|-------|--------|--------|-------|-----|------------|-------------|------|
| 1 | ... | ... | ... | qualified | ... | agent-1 | greenhouse | 45 | — | ... |
| 2 | ... | ... | ... | near_miss | ... | agent-2 | ashby | Unknown | — | ... |
| 3 | ... | ... | ... | skipped | ... | agent-3 | unknown | — | {reason} | ... |

---

*Generated by Job Search Agent System | Run {run-id} | {timestamp}*
```

After writing FINAL-REPORT.md, update run-meta.json with final statistics (ended_at, total counts, sources searched).

---

<!-- ═══════════════════════════════════════════════════════ -->
<!-- SECTION 8: CLEANUP AND FINAL OUTPUT                    -->
<!-- ═══════════════════════════════════════════════════════ -->

## 8. CLEANUP AND FINAL OUTPUT

After generating the final report:

1. Read the final `summary.md` to get the results table
2. Display final status:

```
Job Search Complete
━━━━━━━━━━━━━━━━━━

Duration: {actual time}
Jobs Scanned: {n}
Qualified: {n}
Near Misses: {n}

Top Matches:
1. {Company} — {Role} (Score: {X})
2. {Company} — {Role} (Score: {X})
3. {Company} — {Role} (Score: {X})

Output: {full path to run folder}
Report: {full path to FINAL-REPORT.md}

Next steps:
- Review FINAL-REPORT.md for full analysis
- Check individual job folders for tailored resumes
- Run again with /find-me-a-job to find new listings (previous results are preserved)
```

---

<!-- ═══════════════════════════════════════════════════════ -->
<!-- SECTION 9: CRITICAL RULES                              -->
<!-- ═══════════════════════════════════════════════════════ -->

## CRITICAL RULES

1. **You are the LEAD — do NOT search for jobs yourself.** Your job is to orchestrate agents, manage files, and compile the final report. The Task tool agents do the actual searching.

2. **Spawn all agents in ONE message** with multiple parallel Task tool calls using `run_in_background: true`.

3. **Every agent prompt must be SELF-CONTAINED.** Include the full resume text, full profile JSON, full search config, full scoring rubric, full output instructions, AND the list of previously found job IDs/URLs for dedup. Agents cannot read files you created — they need everything in their prompt.

4. **Agents write to their OWN files only.** Each agent writes to `agent-{N}-results.json`. The lead coordinator merges all results into `found-jobs.json` and `summary.md` AFTER all agents complete. No shared file writes during execution.

5. **Resume-agnostic.** Never hardcode any candidate information. Everything comes from the `--resume` file.

6. **Append-only ledger.** `found-jobs.json` at the candidate root is NEVER overwritten from scratch — only appended to (by the lead coordinator during the merge step).

7. **Nothing is ever deleted.** Previous run folders persist. The cumulative found-jobs.json grows across runs.

8. **Markdown-only output.** Agents produce `.md` files only. Do NOT attempt PDF or DOCX conversion — use `/export-resume` separately if needed. Agents should NEVER attempt to install packages.

9. **Error resilience.** Never crash the entire system because one source failed or one job couldn't be processed. Log errors and continue.

10. **Silent mode.** Minimal terminal output. Status lines only. No verbose commentary. Let the files speak.

11. **The FINAL-REPORT.md is the primary deliverable.** Invest effort in making it comprehensive, well-formatted, and actionable.

12. **Contact info validation.** After all agents complete and before generating the final report, scan every `resume-tailored.md` file for placeholder patterns like `[email]`, `[phone]`, `[name]`, or similar bracket placeholders. If any are found, replace them with the actual values from the candidate's resume. The tailored resumes must have real contact information — they are useless without it.

13. **ATS optimization is mandatory.** Every tailored resume must follow the ATS Keyword Optimization and Skills Section Formatting rules. The skills section is the single most important section for ATS parsing — treat it as the keyword block that determines whether the candidate appears in recruiter searches.

---

Now parse the arguments and begin.