---
name: bug-hunt
description: "Run adversarial bug hunting on your codebase. Uses 3 isolated agents (Hunter, Skeptic, Referee) to find and verify real bugs with high fidelity. Supports dynamic model assignment with --preset and --hunter/--skeptic/--referee flags. Invoke with /bug-hunt or /bug-hunt [options] [path/to/scan]."
argument-hint: "[--preset=claude|codex|gemini|mixed] [--hunter=claude|codex|gemini] [--skeptic=claude|codex|gemini] [--referee=claude|codex|gemini] [path/to/scan]"
disable-model-invocation: true
---

# Bug Hunt - Adversarial Bug Finding

Run a 3-agent adversarial bug hunt on your codebase. Each agent runs in isolation.

## Arguments

The raw arguments are: $ARGUMENTS

## Step 0: Parse Arguments

Parse the arguments string above to extract:

1. **Provider flags**: `--preset=`, `--hunter=`, `--skeptic=`, `--referee=`
2. **Scan target**: Everything that is NOT a flag (no `--` prefix)

**Valid providers:** `claude`, `codex`, `gemini`

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

If no arguments were provided or only a path was given, all roles default to `claude`.

After parsing, announce the configuration to the user:

```
Bug Hunt Configuration:
  Hunter:  [provider]
  Skeptic: [provider]
  Referee: [provider]
  Target:  [path or "current directory"]
```

## Step 1: Read the prompt files

Read these files using the skill directory variable:
- ${CLAUDE_SKILL_DIR}/prompts/hunter.md
- ${CLAUDE_SKILL_DIR}/prompts/skeptic.md
- ${CLAUDE_SKILL_DIR}/prompts/referee.md

## Step 2: Run the Hunter Agent

Dispatch the Hunter based on its assigned provider:

**If provider is `claude`:**
Launch a general-purpose subagent with the hunter prompt via the Agent tool. Include the scan target in the agent's task. The Hunter must use tools (Read, Glob, Grep) to examine the actual code.

**If provider is `codex` or `gemini`:**
Run the external CLI tool via the Bash tool. The command format depends on the provider:

- **Codex:** `codex exec "You must scan the following target: [scan_target]. [hunter prompt content]"`
- **Gemini:** `gemini -p "You must scan the following target: [scan_target]. [hunter prompt content]"`

For external providers, the prompt content must include the full hunter prompt. If the prompt is too long for a single shell argument, write it to a temporary file first and pass it via stdin:

- **Codex:** `cat /tmp/bug-hunt-hunter-prompt.md | codex exec --stdin`
- **Gemini:** `cat /tmp/bug-hunt-hunter-prompt.md | gemini -p -`

Use whichever approach works with the installed CLI version. If the external CLI is not installed, report the error clearly and suggest installing it.

Wait for completion and capture the full output.

## Step 2b: Check for findings

If the Hunter reported TOTAL FINDINGS: 0, skip Steps 3-4 and go directly to Step 5 with a clean report. No need to run Skeptic and Referee on zero findings.

## Step 3: Run the Skeptic Agent

Dispatch the Skeptic based on its assigned provider:

**If provider is `claude`:**
Launch a NEW general-purpose subagent with the skeptic prompt via the Agent tool. Inject the Hunter's structured bug list (BUG-IDs, files, lines, claims, evidence, severity, points). Do NOT include any narrative or methodology text outside the structured findings.

The Skeptic must independently read the code to verify each claim.

**If provider is `codex` or `gemini`:**
Run the external CLI tool via the Bash tool, passing the skeptic prompt combined with the Hunter's structured bug list. Use the same command format as Step 2 but with the skeptic prompt and the injected bug list.

Wait for completion and capture the full output.

## Step 4: Run the Referee Agent

Dispatch the Referee based on its assigned provider:

**If provider is `claude`:**
Launch a NEW general-purpose subagent with the referee prompt via the Agent tool. Inject BOTH:
- The Hunter's full bug report
- The Skeptic's full challenge report

The Referee must independently read the code to make final judgments.

**If provider is `codex` or `gemini`:**
Run the external CLI tool via the Bash tool, passing the referee prompt combined with both the Hunter's and Skeptic's reports. Use the same command format as Step 2 but with the referee prompt and both injected reports.

Wait for completion and capture the full output.

## Step 5: Present the Final Report

Display the Referee's final verified bug report to the user. Include:
1. The configuration used (which provider ran each role)
2. The summary stats
3. The confirmed bugs table (sorted by severity)
4. Low-confidence items flagged for manual review
5. A collapsed section with dismissed bugs (for transparency)

If zero bugs were confirmed, say so clearly — a clean report is a good result.
