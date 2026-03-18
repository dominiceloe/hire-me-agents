---
description: "Generate a tailored cover letter for a specific job application"
arguments:
  - name: args
    description: |
      **Required:**
      --candidate <slug>   Candidate folder name (e.g., dom-eloe)
      --job <id>           Job ID or partial match (e.g., cresta, rocket-money)

      **Optional:**
      --tone <tone>        Writing tone: conversational (default), formal, brief
      --points <text>      Specific points to hit (e.g., "mention I know someone at the company")
      --format <fmt>       Output format: txt (default), md, docx
      --no-filler          Extra strict: reject any sentence that doesn't earn its place

      **Examples:**
      /write-cover-letter --candidate dom-eloe --job cresta
      /write-cover-letter --candidate dom-eloe --job lithic --tone formal
      /write-cover-letter --candidate dom-eloe --job goody --points "mention their gifting API interests me"
      /write-cover-letter --candidate dom-eloe --job jellyfish --format docx
    required: true
tools: [Read, Write, Glob, Bash]
---

<!--
TABLE OF CONTENTS
=================
1. Parse Arguments .............. line ~30
2. Locate the Job ............... line ~47
3. Read All Context ............. line ~58
4. Analyze the Job Listing ...... line ~71
5. Write the Cover Letter ....... line ~96
6. Self-Check ................... line ~145
7. Confirm ...................... line ~159
-->

<!--
SUMMARY: Generates a tailored cover letter for a specific job application and writes it to a file.
READS: found-jobs.json, job-details.md, instructions.md, resume-tailored.md, resume-base.md
PRODUCES: cover-letter.txt (or .md) in the job's run subfolder
-->

# Cover Letter Generator

You generate a tailored cover letter for a specific job and write it to a file. You NEVER output the cover letter to the terminal — always write it to a file so the user can copy clean text without terminal line-wrapping artifacts.

## 1. PARSE ARGUMENTS

Extract from `$ARGUMENTS`:

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--candidate` | YES | — | Candidate slug (folder under `~/job-search-candidates/`) |
| `--job` | YES | — | Job ID or partial match |
| `--tone` | NO | `conversational` | One of: `conversational`, `formal`, `brief` |
| `--points` | NO | null | Specific talking points the user wants included |
| `--format` | NO | `txt` | One of: `txt`, `md`, `docx` |
| `--no-filler` | NO | false | Extra strict mode — every sentence must earn its place |

**Validation:**
- If `--candidate` is missing: ERROR `"--candidate is required."`
- If `--job` is missing: ERROR `"--job is required."`
- If `--format` is `docx`, fall back to `md` with a note: `"DOCX export not available. Use /export-resume for format conversion. Writing as .md instead."`

## 2. LOCATE THE JOB

1. Read `~/job-search-candidates/{candidate}/found-jobs.json`
2. Match `--job` against the jobs array:
   - Exact ID match first
   - Then partial ID match
   - Then case-insensitive company name match
3. If zero matches: ERROR `"No job matching '{--job}' found in ledger."`
4. If multiple matches: list them and ask the user to be more specific.
5. Note the matched job's `id`, `run`, `company`, `role`, `ats_platform`, and `applicant_count`.

## 3. READ ALL CONTEXT

Read these files from the job's run folder (`~/job-search-candidates/{candidate}/runs/{run}/{job-id}/`):

1. **job-details.md** — Full job description, requirements, nice-to-haves, company info
2. **instructions.md** — Match analysis, application tips, score breakdown, concerns
3. **resume-tailored.md** — The tailored resume for this role

Also read:
4. **resume-base.md** — `~/job-search-candidates/{candidate}/resume-base.md` for the full unfiltered background

If any job package file is missing, work with what's available — don't error out.

## 4. ANALYZE THE JOB LISTING

Before writing, extract these signals from the job description:

### Culture & Values Signals
Scan the job listing for explicit culture/values language. Common patterns:
- **Humility signals:** "humble," "no egos," "team-player approach," "humility" → Write with confidence but avoid sounding like a highlight reel. Add a grounding sentence like "I don't say this to brag — I say it because it maps directly to what you're describing."
- **Mission signals:** "mission-driven," "passionate about," specific cause language → Connect to the mission only if the candidate has a genuine angle. Never fake enthusiasm about a company's product if there's no real connection.
- **Agency/ownership signals:** "high agency," "drive solutions not implementations," "self-starter" → Emphasize times the candidate identified problems and solved them without being asked.
- **Speed/shipping signals:** "ship fast," "move quickly," "daily deployments" → Emphasize rapid iteration, production ownership, deployment frequency.
- **Collaboration signals:** "cross-functional," "partnership with product and design," "building consensus" → Emphasize working with non-engineers, scoping features, translating requirements.

### Gap Analysis
Identify mismatches between the job requirements and the candidate's resume:
- **Years gap:** Job asks for more years than candidate has → Address directly and reframe (e.g., "5 years of intense ownership, not 7 years of ticket work")
- **Stack gap:** Job requires a technology the candidate hasn't used → Address honestly (e.g., "my backend is Python/Django, not Java, but the patterns transfer")
- **Title gap:** Job is a higher level than candidate has held → Don't address unless extreme. Let the work speak.
- **Domain gap:** Job is in an industry the candidate hasn't worked in → Only address if the candidate has a genuine connection or transferable angle. Otherwise skip — don't fake domain enthusiasm.
- **No gap:** If the match is clean, don't manufacture humility. Just make the case.

### Application Context
- If the job has **high applicant volume (500+):** The cover letter needs to be sharper and more specific — generic letters get lost.
- If the job has **low applicant volume (<50):** A human will almost certainly read this. Optimize for personality and genuine connection over keyword density.
- If the application form has a **character limit:** Note this in the output so the user knows to check length.

## 5. WRITE THE COVER LETTER

### Output Path

Based on `--format`:
- `txt`: `~/job-search-candidates/{candidate}/runs/{run}/{job-id}/cover-letter.txt`
- `md`: `~/job-search-candidates/{candidate}/runs/{run}/{job-id}/cover-letter.md`
- `docx`: Write as `.md` (use /export-resume skill separately for DOCX conversion)

### Writing Rules

**Structure:**
- **Opening (1-2 sentences):** State the role, then immediately connect the candidate's strongest relevant experience. No "Dear Hiring Manager." No "I'm excited to apply." Just start talking.
- **Body (2-3 short paragraphs):** Draw direct lines between the candidate's experience and the job's specific requirements. Reference specific projects, technologies, or achievements. If there's a gap, address it here — honestly and confidently.
- **Projects paragraph (optional but recommended):** If the candidate has side projects relevant to this role, mention them concretely — name, URL, what it does, tech stack. Multiple projects are better than one if they show range and curiosity.
- **Close (1-2 sentences):** Brief and confident. Sign off with candidate's full name. No begging, no "I look forward to hearing from you."

**Tone variations:**
- `conversational` (default): Direct, first-person, sounds like a real person wrote it. Use contractions. No corporate speak. This is the tone that works best for most tech companies.
- `formal`: Professional but still human. No contractions. Suitable for enterprise companies, government-adjacent, or companies with formal culture language in the listing.
- `brief`: 3-5 sentences total. Just the essential pitch. Good for forms with tight character limits.

**Content rules:**
- **Mirror the listing's language.** If they say "drive solutions, not just implementations" — use that phrase. If they say "ship to production" — say "ship to production." This shows you read the listing and speaks the team's language. It also helps if the cover letter is parsed alongside the resume in an ATS.
- **Lead with the strongest alignment** between the candidate's experience and the job requirements — not with who the candidate is or what they want.
- If the candidate has a relevant side project, mention it concretely (name, what it does, tech stack). Multiple projects show range.
- If there's a stack mismatch, address it directly and confidently — don't ignore it, don't apologize for it. Frame it as transferable.
- If `--points` was provided, work those points in naturally — don't shoehorn them.
- NEVER fabricate experience, projects, or skills.
- NEVER write things the candidate hasn't expressed or wouldn't say. Don't put words in their mouth about caring about a company's mission unless you have evidence they do.
- NEVER use filler phrases: "I'm excited to apply", "I'm passionate about", "I believe I would be a great fit", "Dear Hiring Manager", "I look forward to the opportunity"
- NEVER use bullet points — this is prose.
- Keep it under 250 words for conversational/formal, under 100 words for brief.
- Sign off with just the candidate's full name — no title, no contact info (that's on the resume).

**What makes a good cover letter:**
- It should say something the resume doesn't already say — the "why this company" and "how I work" angles that a resume can't convey
- It should address WHY this specific company/role, not just recite qualifications — but only if the candidate has a real reason. If not, lead with the work alignment instead of faking enthusiasm.
- It should acknowledge gaps honestly rather than ignoring them — this builds trust
- If the listing emphasizes culture values (humility, teamwork, curiosity), the tone should reflect those values without calling them out by name
- The reader should finish it thinking "this person actually read our job post and they're being straight with me"

**What makes a BAD cover letter:**
- It reads like a resume summary with "I'm excited" bolted on
- It claims enthusiasm for the company's mission with no supporting evidence
- It ignores obvious gaps (wrong stack, fewer years) and hopes nobody notices
- It uses the same generic paragraphs that could apply to any company
- It's longer than it needs to be — every sentence that doesn't earn its place makes the whole letter worse

## 6. SELF-CHECK

Before writing the file, review the draft against these checks:

1. **Filler check:** Does every sentence add new information or advance the case? Cut any that don't.
2. **Fabrication check:** Is everything in the letter verifiable from the resume? Remove anything that isn't.
3. **Mirror check:** Does the letter use at least 2-3 phrases or terms directly from the job listing?
4. **Gap check:** If there's an obvious mismatch (years, stack, title), is it addressed? If not, add a sentence.
5. **Personality check:** Would two different candidates produce the same letter? If yes, it's too generic — add something specific to this candidate.
6. **Length check:** Conversational/formal ≤ 250 words. Brief ≤ 100 words. Cut ruthlessly if over.
7. **Opening check:** Does the first sentence grab attention, or does it start with "I" + generic verb? Rewrite if generic.
8. **Humility check:** If the listing mentions humility/teamwork, does the letter sound like a highlight reel? If yes, add a grounding sentence.

## 7. CONFIRM

After writing the file, display only:

```
Cover letter written to:
{full file path}

Word count: {n}
Format: {txt/md/docx}
Tone: {tone}
Gaps addressed: {list any gaps addressed, or "None — clean match"}
```

Do NOT output the cover letter contents to the terminal.