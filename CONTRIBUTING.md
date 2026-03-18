# Contributing

Thanks for considering contributing to hire-me-agents.

## How It Works

This repo is pure prompt architecture — the slash commands are markdown files in `.claude/commands/` that Claude Code interprets at runtime. There's no application code to build or test in the traditional sense.

## Adding a New Slash Command

1. Create a new `.md` file in `.claude/commands/`
2. Include YAML frontmatter with:
   ```yaml
   ---
   description: "Short description of what the command does"
   arguments:
     - name: args
       description: |
         Flag definitions and examples
       required: true
   tools: [Read, Write, Bash, ...]
   ---
   ```
3. Structure the command body with numbered sections: parse arguments, do work, display output
4. Follow the existing convention: long-form output goes to files, terminal output is minimal status messages
5. If the command writes to the candidate workspace, use the same path conventions (`~/job-search-candidates/{candidate}/`)
6. Update the command chain table in `CLAUDE.md`
7. Add the command to the Commands Reference table in `README.md`

## Adding a New Job Source

The `/find-me-a-job` command distributes job sources across parallel search agents. To add a new source:

1. Open `.claude/commands/find-me-a-job.md`
2. Find the **Agent Assignment Strategy** section — add the new source to one of the agent groups
3. Add a new subsection under **Search Strategy Per Source** with:
   - The base URL and search approach
   - How to extract job details from the source
   - Any rate-limiting or access notes
4. Update the source list in `build-search-config.md` under `search.target_sources`

## PR Guidelines

- One feature or fix per PR
- If you change a command, update README.md and CLAUDE.md if the command chain or conventions change
- Test with a sample resume before submitting — run `/setup-candidate` and at least one downstream command
- Keep command files self-contained — agents receive the command prompt directly and can't read other files

## Where to Start

Check the [issues labeled `good first issue`](../../issues?q=is%3Aissue+is%3Aopen+label%3A%22good+first+issue%22) for approachable tasks. These are scoped, well-documented, and designed to help you understand the codebase.

Current priorities:
- Making the data root configurable across all commands
- Adding a `--sources` flag for search flexibility
- Improving the scoring rubric for startups

For larger architectural work, see issues labeled `architecture`.

## Reporting Issues

Use the issue templates for bugs and feature requests. Include:
- Which command you were running
- The error or unexpected behavior
- Your Claude Code version (`claude --version`)
