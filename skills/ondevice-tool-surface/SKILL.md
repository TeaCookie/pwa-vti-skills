---
name: "ondevice-tool-surface"
description: "Explore a repo and define the tool surface and eval cases for on-device tool calling. Use when starting to make a small local model (e.g. Cactus Needle) call an app's functions, before any fine-tuning."
---

# Tool surface and eval cases

Goal: turn an app into (1) a small, unambiguous set of tool definitions, (2) a fixed set of eval prompts with expected calls, and (3) a browser test page that scores raw model output. Everything later (training data, fine-tuning) depends on these being stable, so settle them first.

## Working rules (learned the hard way)
- Keep changes small and staged: one branch per experiment, one variable per experiment. A previous attempt that changed ~40k lines at once got lost.
- Do not change the base model unless the user says so. If they say the model is fixed, all gains must come from tools, data or training.
- Evaluate raw model output first: no app-side guard, no confidence threshold, schema parsing only. Post-model layers (guards, domain-word gates, field cleanup) are added back later and measured separately, otherwise you cannot tell what the model itself learned.

## Steps
1. **Explore the repo.** Read the README/CLAUDE.md, the entry HTML/JS, and find every user-visible action and read-only query. List state the app owns (e.g. light states, log columns, terminal history). Note which runtime loads the model (wasm, worker) and how the worker registers tools.
2. **Design tools.** Flat `{name, description, parameters}` objects. Prefer few, orthogonal tools with enum parameters over many near-duplicates. Make destructive actions (delete, clear) require explicit arguments (e.g. delete needs a time). Keep one tool per intent; avoid optional fields that tempt the model to invent values.
3. **Know the registration transform.** Local-only markers (e.g. `noop`, `localAllowed`, property-level `localPattern`) are stripped before the engine sees the tools (`forEngine`). Training data must use the stripped form, so keep one function that produces it and test parity.
4. **Write eval cases.** A dev set (~80) and a separate hold-out (~35), as JSON `{group, text, expected:[{name, arguments}]}`. Cover each tool, each argument value, compound requests (two tools), read-vs-destructive near misses ("list the red logs" vs "clear the red log"), and ~15 off-topic prompts with `expected: []`. Rule: an argument may only contain values present in the query.
5. **Build the test page.** Load a manifest (`?manifest=`, `?tools=`, `?cases=` params), run all cases with model reset between cases, show strict / lenient / names-only scores, per-group results and failures, and a "Save results (JSON)" button that records meta (tools, cases, manifest, model file/size/sha, max tokens, mode "raw"), summary, and per-case rows (calls, error, confidence, latency, suppressed, raw output).
6. **Cache hazard.** Worker asset caches may key on URL + size only. Retrained models must put a content hash in the URL (`?v=<sha256 prefix>`) in their manifest.
7. **Baseline.** Run the untuned model on dev and hold-out and save the JSONs. Expect noise of about ±5 points on 83 cases and ±7.5 on 37; do not over-read small differences.
8. **Agree gates up front** (adjust with the user): loads in browser; no regression on simple tools; ≥90% raw strict; zero destructive calls on off-topic/read prompts; size and median latency targets (e.g. ≤30 MB, ≤1.2 s).

## Pitfalls
- Writing both training templates and eval prompts yourself leaks style. Ask the user for a sealed hold-out written in their own phrasing, and record that overlap exists.
- Tool definitions have a ceiling: past a point, rewording descriptions stops helping. If dev stays well under target after a couple of rounds, move to fine-tuning.
- Without a GPU or HF access in a sandbox, tokenizer-dependent checks must be run by the user locally.