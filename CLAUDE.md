# CLAUDE.md

This file provides project-level context for Claude Code when working in this repository.

## What This Is

A set of 9 Claude Code slash commands that turn Claude into a multi-agent job search system. No application code, no runtime dependencies (except one Python script for DOCX export). The repo is pure prompt architecture — markdown files that orchestrate Claude Code's Task, WebSearch, WebFetch, and file management tools into a complete job search pipeline.

Users clone the repo, drop in a resume, and run commands. Everything else is generated.

## Data Model

All candidate data lives **outside** this repo in `~/job-search-candidates/` (configurable). Each candidate gets an isolated workspace:

```
~/job-search-candidates/
└── {candidate-slug}/
    ├── resume-base.md              # Source-of-truth resume
    ├── search-config.yaml          # Search preferences and filters
    ├── found-jobs.json             # Cumulative job ledger (append-only)
    ├── candidate-profile.json      # Structured profile (auto-generated)
    ├── tailored-resumes/           # All tailored resumes across runs
    ├── applications/               # Submitted application tracking
    ├── reports/                    # Exported stats reports
    └── runs/                       # Search run output folders
        └── run-{YYYY-MM-DD-HH-MM}/
            ├── FINAL-REPORT.md     # Primary deliverable per run
            ├── run-meta.json       # Run configuration and stats
            ├── agent-{N}-results.json  # Per-agent search results
            ├── summary.md          # Results table
            └── {company}_{role}/   # Per-job application package
                ├── job-details.md
                ├── instructions.md
                ├── resume-tailored.md
                └── cover-letter.md
```

## Command Chain

The intended workflow order:

| # | Command | Purpose |
|---|---------|---------|
| 1 | `/setup-candidate` | Bootstrap a candidate workspace from a resume |
| 2 | `/build-search-config` | Auto-generate search preferences from resume analysis |
| 3 | `/find-me-a-job` | Spawn parallel agents to search, score, and package jobs |
| 4 | `/add-job` | Manually add a job from a URL and generate full application package |
| 5 | `/write-cover-letter` | Generate a tailored cover letter for a specific job |
| 6 | `/export-resume` | Convert a tailored resume markdown to .docx |
| 7 | `/mark-submitted` | Update application status in the ledger |
| 8 | `/job-stats` | Pipeline stats, weekly activity, and unemployment reporting data |
| 9 | `/interview-prep` | Interview prep document, interactive mock interviews, or post-interview debrief |

## Conventions

- **Append-only ledger.** `found-jobs.json` is never overwritten from scratch — only appended to. Runs accumulate over time.
- **Isolated agent writes.** During `/find-me-a-job`, each search agent writes to its own `agent-{N}-results.json`. The lead coordinator merges all results after agents complete.
- **Files, not terminal.** Long-form output (reports, resumes, cover letters, prep docs) is always written to files. Terminal output is limited to short status messages.
- **Copy, never move.** Original files (resumes, configs) are preserved when copied into the workspace.
- **Nothing is ever deleted.** Previous run folders persist. The ledger grows across runs.

## Default Paths

- **Candidate data root:** `~/job-search-candidates/` (configurable via `--root` in `/setup-candidate`)
- **DOCX conversion script:** `./scripts/md-to-docx.py` (requires `pip3 install python-docx`)

## Dependencies

- [Claude Code](https://claude.ai/code) — the CLI tool that executes these commands
- Python 3 + `python-docx` — only needed for `/export-resume`
