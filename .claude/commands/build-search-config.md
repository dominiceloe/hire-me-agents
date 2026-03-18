---
description: "Analyze a resume and auto-generate a search-config.yaml for use with /find-me-a-job"
arguments:
  - name: args
    description: |
      **Required:**
      --resume <path>    Path to a markdown resume file

      **Optional:**
      --output <path>    Where to write the config. Default: same directory as resume
      --strict           Generate stricter filters (fewer results, higher relevance)
      --loose            Generate looser filters (more results, broader net)

      **Examples:**
      /build-search-config --resume ~/resumes/dom.md
      /build-search-config --resume ~/resumes/dom.md --output ~/configs/dom-search.yaml
      /build-search-config --resume ~/resumes/dom.md --strict
    required: true
tools: [Read, Write, Bash, WebSearch, WebFetch]
---

<!--
TABLE OF CONTENTS
=================
1. Parse Arguments .............. line ~30
2. Read and Analyze Resume ...... line ~50
3. Generate Config .............. line ~75
   3A. Preferences .............. line ~79
   3B. Avoid Rules .............. line ~123
   3C. Prioritize Rules ......... line ~156
   3D. Background Section ....... line ~177
   3E. Search Tuning ............ line ~205
4. Write the Config File ........ line ~229
5. Display Output ............... line ~363
6. Critical Rules ............... line ~392
-->

<!--
SUMMARY: Analyzes a resume to infer ideal job search parameters and outputs a complete search-config.yaml.
READS: --resume markdown file
PRODUCES: search-config.yaml with preferences, avoid rules, prioritize rules, background section, and search tuning
-->

# Search Config Generator

You read a resume, analyze the candidate's profile, infer their ideal job search parameters, and output a complete `search-config.yaml` file ready for use with `/find-me-a-job --config`.

The output config should be immediately usable but also well-commented so the candidate can review and tweak it before running a search.

---

## 1. PARSE ARGUMENTS

Extract from `$ARGUMENTS`:

| Flag | Default | Description |
|------|---------|-------------|
| `--resume` | **REQUIRED** | Path to markdown resume |
| `--output` | Same dir as resume | Where to write search-config.yaml |
| `--strict` | false | Tighter filters, fewer results |
| `--loose` | false | Looser filters, broader net |

**Validation:**
- `--resume` is required
- `--strict` and `--loose` are mutually exclusive — if both provided, ERROR: `"Cannot use --strict and --loose together."`
- If resume file doesn't exist, ERROR with path

Default mode (neither flag) = balanced.

---

## 2. READ AND ANALYZE RESUME

Read the resume file. Extract everything you need to make informed inferences:

### Direct Extraction
- **Name** — for the config header comment
- **Location** — city, state, country
- **Current/most recent title** — exact title held
- **All titles held** — career progression
- **Years of experience** — calculated from work history dates
- **Primary stack** — technologies used heavily in experience sections
- **Secondary stack** — technologies mentioned but not heavily featured
- **Industries worked in** — SaaS, edtech, fintech, etc.
- **Company sizes worked at** — startup, mid-size, enterprise (infer from context clues: "first engineer hired" = startup, "Fortune 500" = enterprise, acquisition stories, team sizes mentioned)
- **Education** — degrees, schools, completion status
- **Achievements** — scale numbers, patents, acquisitions, notable projects
- **Employment gaps** — any gaps > 6 months between roles
- **Career trajectory** — is this person moving up, lateral, pivoting?

### Inference Analysis

From the extracted data, reason through each config section. Think carefully — the quality of these inferences determines whether the job search finds good matches or wastes time.

---

## 3. GENERATE CONFIG

Build the YAML config by inferring each section from the resume analysis.

### 3A. Preferences

**remote_only:**
- If resume shows only remote roles → `true`
- If resume shows mix of remote and onsite → `false` (but note remote preference in comments)
- If resume location is in a major tech hub (SF, NYC, Seattle, Austin, etc.) → `false`
- If resume location is NOT a major tech hub → `true` (remote is likely necessary)
- `--strict` mode: `true` unless resume explicitly shows onsite preference
- `--loose` mode: `false`

**min_salary:**
- Estimate based on: experience level + primary stack + location + most recent title
- Use these rough bands (2026 market, remote US roles):
  - Junior (0-2 yrs): $70k-$100k
  - Mid (3-5 yrs): $100k-$140k
  - Senior (5-8 yrs): $130k-$175k
  - Staff+ (8+ yrs): $170k-$220k
- Adjust up for: FAANG-adjacent stack, specialized skills (WebRTC, ML, etc.), leadership experience
- Adjust down for: non-traditional background, career gaps, location in lower cost-of-living area
- Set min_salary at the LOW END of the appropriate band (we want to cast a reasonable net)
- `--strict` mode: set at MIDPOINT of band
- `--loose` mode: set 20% below low end of band
- Comment explaining the reasoning

**max_salary:**
- Don't set a max — no reason to filter out jobs that pay more than expected

**company_size_min:**
- If resume shows startup experience and comfort → `10`
- If resume shows only mid-size+ companies → `50`
- If resume emphasizes stability → `100`
- `--strict` mode: `100` minimum
- `--loose` mode: `10`

**company_size_max:**
- Generally don't cap this unless resume suggests a preference
- If resume shows only startup/small company experience and no enterprise → `5000` (might struggle in huge orgs)
- Otherwise: omit (no max)

**experience_level_target:**
- Based on years + trajectory
- Include the candidate's level AND one level down (e.g., senior person should also see mid-senior roles)
- Comment explaining why

### 3B. Avoid Rules

**avoid.industries:**
Infer which industries to avoid based on:
- Industries the candidate has NO experience in that require domain expertise (e.g., if no finance background, fintech requiring domain knowledge)
- Industries with heavy regulatory requirements that may not match the candidate's background
- Government/defense (almost always requires clearance + background checks)
- Do NOT avoid industries just because the candidate hasn't worked there — only avoid if the industry requires specialized domain knowledge or credentials the candidate lacks

`--strict` mode: also avoid industries where the candidate has zero signal
`--loose` mode: only avoid government/defense

**avoid.keywords:**
Standard red flags that waste time:
- "security clearance" — requires government background check
- "clearance required" — same
- "FINRA" — financial regulatory certification
- "FedRAMP" — federal security compliance
- "polygraph" — government/intelligence
- Add any domain-specific certifications the candidate clearly doesn't have (e.g., if no healthcare experience, "HIPAA compliance officer")
- Do NOT add technology keywords here — those are handled by the scoring rubric

**avoid.companies:**
- Start empty — the candidate should populate this with companies they've already applied to or want to exclude
- Add a comment telling them to fill this in
- If the resume shows a recent employer, add that employer (don't want to apply back to the company that just employed you)

**avoid.title_patterns:**
- Patterns that indicate wrong level or wrong role type
- If candidate is senior: avoid "intern", "junior", "associate", "entry level"
- If candidate is an IC: avoid "director", "VP", "head of" (unless resume shows management trajectory)
- If candidate is a developer: avoid "DevOps", "SRE", "QA", "support engineer" (unless resume shows this experience)

### 3C. Prioritize Rules

**prioritize.industries:**
- Industries the candidate has direct experience in (highest weight)
- Adjacent industries where their skills transfer well
- Comment explaining the ranking

**prioritize.keywords:**
- Technologies and concepts from the candidate's primary stack
- Specialized skills that differentiate the candidate (e.g., "WebRTC", "real-time", "live streaming")
- Domain terms from their experience (e.g., "community platform", "SaaS", "marketplace")
- Do NOT just repeat their entire tech stack — focus on differentiators

**prioritize.companies:**
- Leave empty with a comment — this is for the candidate to fill in with dream companies
- Optionally: if the candidate's stack and experience strongly align with well-known companies, list 3-5 as suggestions in comments

**prioritize.role_types:**
- Infer from resume: product engineering, platform, infrastructure, developer tools, etc.
- Rank based on what the candidate has actually done — the first entry is their primary type, second is secondary, etc.
- This ranking is used by the scoring rubric in /find-me-a-job: primary role type scores 10, secondary scores 7, tertiary scores 5, unrelated scores 3

### 3D. Background Section

This section handles sensitive employment considerations. Infer conservatively:

**background.has_gaps:**
- `true` if there are employment gaps > 6 months
- If true, add `gap_explanation_hint` with a neutral framing suggestion
- `false` otherwise

**background.has_degree:**
- `true` if a completed degree is listed
- `false` if education shows "attended" or no degree completion
- If false, add a comment noting many companies no longer require degrees

**background.non_traditional_path:**
- `true` if: bootcamp graduate, self-taught, career changer, non-CS degree
- This triggers prioritization of companies known for valuing non-traditional backgrounds

**background.avoid_keywords:**
- If `has_gaps: true`: add nothing (gaps are common and don't need avoidance rules)
- Always include: "security clearance", "government", "polygraph" (these involve deep background checks)
- These are HARD SKIP rules in /find-me-a-job — jobs matching these keywords are skipped entirely

**background.avoid_industries:**
- "government" and "defense" by default (background check intensive)
- "financial services" if the role would require FINRA or similar regulatory checks
- Comment explaining that this is about background check intensity, not industry preference

### 3E. Search Tuning

**search.max_posting_age_days:**
- `--strict`: 14
- balanced: 30
- `--loose`: 60

**search.include_no_salary:**
- `--strict`: `false` (skip listings without salary)
- balanced: `true` (include but score lower)
- `--loose`: `true`

**search.include_contract:**
- If resume shows only full-time roles → `false`
- If resume shows contract/freelance work → `true`
- Default: `false` with comment suggesting they change it if open to contract

**search.target_sources:**
- Always include: ["hn_hiring", "weworkremotely", "greenhouse", "lever"]
- If candidate is in a specific niche, add relevant sources
- Comment listing all available sources

---

## 4. WRITE THE CONFIG FILE

Output a well-structured, heavily-commented YAML file. Comments are critical — they explain WHY each value was chosen so the candidate can make informed edits.

Write to the `--output` path (or same directory as resume if not specified). Filename: `search-config.yaml`

### Template

```yaml
# ============================================================
# Job Search Config — {Candidate Name}
# Generated: {ISO timestamp}
# Resume: {resume filename}
# Mode: {strict|balanced|loose}
#
# This file configures /find-me-a-job search behavior.
# Review each section and adjust before running.
# Lines starting with # are comments — they won't affect the search.
# ============================================================

# --- Preferences ---
# Hard filters — jobs that don't meet these are excluded entirely.
preferences:
  remote_only: {true|false}        # {reasoning}
  min_salary: {number}             # {reasoning for this floor}
  # max_salary: 999999             # Uncomment to cap (not recommended)
  company_size_min: {number}       # {reasoning}
  # company_size_max: {number}     # Uncomment to cap
  experience_level_target:         # Roles to search for
    - "{level}"                    # Primary target
    - "{level-1}"                  # Also consider (wider net)

# --- Avoid Rules ---
# Jobs matching these are auto-skipped (not scored, just dropped).
avoid:
  industries:
    - "{industry1}"                # {why}
    - "{industry2}"                # {why}

  keywords:
    - "security clearance"         # Requires government background check
    - "clearance required"         # Same
    - "FINRA"                      # Financial regulatory certification
    - "FedRAMP"                    # Federal security compliance
    - "polygraph"                  # Government/intelligence requirement
    # Add more as needed:
    # - "keyword"

  companies:
    # Add companies you've already applied to or want to skip:
    # - "Company Name"
    - "{recent_employer}"          # Your most recent employer

  title_patterns:
    - "{pattern1}"                 # {why}
    - "{pattern2}"                 # {why}

# --- Prioritize Rules ---
# Jobs matching these get bonus points in scoring.
prioritize:
  industries:
    - "{industry1}"                # Direct experience — highest relevance
    - "{industry2}"                # Adjacent — skills transfer well
    - "{industry3}"                # Emerging fit based on stack

  keywords:
    - "{differentiator1}"          # {why this stands out}
    - "{differentiator2}"          # {why}
    - "{differentiator3}"          # {why}
    # These are your differentiators — skills that set you apart from
    # generic candidates with similar stacks.

  companies:
    # Add dream companies here:
    # - "Company Name"
    # Suggestions based on your stack and experience:
    # - "{suggestion1}" — {why}
    # - "{suggestion2}" — {why}
    # - "{suggestion3}" — {why}

  role_types:
    - "{type1}"                    # Primary (scores 10) — what you've done most
    - "{type2}"                    # Secondary (scores 7) — also a fit
    # - "{type3}"                  # Tertiary (scores 5) — uncomment if applicable

# --- Background ---
# Sensitive filters. These affect which jobs are hard-skipped.
# Edit carefully — being too aggressive here means missing good opportunities.
background:
  has_gaps: {true|false}           # {gap details if true}
  has_degree: {true|false}         # {degree details}
  non_traditional_path: {true|false}  # {explanation}

  # These keywords trigger HARD SKIPS (job is dropped, not scored):
  avoid_keywords:
    - "security clearance"
    - "government"
    - "polygraph"
    # Only add keywords here that represent genuine dealbreakers.
    # The scoring rubric handles "soft" mismatches — this is for hard stops.

  avoid_industries:
    - "government"                 # Background check intensive
    - "defense"                    # Clearance requirements
    # - "financial services"       # Uncomment if FINRA/regulatory checks are a concern

# --- Search Tuning ---
# Controls how the search agents behave.
search:
  max_posting_age_days: {14|30|60} # {mode} mode — {reasoning}
  include_no_salary: {true|false}  # Include jobs that don't list salary?
  include_contract: {true|false}   # {reasoning}
  # Available sources: hn_hiring, weworkremotely, remoteok, arc_dev,
  #   builtin, greenhouse, lever, linkedin, wellfound, company_direct
  target_sources:
    - "hn_hiring"
    - "weworkremotely"
    - "greenhouse"
    - "lever"
    # - "remoteok"                 # Can be rate-limited
    # - "arc_dev"
    # - "builtin"
    # - "wellfound"
    # - "company_direct"           # Searches career pages of target companies

# ============================================================
# HOW TO USE
# 1. Review and edit this file
# 2. Run: /find-me-a-job --resume {resume_path} --config {this_file_path}
# 3. After the first run, check FINAL-REPORT.md for suggested config changes
# ============================================================
```

---

## 5. DISPLAY OUTPUT

After writing the file, display:

```
Search Config Generated
━━━━━━━━━━━━━━━━━━━━━━━
Candidate: {name}
Mode:      {strict|balanced|loose}
Output:    {full path to config file}

Key Decisions:
  Remote only: {value} — {one-line reason}
  Min salary:  ${value} — {one-line reason}
  Company min: {value} employees
  Avoiding:    {n} industries, {n} keywords, {n} title patterns
  Prioritizing: {n} industries, {n} differentiator keywords

⚠️  Review before running:
  - Check avoid.companies — add any you've already applied to
  - Check prioritize.companies — add dream companies
  - Check background section — adjust if needed
  - Verify min_salary matches your expectations

Ready to search: /find-me-a-job --resume {resume_path} --config {config_path}
```

---

## CRITICAL RULES

1. **Everything is inferred from the resume.** Do not ask questions. Do not prompt for input. Read, analyze, output.

2. **Comments are mandatory.** Every non-obvious value must have a comment explaining WHY it was chosen. The candidate needs to understand the reasoning to make good edits.

3. **Conservative on avoid rules.** When in doubt, DON'T add an avoid rule. It's better to surface a marginal match than to silently skip a good opportunity. The scoring rubric handles soft mismatches — avoid rules are for hard dealbreakers only.

4. **Aggressive on prioritize rules.** Prioritize rules add bonus points — they can't hurt. Be generous with what counts as a differentiator.

5. **Never fabricate information.** If you can't determine something from the resume (e.g., salary expectations), use market data and explain your reasoning in comments. Don't guess at personal preferences.

6. **The config must be valid YAML.** Test by ensuring all strings with special characters are quoted, arrays use consistent formatting, and indentation is correct.

7. **Background section is sensitive.** Be factual, not judgmental. "has_gaps: true" is a fact. Don't editorialize about why gaps exist or what they mean.

8. **Mode affects thresholds, not structure.** Strict/balanced/loose change the VALUES in the config, not which sections exist. Every config has the same complete structure regardless of mode.

---

Now parse the arguments and begin.
