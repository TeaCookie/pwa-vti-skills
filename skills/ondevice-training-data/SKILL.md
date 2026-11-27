---
name: "ondevice-training-data"
description: "Build, validate and de-contaminate JSONL training data for fine-tuning a small tool-calling model (Cactus Needle). Use after the tool surface and eval cases exist and before running training."
---

# Training data pipeline

Format, one JSON object per line: `{query, tools, answers, reasoning}`.
- `tools`: exactly what the engine sees (flat, local markers stripped). Put it in one file (`tools.train.json`) generated from the app's tools and test that it is current.
- `answers`: list of calls `{name, arguments}`; `[]` for off-topic prompts.
- `reasoning`: one short line that derives each argument from the query. Arguments may only contain values present in the query. This is what teaches grounding and abstention.

## Pipeline
1. **Seed with templates (deterministic).** A seeded RNG generates varied phrasings for every tool and argument value: toggles on/off, create vs update, optional times in several forms (HH:MM, 7:45 am, 7 am), delete, clear, read, export, status, time, terminal, and compounds (never the same tool or light twice). Add negatives at roughly 15-20%: far off-topic prompts plus near misses that share keywords ("set a timer", "turn the brightness up"). Same seed must give the same file.
2. **Augment with an LLM** (`cactus generate-data --augment seed.jsonl --num-samples ~800 --batch-size 25 --workers 8`). The provider key lives in the user's WSL environment; never ask for it. Treat the output as untrusted.
3. **Check and canonicalize** with a script: reject examples with no/long query, non-array or >3 answers, missing reasoning, calls that fail the same parser the app uses, fields from the wrong column, missing state field, times not present in the query (normalize 12-hour forms first), delete without a time. Replace `tools` with the canonical set, dedupe on normalized query, drop anything that matches an eval prompt (Jaccard >= 0.8 on words), and print drop reasons and the mix (negatives share, per-tool counts).
4. **Report eval overlap** per group between training and the dev/hold-out sets. If overlap is high, say so; it inflates scores.
5. **Check token length** with the trainer's own tokenizer and renderer: examples longer than `--max-len` are silently truncated, and training on truncated calls teaches the model to stop early. Tool JSON alone can be ~750-990 tokens; use `--max-len 2048` unless the check says otherwise. Fail if any example exceeds the cap.

## Notes
- Validation split is a random 10% of the same file, so near-duplicates leak into it; the val loss is optimistic.
- Do not commit generated seed/augmented files that can be regenerated; commit the final `train.jsonl` only when the user wants the experiment reproducible. Track large binaries with git LFS.
- Coverage gaps show up as misses on unseen phrasings (e.g. "what does the clock say", a compound with mixed on/off). Fix them by adding templates for the missed intent, not by copying the eval prompt.