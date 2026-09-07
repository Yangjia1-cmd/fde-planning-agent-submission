# Decisions

---

## What I deliberately did not do, and why

`execute_step`'s two helpers (`_execute_text_step`, `_execute_tool_step`) call
`chat`/`chat_with_tools` without the try/except guard that `write_plan` and
`revise_plan` use, so a raised exception from the executor or observer would
propagate all the way up and crash the run instead of degrading into an error
step. I left it alone because `execute_step` is marked provided and out of
scope for Part A, but if I owned that file I'd wrap it the same defensively.
I kept `revise_plan`'s renumbering fallback (start from `remaining[0].n` when
`done` is empty) minimal rather than special-casing it further, since that
path is only exercised by an isolated unit test and never occurs in the real
`run_planning_agent` loop, where `revise_plan` is only ever called after at
least one step has run. I didn't add retry/backoff logic anywhere, since
grading runs in replay mode and that's a live-mode concern outside this
exercise. I also kept `queue.pop(0)` on a plain list in `run_planning_agent`
instead of a `deque` — it's O(n) per pop, but plans cap at 5-6 steps, so
optimizing it would trade readability for nothing measurable.

## How I would know this works

Replay mode is deterministic, so the floor is a golden-trace test: pin a
handful of goals, run them against `fixtures/llm.json`, and diff the full
`PlanRun` (not just the final answer) against a checked-in baseline on every
commit — any diff is a real regression, not a fluke. That catches code bugs
but not model or prompt drift, so in live mode I'd track three numbers over a
rolling window: the parse-failure rate (times `_parse_plan` returned `[]` and
we fell back to a degraded plan or an echoed remaining list), which I'd treat
as bad above roughly 2-3%, since a fallback is a silent quality loss even
though the run doesn't crash; the revision rate (fraction of runs that hit a
"surprise" observation), where a sudden jump usually means the observer
started crying "surprise" on ordinary results rather than the world getting
more surprising; and the coordinators' own approve-gate rejection rate, since
that's the ground truth on plan quality that no automated eval can fake. On
top of those counters I'd sample a fixed weekly batch of live final answers
for a human (or an LLM-judge against a short rubric: cites sources, respects
the stated constraint, addresses anything flagged as a surprise) to score
pass/fail, because free-text answer quality is exactly what unit tests can't
see.

---

**AI assistance:** I used AI assistance for the entire implementation because I'm a designer, not an engineer, so I relied on it to write and verify all three functions rather than writing or debugging the Python myself. The only bug that came up was a visual styling gap in the revision panel (a tool-hint badge losing its CSS in the "what changed" view), which I fixed by testing the app in the browser and catching it there rather than from the automated tests.

**Time spent:** 60 minutes.
