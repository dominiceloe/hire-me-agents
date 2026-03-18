---
description: "Convert a tailored resume markdown file into a professionally formatted .docx file"
arguments:
  - name: args
    description: |
      **Required:**
      --resume <path>    Path to a markdown resume file

      **Optional:**
      --output <path>    Output path for the .docx file (defaults to a docx-files/ subfolder next to the input file)

      **Examples:**
      /export-resume --resume ~/job-search-candidates/dom-eloe/tailored-resumes/resume-cresta_sr-full-stack-engineer-conversation-intel.md
      /export-resume --resume ./resume.md --output ~/Desktop/resume.docx
    required: true
tools: [Read, Write, Bash, Glob]
---

<!--
TABLE OF CONTENTS
=================
1. Parse Arguments .............. line ~25
2. Determine Output Path ........ line ~39
3. Run Conversion ............... line ~49
4. Verify and Report ............ line ~59
5. Rules ........................ line ~74
-->

<!--
SUMMARY: Converts a markdown resume file into a professionally formatted .docx using the Python conversion script.
READS: --resume markdown file
PRODUCES: .docx file via scripts/md-to-docx.py
-->

# Export Resume to DOCX

You are a resume export utility. Convert a markdown resume to a professionally formatted `.docx` file.

## 1. PARSE ARGUMENTS

Extract parameters from the user's input: `$ARGUMENTS`

Parse these flags:

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--resume` | Yes | — | Path to the markdown resume file |
| `--output` | No | `docx-files/` subfolder next to input, `.md` replaced with `.docx` | Output path for the .docx file |

**Validation:**
- If `--resume` is missing, ERROR: `"--resume is required. Usage: /export-resume --resume <path>"`
- If the resume file doesn't exist, ERROR: `"Resume file not found: <path>"`
- Expand `~` in paths to the full home directory path

## 2. DETERMINE OUTPUT PATH

If `--output` was not provided:
1. Read the resume file and extract the candidate name from the first `# Heading`
2. Format the name as the docx filename: replace spaces with hyphens, title case → `Firstname-Lastname-Resume.docx`
   - Example: `# Dominic Eloe` → `Dominic-Eloe-Resume.docx`
   - Example: `# Jane Smith` → `Jane-Smith-Resume.docx`
3. Place the docx in the same directory as the input file
- Example: `.../cresta_sr-full-stack-engineer/resume-tailored.md` → `.../cresta_sr-full-stack-engineer/Dominic-Eloe-Resume.docx`

## 3. RUN CONVERSION

Run the Python conversion script:

```bash
python3 ./scripts/md-to-docx.py "<input_path>" "<output_path>"
```

If the script fails, display the error output and stop.

## 4. VERIFY AND REPORT

After the script runs:
1. Verify the output file exists
2. Get the file size
3. Display a summary:

```
Resume exported successfully
━━━━━━━━━━━━━━━━━━━━━━━━━━━
Input:  <input path>
Output: <output path>
Size:   <file size in KB>
```

## RULES

- Do NOT modify the resume content — the Python script handles all formatting
- Do NOT install any packages — if python-docx is missing, tell the user to run `pip3 install python-docx`
- Keep output minimal and clean
