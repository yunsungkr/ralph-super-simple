---
name: ralphss-clear
description: Delete .ralphss/ directory. Use for ralphss clear / reset.
---

# ralphss-clear

Deletes the `.ralphss/` directory to prepare a clean state for the next import.

## Delete Target

| Target | Action |
|--------|--------|
| `.ralphss/` | **Delete** — All import artifacts and runtime outputs |

## Commands

```bash
rm -rf .ralphss/
```

## Execution Steps

1. Check if `.ralphss/` directory exists
2. Run `rm -rf .ralphss/`
3. Output: "`.ralphss/` deleted. Re-initialize with `ralphss import`."
