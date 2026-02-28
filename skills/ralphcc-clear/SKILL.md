---
name: ralphcc-clear
description: Delete .ralph/ directory and temp files. Use for ralph clear / reset.
---

# ralphcc-clear

Deletes the `.ralph/` directory and import temp files to prepare a clean state for the next import.

## Preserve/Delete Criteria

| Target | Location | Action | Reason |
|--------|----------|--------|--------|
| `.ralphrc` | Project root | **Preserve** | Project settings (ALLOWED_TOOLS, timeout). Immutable across runs |
| `.ralph/` | Project root | **Delete** | All import artifacts and runtime outputs |
| `.ralph_conversion_*` | Project root | **Delete** | ralph_import.sh temp files (auto-cleaned on normal exit, may remain on interruption) |

### .ralph/ Internal Details

**Import artifacts** (created by ralphcc-import):
- Single Phase: `PROMPT.md`, `fix_plan.md`, `AGENT.md`, `specs/`
- Multi Phase additions: `MASTER_PLAN.md`, `phase_runner.sh`, `phases/` (per-phase `PROMPT.md`, `fix_plan.md`, `verify.sh`)

**Runtime outputs** (created by ralph_loop.sh):
- `logs/`, `status.json`, `progress.json`, `live.log`
- `.circuit_breaker_*`, `.call_count`, `.last_reset`
- `.exit_signals`, `.ralph_session*`, `.loop_start_sha`

### Import Temp Files (Project Root)

- `.ralph_conversion_prompt.md` — prompt sent to Claude during import
- `.ralph_conversion_output.json` — Claude response JSON
- `.ralph_conversion_output.json.err` — stderr output

## Commands

```bash
rm -rf .ralph/
rm -f .ralph_conversion_prompt.md .ralph_conversion_output.json .ralph_conversion_output.json.err
```

## Execution Steps

1. Check if `.ralph/` directory exists
2. Run `rm -rf .ralph/`
3. Run `rm -f .ralph_conversion_*` (clean up remaining temp files)
4. Output: "`.ralph/` and temp files deleted. `.ralphrc` preserved. Re-initialize with `ralphcc import`."
