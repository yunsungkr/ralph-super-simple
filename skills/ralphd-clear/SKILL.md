---
name: ralphd-clear
description: Delete .discussion/ directory. Use for discussion clear / reset.
---

# ralphd-clear

Deletes the `.discussion/` directory to prepare a clean state for the next discussion.

## Delete Target

| Target | Action |
|--------|--------|
| `.discussion/` | **Delete** — All discussion artifacts and round outputs |

## Commands

```bash
rm -rf .discussion/
```

## Execution Steps

1. Check if `.discussion/` directory exists
2. Run `rm -rf .discussion/`
3. Output: "`.discussion/` deleted. Re-initialize with `ralphd import`."
