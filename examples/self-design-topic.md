# ralph-discussion Skill Design

## Question
How should a design discussion skill work that alternates between Codex and Gemini to cross-validate architectural decisions?

## Context
- Part of the ralph-super-simple toolkit for Claude Code autonomous workflows
- Uses `codex exec` and `gemini -p` CLI tools for non-interactive execution
- Goal: leverage different LLM perspectives to refine designs through structured debate
- Output: a single synthesized markdown document with final decisions

## Current Proposal
- 5 rounds: Codex(architect) → Gemini(critic) → Codex(refine) → Gemini(review) → Codex(synthesize)
- Each round receives all previous outputs as accumulated context
- run.sh automates the full cycle
- Import skill creates `.discussion/` structure with topic.md and run.sh

## Design Questions
1. Is 5 rounds optimal? Should it be configurable, and what's the point of diminishing returns?
2. Should roles be fixed (architect/critic alternation) or customizable per discussion?
3. Should the synthesis round use a third tool (e.g., Claude) for neutral arbitration?
4. How should output quality be handled — codex/gemini may include noise (tool calls, ANSI codes)?
5. Should there be convergence detection (early exit if positions stabilize)?
6. Is the accumulated context approach scalable, or should later rounds receive summaries instead?
