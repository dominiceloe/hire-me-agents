---
description: "Prepare for a job interview — generate a study guide, run a mock interview, or debrief after the real thing"
arguments:
  - name: args
    description: |
      **Required:**
      --candidate <slug>   Candidate folder name (e.g., dom-eloe)
      --job <id>           Job ID or partial match (e.g., cresta, rocket-money)

      **Mode (pick one, default --prep):**
      --prep               Generate a comprehensive interview prep document
      --mock               Run an interactive mock interview in the terminal
      --debrief            Post-interview debrief and analysis

      **Optional:**
      --focus <areas>      Comma-separated focus areas: behavioral, technical, system-design, culture, negotiation
      --rounds <n>         Mock interview question count (default 10, range 5-25)
      --difficulty <level> Mock difficulty: standard (default), tough, gauntlet

      **Examples:**
      /interview-prep --candidate dom-eloe --job cresta
      /interview-prep --candidate dom-eloe --job lithic --mock --rounds 15 --difficulty tough
      /interview-prep --candidate dom-eloe --job goody --mock --focus behavioral,culture --difficulty gauntlet
      /interview-prep --candidate dom-eloe --job jellyfish --debrief
      /interview-prep --candidate dom-eloe --job cresta --prep --focus technical,system-design
    required: true
tools: [Read, Write, Glob, Bash, WebFetch, WebSearch, AskUserQuestion]
---

<!--
TABLE OF CONTENTS
=================
1. Parse Arguments .................. line ~36
2. Locate the Job ................... line ~60
3. Read All Context ................. line ~71
4. Route to Mode .................... line ~89
5. Prep Mode ........................ line ~97
   5a. Company Research ............. line ~99
   5b. Analyze and Predict Questions  line ~114
   5c. Gap Analysis / Objection ...... line ~149
   5d. AI & Modern Tooling .......... line ~163
   5e. Salary and Negotiation ....... line ~210
   5f. Write the Prep Document ...... line ~224
   5g. Confirm ...................... line ~310
6. Mock Mode ........................ line ~328
   6a. Generate Question Bank ....... line ~330
   6b. Run the Mock Interview ....... line ~359
   6c. Write the Scorecard .......... line ~421
   6d. Confirm ...................... line ~473
7. Debrief Mode ..................... line ~485
   7a. Collect Interview Data ....... line ~487
   7b. Analyze the Interview ........ line ~519
   7c. Compare to Prep .............. line ~532
   7d. Follow-Up Email Suggestions .. line ~543
   7e. Write the Debrief ............ line ~559
   7f. Confirm ...................... line ~614
-->

<!--
SUMMARY: Prepares candidates for interviews via three modes: study guide generation, interactive mock interview, or post-interview debrief.
READS: found-jobs.json, job-details.md, instructions.md, resume-tailored.md, resume-base.md, candidate-profile.json, search-config.yaml, web research
PRODUCES: interview-prep.md (prep mode), mock-interview-{timestamp}.md (mock mode), or debrief-{timestamp}.md (debrief mode)
-->

# Interview Prep

You help candidates prepare for job interviews. You have three modes: generate a comprehensive prep document (`--prep`), run a live mock interview (`--mock`), or conduct a post-interview debrief (`--debrief`).

**CRITICAL:** In `--prep` and `--debrief` modes, all long-form output must be written to FILES, never to the terminal. Terminal output should only be short status messages. In `--mock` mode, the interaction happens live in the terminal — questions and feedback are displayed interactively, but the final scorecard is written to a file.

<!-- ═══════════════════════════════════════════════════════ -->
<!-- SECTION 1: PARSE ARGUMENTS                             -->
<!-- ═══════════════════════════════════════════════════════ -->

## 1. PARSE ARGUMENTS

Extract from `$ARGUMENTS`:

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--candidate` | YES | — | Candidate slug (folder under `~/job-search-candidates/`) |
| `--job` | YES | — | Job ID or partial match |
| `--prep` | NO | default mode | Generate prep document |
| `--mock` | NO | — | Run interactive mock interview |
| `--debrief` | NO | — | Post-interview debrief |
| `--focus` | NO | all areas | Comma-separated: `behavioral`, `technical`, `system-design`, `culture`, `negotiation` |
| `--rounds` | NO | `10` | Number of mock questions (5-25) |
| `--difficulty` | NO | `standard` | One of: `standard`, `tough`, `gauntlet` |

**Validation:**
- If `--candidate` is missing: ERROR `"--candidate is required."`
- If `--job` is missing: ERROR `"--job is required."`
- If more than one mode flag is provided: ERROR `"Pick one mode: --prep, --mock, or --debrief."`
- If `--rounds` is outside 5-25: clamp to nearest bound and WARN `"Rounds clamped to {n} (valid range: 5-25)."`
- If `--difficulty` is not one of the valid values: ERROR `"Invalid difficulty '{value}'. Use: standard, tough, gauntlet."`
- If `--focus` contains invalid areas: ERROR `"Invalid focus area '{value}'. Valid: behavioral, technical, system-design, culture, negotiation."`
- If `--rounds`, `--difficulty`, or `--focus` are provided without `--mock`: WARN `"--rounds, --difficulty, and --focus only apply to --mock mode. Ignoring."` (exception: `--focus` is also respected in `--prep` mode to weight sections)

<!-- ═══════════════════════════════════════════════════════ -->
<!-- SECTION 2: LOCATE THE JOB                              -->
<!-- ═══════════════════════════════════════════════════════ -->

## 2. LOCATE THE JOB

1. Read `~/job-search-candidates/{candidate}/found-jobs.json`
2. Match `--job` against the jobs array:
   - Exact ID match first
   - Then partial ID match (job ID contains the search string)
   - Then case-insensitive company name match
3. If zero matches: ERROR `"No job matching '{--job}' found in ledger."`
4. If multiple matches: list them and ask the user to be more specific.
5. Note the matched job's `id`, `run`, `company`, `role`, `comp_range`, `score`, `score_breakdown`, and any other fields.

<!-- ═══════════════════════════════════════════════════════ -->
<!-- SECTION 3: READ ALL CONTEXT                            -->
<!-- ═══════════════════════════════════════════════════════ -->

## 3. READ ALL CONTEXT

Read these files from the job's run folder (`~/job-search-candidates/{candidate}/runs/{run}/{job-id}/`):

1. **job-details.md** — Full job description, requirements, nice-to-haves, company info
2. **instructions.md** — Match analysis, application tips, score breakdown, concerns
3. **resume-tailored.md** — The tailored resume for this role
4. **cover-letter.md** (or .txt) — If a cover letter was generated

Also read from the candidate root (`~/job-search-candidates/{candidate}/`):

5. **resume-base.md** — Full unfiltered background and work history
6. **candidate-profile.json** — Structured profile with stack, experience, target titles
7. **search-config.yaml** — Preferences, salary minimums, avoid rules

If any file is missing, work with what's available — don't error out. But if job-details.md is missing, WARN `"No job-details.md found — prep quality will be limited."`.

<!-- ═══════════════════════════════════════════════════════ -->
<!-- SECTION 4: ROUTE TO MODE                               -->
<!-- ═══════════════════════════════════════════════════════ -->

## 4. ROUTE TO MODE

Based on the mode flag, proceed to the appropriate section:
- `--prep` (default) → Section 5
- `--mock` → Section 6
- `--debrief` → Section 7

---

<!-- ═══════════════════════════════════════════════════════ -->
<!-- SECTION 5: PREP MODE                                   -->
<!-- ═══════════════════════════════════════════════════════ -->

## 5. PREP MODE — Generate Interview Prep Document

### 5a. COMPANY RESEARCH

Use WebSearch and WebFetch aggressively to research the company. Search for ALL of the following:

1. **Recent news** — Search `"{company}" news {current year}` — funding rounds, product launches, layoffs, acquisitions, pivots
2. **Engineering blog** — Search `"{company}" engineering blog` — tech stack details, architecture decisions, team culture
3. **Glassdoor interview reviews** — Search `"{company}" interview questions glassdoor` — these often reveal exact questions the company asks. This is gold.
4. **Leadership team** — Search `"{company}" engineering leadership` or `"{company}" CTO VP Engineering` — knowing who you might talk to helps
5. **Culture and values** — Search `"{company}" company culture values` — look for their careers page, about page, or press about culture
6. **Tech stack details** — Search `"{company}" tech stack` or `"{company}" engineering stack` — StackShare, blog posts, job listings
7. **Funding and growth** — Search `"{company}" funding crunchbase` — stage, investors, trajectory
8. **AI stance and investment** — Search `"{company}" AI artificial intelligence strategy`, `"{company}" machine learning engineering blog`, and `"{company}" AI acquisitions investments`. Also scan the job listing itself for AI/ML mentions, and check recent press and leadership statements for AI positioning. This informs the AI & Modern Tooling Positioning section — if the company is heavily invested in AI, that section goes deep; if there are zero signals, it stays brief.

Spend real effort here. The more context you gather, the better the prep doc. Don't settle for surface-level results.

### 5b. ANALYZE AND PREDICT QUESTIONS

Using all gathered context (job description, company research, candidate background), generate questions across these categories:

#### Behavioral Questions
- Identify culture signals in the job listing: humility, ownership, collaboration, speed, etc.
- For each signal, predict 3-5 questions the interviewer might ask
- For each predicted question, draft a STAR-format answer outline using REAL projects and experiences from the candidate's resume:
  - **Situation:** Specific context from the candidate's actual work
  - **Task:** What they needed to accomplish
  - **Action:** What they specifically did
  - **Result:** Concrete outcome, metrics if available
- NEVER fabricate experiences. If the candidate doesn't have a perfect example, use the closest relevant one and note how to bridge it

#### Technical Questions
- Based on the required stack and the candidate's experience:
  - Predict language/framework-specific questions (e.g., "explain React's reconciliation algorithm" for React roles)
  - Predict system design questions relevant to the company's domain (e.g., "design a real-time bidding system" for an ad-tech company)
  - Identify areas where the candidate is strong — these are opportunities to shine
  - Identify areas where the candidate should brush up — be specific about what to study
- If there's a stack gap (job requires X, candidate knows Y), predict how the interviewer will probe it and prepare honest, confident responses

#### Role-Specific Questions
- Based on the type of role (product eng, platform, infra, etc.):
  - Predict questions about the candidate's approach to the work
  - Examples: "How do you handle scope creep?" for product eng, "Walk me through your deployment process" for platform, "How do you prioritize reliability vs. feature velocity?" for infra
  - Draft answers that reflect the candidate's actual experience and approach

#### Questions the Candidate Should Ask THEM
- Tailor these to the company's specific situation — NOT generic "what does success look like" questions
- Use the company research: reference recent news, growth stage, team structure, tech decisions
- Good candidate questions are a differentiator. They show preparation and genuine curiosity.
- Categorize: team/culture questions, technical/architecture questions, business/growth questions
- Include 8-12 strong questions. The candidate won't ask all of them but should have options.

### 5c. GAP ANALYSIS AND OBJECTION HANDLING

From the score breakdown in instructions.md and any concerns listed:

1. Identify every gap between the candidate and the job requirements
2. For each gap, predict how the interviewer will probe it (what question they'll ask)
3. Draft a response that addresses the gap honestly and confidently:
   - Acknowledge the gap directly — don't dodge
   - Reframe with transferable experience
   - Show awareness of what they'd need to learn
   - Express genuine interest in growing in that area (only if true)
4. Use the same philosophy as the cover letter command — honest confidence beats fake perfection

### 5d. AI & MODERN TOOLING POSITIONING

Analyze the candidate's AI experience and the company's AI direction to generate positioning guidance.

#### Assess the candidate's AI signals

Read resume-base.md and candidate-profile.json carefully for:
- **AI/ML development tools in daily workflow** — Claude Code, Cursor, Copilot, ChatGPT, Cody, etc.
- **AI-related side projects** — fine-tuning models, building AI-powered tools, RAG pipelines, chatbots, etc.
- **Agentic systems experience** — multi-agent coordination, tool orchestration, prompt engineering, AI pipeline design
- **AI/ML skills in their stack** — PyTorch, TensorFlow, LangChain, vector databases, embedding models, etc.

If the candidate has minimal or no AI experience, note that. The section should still be generated but will focus on Tier 1 (AI tool usage framing) rather than deeper tiers.

#### Assess the company's AI signals

From the research gathered in 5a (search #8) and the job listing itself, classify the company's AI investment level:
- **Heavy AI investment** — AI is core to their product, they have ML teams, AI appears in their strategy/press/blog, leadership talks about AI publicly
- **Moderate AI interest** — AI mentioned in job listings or blog but not central to the product, exploring AI features
- **No clear AI signals** — No AI mentions found in research or job listing

#### Generate three-tier positioning guidance

**Tier 1: Natural mentions (always generate — use in any interview)**
- How to naturally reference AI tooling when describing day-to-day workflow without making it the topic
- Framing: AI as a force multiplier for existing engineering skills, not a replacement for thinking
- 3-5 specific one-liners the candidate can drop when asked "how do you work?" or "how do you stay current?"
- Examples calibrated to the candidate's actual usage — don't suggest talking about tools they don't use

**Tier 2: If AI/projects come up (generate if candidate has AI side projects or meaningful AI experience)**
- Which side projects to highlight and how to frame them — focus on the AI angle without making it the whole story
- Technical depth to show: what they actually built, what they learned, what's hard about it
- Framing for different audiences: recruiter gets the "what and why," technical interviewer gets architecture and trade-offs
- If the candidate has experience with agentic systems, prompt engineering, or fine-tuning — these are differentiators worth surfacing

**Tier 3: Strategic alignment (generate ONLY if the company has moderate or heavy AI investment)**
- How to connect the candidate's AI experience to the company's strategic direction
- Talking points that position the candidate as a fit for where the company is heading, not just the role today
- Specific references to the company's AI initiatives discovered in research (blog posts, product features, acquisitions)
- Frame as genuine interest and awareness, not a sales pitch

#### Generate "What NOT to Say" warnings
- Never frame AI as doing your work for you. Don't say "I use AI to write my code." The framing is always about augmentation, speed, and architectural thinking.
- Don't lead with AI in a recruiter screen unless they bring it up or the role is AI-specific. Read the room.
- Match the depth of AI discussion to the interview stage and interviewer: recruiter gets surface-level signals, technical interviewer gets architecture and tooling discussions, hiring manager gets strategic alignment.
- Don't overstate AI skills. If the candidate uses Copilot for autocomplete, that's different from building fine-tuned models. Be honest about the level.
- Don't position AI experience as a substitute for fundamentals. It's additive, not compensatory.

### 5e. SALARY AND NEGOTIATION

Build a negotiation section:

1. **Market data:** What the role typically pays (use WebSearch for salary data: levels.fyi, Glassdoor, etc.)
2. **Listed comp:** What the job posting says (from found-jobs.json `comp_range`)
3. **Candidate's floor:** From search-config.yaml `min_salary`
4. **Talking points if comp comes up early:**
   - How to deflect premature salary questions
   - How to state a range without anchoring low
   - What to say if their range is below the candidate's floor
   - When and how to negotiate (timing matters)
5. **Total comp considerations:** Equity, benefits, signing bonus, remote flexibility as negotiation levers

### 5f. WRITE THE PREP DOCUMENT

Write everything to `~/job-search-candidates/{candidate}/runs/{run}/{job-id}/interview-prep.md`

Structure the document as:

```
# Interview Prep: {Company} — {Role}

*Generated: {date}*
*Match Score: {score}/100*

---

## Company Overview
{Synthesized company research — who they are, what they do, recent news, stage, size}

## Team & Leadership
{Engineering leadership, team structure if discoverable, who the candidate might interview with}

## Culture & Values
{What the company values, how it shows up in the listing and public materials}

## Tech Stack
{Known stack, how it maps to the candidate's experience}

---

## Predicted Interview Questions

### Behavioral
{Questions with STAR-format answer outlines}

### Technical
{Technical questions with preparation notes}

### System Design
{System design scenarios relevant to the company's domain}

### Role-Specific
{Questions about approach to the specific type of work}

---

## Questions to Ask Them
{Categorized questions the candidate should ask, tailored to this company}

---

## Gap Analysis & Objection Handling
{Each gap with predicted probe question and prepared response}

---

## AI & Modern Tooling Positioning

**Company AI Signal Level:** {Heavy / Moderate / None}

### Tier 1: Natural Mentions (Use in Any Interview)
{How to reference AI tooling naturally when describing workflow}
{3-5 ready-to-use one-liners}

### Tier 2: When AI/Projects Come Up
{Which projects to highlight, how to frame them, technical depth by audience}
{Only include if candidate has AI side projects or meaningful AI experience — omit this subsection entirely if not}

### Tier 3: Strategic Alignment
{How candidate's AI experience connects to company direction, specific talking points referencing company initiatives}
{Only include if company has moderate or heavy AI investment — omit this subsection entirely if not}

### What NOT to Say
{Specific warnings: don't frame AI as doing your work, match depth to interviewer, don't overstate}

---

## Salary & Negotiation
{Market data, listed comp, candidate's floor, talking points, strategy}

---

## Study Checklist
{Specific things to review before the interview — technologies to brush up on, company blog posts to read, etc.}
```

This document should be LONG and thorough — it's the candidate's study guide. Don't be brief. Aim for comprehensive coverage that the candidate can spend real time with.

### 5g. CONFIRM

After writing the file, display only:

```
Interview prep written to:
{full file path}

Sections: Company Overview, Behavioral, Technical, System Design, Role-Specific, Questions to Ask, Gap Analysis, AI & Tooling Positioning, Salary & Negotiation, Study Checklist
Word count: {n}
Company research sources: {number of web sources consulted}
Predicted questions: {total count across all categories}
```

Do NOT output the prep document contents to the terminal.

---

<!-- ═══════════════════════════════════════════════════════ -->
<!-- SECTION 6: MOCK MODE                                   -->
<!-- ═══════════════════════════════════════════════════════ -->

## 6. MOCK MODE — Interactive Mock Interview

### 6a. GENERATE QUESTION BANK

Using the same analysis as prep mode (job description, company research, candidate background), generate a bank of questions. Weight the question distribution based on `--focus` if provided, otherwise use a balanced mix:

**Default distribution (no --focus):**
- Behavioral: 30%
- Technical: 30%
- System Design: 15%
- Role-Specific: 15%
- Culture/Negotiation: 10%

**AI-related questions (include 1-2 in every mock by default):**
Always include at least one AI-related question in the question bank, drawn from these depending on context:
- "How do you use AI tools in your development workflow?" — categorize as behavioral. Good for any interview.
- "What's your take on AI-assisted development? How does it change how you work?" — categorize as culture. Tests self-awareness and adaptability.
- If the company has AI strategy (from research): "How would you approach integrating AI capabilities into {company's product domain}?" — categorize as technical/role-specific. Only include if company AI signals are moderate or heavy.
- For `tough` or `gauntlet` difficulty, add follow-ups like: "How do you ensure code quality when using AI-generated code?" or "Where do you draw the line between AI assistance and understanding the code yourself?"

These should be woven into the natural question flow — not grouped together. They're part of the regular distribution, not a separate category.

**If --focus is provided:** Weight heavily toward the specified areas while keeping a minimum of 1 question from other areas.

**Difficulty adjustments:**
- `standard` — Typical first-round questions. Clear, direct, one question at a time. Friendly interviewer tone.
- `tough` — Senior/staff level. Follow-up questions after responses ("tell me more about that," "what would you do differently?"). Expects depth and specificity. Professional but probing tone.
- `gauntlet` — Deliberately adversarial. Stress questions ("why shouldn't we hire you?"), rapid follow-ups, curveball scenarios, challenges assumptions in responses. Not mean, but intense — simulates the hardest interviewer the candidate might face.

Generate at least `--rounds` + 5 questions so you have flexibility to adapt based on responses.

### 6b. RUN THE MOCK INTERVIEW

Begin the interactive session:

1. **Introduction:**
```
Starting mock interview for {Company} — {Role}
Difficulty: {standard/tough/gauntlet}
Rounds: {n}
Focus: {areas or "balanced"}

I'll ask you questions one at a time. Type your response as if you were speaking to an interviewer. Take your time — there's no timer.

Type "skip" to skip a question, "stop" to end early.

---
```

2. **For each question (1 through rounds):**

   a. Display the question:
   ```
   Question {n}/{total} [{category}]

   {question text}
   ```
   If difficulty is `tough` or `gauntlet`, you may add context or constraints to the question.

   b. Wait for the candidate to type their response using `AskUserQuestion`. Present the question and let them type freely.

   c. If they type "skip": note it in the scorecard and move to the next question.

   d. If they type "stop": end the interview early and proceed to scorecard.

   e. After receiving their response, provide feedback IMMEDIATELY before moving to the next question:

   ```
   Feedback:

   Score: {Strong ✓ / Good ~ / Needs Work ✗}

   What worked:
   - {specific strength in their response}
   - {another strength if applicable}

   What was missing:
   - {specific gap or missed opportunity}

   Suggested improvement:
   {A tighter version of their answer — not a full rewrite, but the key adjustment that would make it land better}
   ```

   For `tough` difficulty: After feedback, ask ONE follow-up question before moving on.
   For `gauntlet` difficulty: After feedback, challenge something in their answer ("But what about...", "That sounds like...", "Your competitor would say...") and ask for a revised response. Then provide final feedback.

   f. Move to the next question.

3. **End of interview:**
```
Mock interview complete. Writing scorecard...
```

### 6c. WRITE THE SCORECARD

Get timestamp via `date -u +%Y-%m-%dT%H:%M:%SZ` and format it for the filename as `YYYYMMDD-HHMM`.

Write to `~/job-search-candidates/{candidate}/runs/{run}/{job-id}/mock-interview-{timestamp}.md`:

```
# Mock Interview Scorecard: {Company} — {Role}

*Date: {date}*
*Difficulty: {level}*
*Rounds: {completed}/{planned}*
*Focus: {areas}*

## Overall Assessment

**Overall Score: {Strong/Good/Needs Work}**

{2-3 sentence summary of performance — patterns, strengths, areas to work on}

---

## Question-by-Question Breakdown

### Q1: {question text} [{category}]

**Candidate's Response:**
{what they said}

**Score:** {Strong ✓ / Good ~ / Needs Work ✗}

**Feedback:**
{what worked, what was missing, suggested improvement}

---

{repeat for all questions}

---

## Patterns & Recommendations

### Strengths
- {patterns that showed up across multiple answers}

### Areas to Improve
- {recurring gaps or weaknesses}

### Specific Study Recommendations
- {concrete things to practice or review before the real interview}
```

### 6d. CONFIRM

```
Scorecard written to:
{full file path}

Results: {count Strong} Strong, {count Good} Good, {count Needs Work} Needs Work
Overall: {overall assessment}
```

---

<!-- ═══════════════════════════════════════════════════════ -->
<!-- SECTION 7: DEBRIEF MODE                                -->
<!-- ═══════════════════════════════════════════════════════ -->

## 7. DEBRIEF MODE — Post-Interview Analysis

### 7a. COLLECT INTERVIEW DATA

Run an interactive session to gather what happened:

1. **Introduction:**
```
Let's debrief your interview at {Company} for {Role}.

I'll ask you about each question they asked. For each one, tell me:
1. What they asked
2. How you answered (brief summary is fine)

Type "done" when you've covered all the questions.

---
```

2. **For each question:**

   a. Ask: `"What was question {n}?"` using `AskUserQuestion`

   b. If they type "done": proceed to analysis.

   c. After they provide the question, ask: `"How did you answer?"` using `AskUserQuestion`

   d. Record both the question and their response summary.

   e. Continue: `"Next question? (or type 'done' if that's all)"` using `AskUserQuestion`

3. After collecting all questions, ask one final question using `AskUserQuestion`:
   `"Overall, how do you feel it went? Any moments that felt particularly strong or weak?"`

### 7b. ANALYZE THE INTERVIEW

For each question the candidate reported:

1. **What they were evaluating:** Explain what skill, trait, or competency the interviewer was likely testing with that question. This is the most valuable part of the debrief — help the candidate understand the "why" behind each question.

2. **Response analysis:**
   - What was strong about their answer
   - What could have been better — be specific and constructive
   - What the interviewer was probably hoping to hear

3. **Improved response:** Draft what a stronger answer would have sounded like, using the candidate's actual experience as the foundation.

### 7c. COMPARE TO PREP (if available)

If `interview-prep.md` exists in the job folder:

1. Compare the actual questions against predicted questions
2. Note which predictions were accurate (validates the prep)
3. Note what surprised — questions that weren't predicted
4. Assess whether the prep document was useful and what could be added for future interviews

If no prep document exists, skip this section.

### 7d. FOLLOW-UP EMAIL SUGGESTIONS

Based on how the interview went:

1. **Thank-you email guidance:**
   - What to emphasize (their strongest moments)
   - What to clarify or expand on (moments that felt weak)
   - Specific details to reference that show attentiveness (something the interviewer said, a project discussed)
2. **Draft 2-3 key sentences** for the thank-you email — not the full email, just the high-value lines the candidate should include
3. **Timing:** When to send it (same day, within 24 hours)
4. **AI mention guidance (if AI came up during the interview):**
   - If AI tools or AI experience was discussed, include a brief note on how to reinforce it in the thank-you without overdoing it
   - The goal is to leave a signal of AI fluency — one sentence max, positioned as forward-thinking, not as the main pitch
   - Example framing: referencing a specific tool or approach discussed, or connecting it to a company initiative mentioned by the interviewer
   - If AI didn't come up at all, don't force it into the follow-up email — skip this point entirely

### 7e. WRITE THE DEBRIEF

Get timestamp via `date -u +%Y-%m-%dT%H:%M:%SZ` and format it for the filename as `YYYYMMDD-HHMM`.

Write to `~/job-search-candidates/{candidate}/runs/{run}/{job-id}/debrief-{timestamp}.md`:

```
# Interview Debrief: {Company} — {Role}

*Date: {date}*
*Overall Feeling: {candidate's self-assessment}*

---

## Question-by-Question Analysis

### Q1: {question as reported}

**What they were evaluating:** {skill/competency being tested}

**Your response (summary):** {what the candidate said they answered}

**Analysis:**
- Strengths: {what worked}
- Could improve: {specific suggestions}

**Stronger version:**
{improved response using candidate's real experience}

---

{repeat for all questions}

---

## Prep Accuracy
{If interview-prep.md existed: comparison of predicted vs actual questions}

## Patterns
- **Strongest moments:** {what went well across the interview}
- **Areas to improve:** {recurring themes in what could be better}

## Follow-Up Email
**Send within:** 24 hours
**Key points to hit:**
- {point 1}
- {point 2}
- {point 3}

**Suggested lines:**
- "{draft sentence 1}"
- "{draft sentence 2}"
- "{draft sentence 3}"
```

### 7f. CONFIRM

```
Debrief written to:
{full file path}

Questions analyzed: {count}
Prep predictions matched: {count if prep existed, or "No prep document found"}
Follow-up email points: {count}
```
