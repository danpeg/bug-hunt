---
name: bug-hunt
description: "Run adversarial bug hunting on your codebase. Uses 3 isolated agents (Hunter, Skeptic, Referee) to find and verify real bugs with high fidelity. Supports dynamic model assignment with --preset and --hunter/--skeptic/--referee flags, plus branch-diff scans with -b/--base."
argument-hint: "[--preset=claude|codex|gemini|mixed] [--hunter=claude|codex|gemini] [--skeptic=claude|codex|gemini] [--referee=claude|codex|gemini] [path | -b <branch> [--base <base-branch>]]"
disable-model-invocation: true
---

# Bug Hunt - Adversarial Bug Finding

Run a 3-agent adversarial bug hunt on your codebase. Each agent runs in isolation.

## Usage

```
/bug-hunt
/bug-hunt src/
/bug-hunt lib/auth.ts
/bug-hunt -b feature-xyz
/bug-hunt -b feature-xyz --base dev
/bug-hunt --preset=mixed src/
/bug-hunt --hunter=codex --referee=gemini -b feature-xyz --base dev
```

## Arguments

The raw arguments are: $ARGUMENTS

## Step 0: Parse and Validate Arguments

Parse the arguments string above to extract:

1. **Provider selection flags**: `--preset=`, `--hunter=`, `--skeptic=`, `--referee=`
2. **Branch diff flags**: `-b <branch>` and optional `--base <base-branch>`
3. **Scan target**: either a path target or the explicit file list from branch diff mode

**Valid providers:** `claude`, `codex`, `gemini`

**Validation:** If any role-specific provider flag (`--hunter`, `--skeptic`, or `--referee`) contains a value other than `claude`, `codex`, or `gemini`, stop immediately and report the error:
```
Error: Invalid provider "[value]". Valid providers are: claude, codex, gemini
```

If `--preset` contains a value other than `claude`, `codex`, `gemini`, or `mixed`, stop immediately and report the error:
```
Error: Invalid preset "[value]". Valid presets are: claude, codex, gemini, mixed
```

**Preset definitions:**

| Preset | Hunter | Skeptic | Referee |
|--------|--------|---------|---------|
| `claude` (default) | claude | claude | claude |
| `codex` | codex | codex | codex |
| `gemini` | gemini | gemini | gemini |
| `mixed` | codex | claude | gemini |

**Resolution order:**
1. Start with the preset (default: `claude`)
2. Individual `--hunter=`, `--skeptic=`, `--referee=` flags override the preset for that role

**Target resolution:**

1. If arguments contain `-b <branch>`, use **branch diff mode**.
   - Extract the branch name after `-b`.
   - If `--base <base-branch>` is also present, use that as the base branch. Otherwise default to `main`.
   - Run `git diff --name-only <base>...<branch>` using the Bash tool to get the list of changed files.
   - If the command fails (for example, branch not found), report the error to the user and stop.
   - If no files changed, tell the user there are no changes to scan and stop.
   - The scan target is the list of changed files. Scan their full contents, not just the diff.
2. If arguments do NOT contain `-b`, treat the remaining non-provider arguments as a path target (file or directory). If empty, scan the current working directory.

**Path validation:** If a path target was specified, verify it exists using Glob or the filesystem before proceeding. If the target does not exist, stop immediately and report:
```
Error: Scan target "[path]" does not exist.
```

After parsing and validation, announce the configuration to the user:

```
Bug Hunt Configuration:
  Hunter:  [provider]
  Skeptic: [provider]
  Referee: [provider]
  Target:  [path, "current directory", or changed files from <base>...<branch>]
```

## Step 1: Read the prompt files

Read these files using the skill directory variable:
- ${CLAUDE_SKILL_DIR}/prompts/hunter.md
- ${CLAUDE_SKILL_DIR}/prompts/skeptic.md
- ${CLAUDE_SKILL_DIR}/prompts/referee.md

## Step 2: Run the Hunter Agent

Dispatch the Hunter based on its assigned provider:

**If provider is `claude`:**
Launch a general-purpose subagent with the hunter prompt via the Agent tool. Include the resolved scan target in the agent's task. If in branch diff mode, pass the explicit file list so the Hunter only scans those files, using their full contents. The Hunter must use tools (Read, Glob, Grep) to examine the actual code.

**If provider is `codex` or `gemini`:**
Write the full prompt (hunter instructions + scan target) to a unique temporary file using `mktemp`:

```bash
PROMPT_FILE=$(mktemp /tmp/bug-hunt-hunter-XXXXXX.md)
```

Write the prompt content to `$PROMPT_FILE` using the Write tool, then invoke the CLI via stdin:

- **Codex:** `cat "$PROMPT_FILE" | codex exec -`
- **Gemini:** `cat "$PROMPT_FILE" | gemini -p -`

**Never interpolate prompt content or scan target directly into shell command strings.** Always pass via stdin or file to prevent injection.

If the external CLI is not installed, report the error clearly and suggest installing it. Clean up the temp file after capturing the output.

Wait for completion and capture the full output.

## Step 2b: Check Hunter success and findings

**First, verify the Hunter completed successfully.** If the Hunter agent failed (CLI error, crash, empty output, or missing structured findings), stop immediately and report the error to the user. Do not proceed to the Skeptic or Referee.

If the Hunter reported TOTAL FINDINGS: 0, skip Steps 3-4 and go directly to Step 5 with a clean report. No need to run Skeptic and Referee on zero findings.

## Step 3: Run the Skeptic Agent

Dispatch the Skeptic based on its assigned provider:

**If provider is `claude`:**
Launch a NEW general-purpose subagent with the skeptic prompt via the Agent tool. Inject the Hunter's structured bug list (BUG-IDs, files, lines, claims, evidence, severity, points). Do NOT include any narrative or methodology text outside the structured findings.

The Skeptic must independently read the code to verify each claim.

**If provider is `codex` or `gemini`:**
Write the full prompt (skeptic instructions + Hunter's structured bug list) to a unique temporary file using `mktemp`:

```bash
PROMPT_FILE=$(mktemp /tmp/bug-hunt-skeptic-XXXXXX.md)
```

Write the prompt content to `$PROMPT_FILE`, then invoke the CLI via stdin:

- **Codex:** `cat "$PROMPT_FILE" | codex exec -`
- **Gemini:** `cat "$PROMPT_FILE" | gemini -p -`

**Never interpolate prompt content or report data directly into shell command strings.** Always pass via stdin or file.

Clean up the temp file after capturing the output.

Wait for completion and capture the full output.

## Step 4: Run the Referee Agent

Dispatch the Referee based on its assigned provider:

**If provider is `claude`:**
Launch a NEW general-purpose subagent with the referee prompt via the Agent tool. Inject BOTH:
- The Hunter's full bug report
- The Skeptic's full challenge report

The Referee must independently read the code to make final judgments.

**If provider is `codex` or `gemini`:**
Write the full prompt (referee instructions + Hunter's report + Skeptic's report) to a unique temporary file using `mktemp`:

```bash
PROMPT_FILE=$(mktemp /tmp/bug-hunt-referee-XXXXXX.md)
```

Write the prompt content to `$PROMPT_FILE`, then invoke the CLI via stdin:

- **Codex:** `cat "$PROMPT_FILE" | codex exec -`
- **Gemini:** `cat "$PROMPT_FILE" | gemini -p -`

**Never interpolate prompt content or report data directly into shell command strings.** Always pass via stdin or file.

Clean up the temp file after capturing the output.

Wait for completion and capture the full output.

## Step 5: Present the Final Report

Display the Referee's final verified bug report to the user. Include:
1. The configuration used (which provider ran each role)
2. The summary stats
3. The confirmed bugs table (sorted by severity)
4. Low-confidence items flagged for manual review
5. A collapsed section with dismissed bugs (for transparency)

If zero bugs were confirmed, say so clearly - a clean report is a good result.
