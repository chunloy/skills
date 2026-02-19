---
name: codex
description: Use when the user explicitly mentions "codex" (e.g., "run codex", "use codex", "codex exec", "codex resume"). Do NOT invoke unless the user specifically asks for Codex.
---

# Codex Skill Guide

## Running a Task

1. **Fetch the current model list** by using the `WebFetch` tool on `https://developers.openai.com/codex/models/` with the prompt: "Extract the complete list of all available Codex model IDs (the exact strings passed to the -m flag) along with a short description of each. Group them into 'Recommended' and 'Alternative' models. Return ONLY the model IDs and descriptions."
   - Use the fetched results to populate the model options in the next step.
   - If the fetch fails, fall back to presenting these known models: `gpt-5.3-codex` (recommended), `gpt-5.2-codex`, `gpt-5.1-codex-mini`.
2. Ask the user (via `AskUserQuestion` tool) which model to run AND which reasoning effort to use (`high`, `medium`, or `low`) in a **single prompt with two questions**.
   - Present the top 3 recommended models from the fetched list as the primary options in `AskUserQuestion`, with the first one marked as recommended.
   - Mention the remaining models are available if the user selects "Other".
3. Select the sandbox mode required for the task; default to `--sandbox read-only` unless edits or network access are necessary.
4. Assemble the command with the appropriate options:
   - `-m, --model <MODEL>`
   - `--config model_reasoning_effort="<high|medium|low>"`
   - `--sandbox <read-only|workspace-write|danger-full-access>`
   - `--full-auto`
   - `-C, --cd <DIR>`
   - `--skip-git-repo-check`
5. Always use --skip-git-repo-check.
6. When continuing a previous session, use `codex exec --skip-git-repo-check resume --last` via stdin. When resuming don't use any configuration flags unless explicitly requested by the user e.g. if he species the model or the reasoning effort when requesting to resume a session. Resume syntax: `echo "your prompt here" | codex exec --skip-git-repo-check resume --last 2>/dev/null`. All flags have to be inserted between exec and resume.
7. **IMPORTANT**: By default, append `2>/dev/null` to all `codex exec` commands to suppress thinking tokens (stderr). Only show stderr if the user explicitly requests to see thinking tokens or if debugging is needed.
8. Run the command, capture stdout/stderr (filtered as appropriate), and summarize the outcome for the user.
9. **After Codex completes**, inform the user: "You can resume this Codex session at any time by saying 'codex resume' or asking me to continue with additional analysis or changes."

### Quick Reference

| Use case                       | Sandbox mode            | Key flags                                                                                        |
| ------------------------------ | ----------------------- | ------------------------------------------------------------------------------------------------ |
| Read-only review or analysis   | `read-only`             | `--sandbox read-only 2>/dev/null`                                                                |
| Apply local edits              | `workspace-write`       | `--sandbox workspace-write --full-auto 2>/dev/null`                                              |
| Permit network or broad access | `danger-full-access`    | `--sandbox danger-full-access --full-auto 2>/dev/null`                                           |
| Resume recent session          | Inherited from original | `echo "prompt" \| codex exec --skip-git-repo-check resume --last 2>/dev/null` (no flags allowed) |
| Run from another directory     | Match task needs        | `-C <DIR>` plus other flags `2>/dev/null`                                                        |

## Following Up

- After every `codex` command, immediately use `AskUserQuestion` tool to confirm next steps, collect clarifications, or decide whether to resume with `codex exec resume --last`.
- When resuming, pipe the new prompt via stdin: `echo "new prompt" | codex exec resume --last 2>/dev/null`. The resumed session automatically uses the same model, reasoning effort, and sandbox mode from the original session.
- Restate the chosen model, reasoning effort, and sandbox mode when proposing follow-up actions.

## Error Handling

- Stop and report failures whenever `codex --version` or a `codex exec` command exits non-zero; request direction before retrying.
- Before you use high-impact flags (`--full-auto`, `--sandbox danger-full-access`, `--skip-git-repo-check`) ask the user for permission using AskUserQuestion unless it was already given.
- When output includes warnings or partial results, summarize them and ask how to adjust using `AskUserQuestion` tool.
