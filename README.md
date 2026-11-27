# pwa-vti-skills

A Claude Code plugin with four skills that capture the workflow we used to take a vanilla PWA (`pwa-vti-bmock`) from "explore the repo" to a fine-tuned on-device [Cactus](https://cactuscompute.com) Needle 3 tool-calling model, and then host it in a service worker.

The skills are written from a real run, not from theory. The reference numbers below come from experiment 1 of that run.

## The workflow

```
explore repo -> tool surface + eval cases -> training data -> finetune, build, evaluate -> host in the PWA
   (1)                  (1)                      (2)                  (3)                      (4)
```

| # | Skill | Use it when |
|---|-------|-------------|
| 1 | [`ondevice-tool-surface`](skills/ondevice-tool-surface/SKILL.md) | Starting out: read the app, design the tool definitions, write dev and hold-out eval cases, build a raw-output test page, measure the untuned baseline, agree the gates. |
| 2 | [`ondevice-training-data`](skills/ondevice-training-data/SKILL.md) | Building `train.jsonl`: seeded templates, LLM augmentation, validation, dedupe, eval-overlap report, token-length check. |
| 3 | [`cactus-finetune-experiment`](skills/cactus-finetune-experiment/SKILL.md) | Training in WSL with `cactus finetune`, `cactus build`, manifest, browser testing, comparing to baseline, then shrinking with the layer-depth ladder. |
| 4 | [`needle-in-service-worker`](skills/needle-in-service-worker/SKILL.md) | Hosting the wasm model in `sw.js` instead of a page-owned Web Worker (lifecycle, caching, cold-start trade-offs). |

## Install

In Claude Code:

```
/plugin marketplace add TeaCookie/pwa-vti-skills
/plugin install pwa-vti-skills@pwa-vti-skills
```

Or load it from a local clone for a session: `claude --plugin-dir ./pwa-vti-skills`.

The skills trigger from their descriptions, so you can just say things like "help me build training data for the Needle model" or "run the next finetune experiment". They can also be invoked by name.

## Principles the skills encode

- Stage the work: one branch and one variable per experiment. A previous one-shot attempt with ~40k lines of changes got lost.
- Do not change models unless asked. Gains come from tools, data and training.
- Measure raw model output first (no guard, no confidence threshold). Add pre- and post-model layers later and compare against the raw numbers.
- Treat eval numbers with suspicion: about +/-5 points of noise on 83 cases, about +/-7.5 on 37, and training templates written by the same author as the eval prompts leak style.
- Retrained models need a content hash in the manifest URL, or the worker cache will serve the old file.

## Reference result (experiment 1)

Raw output, no guard, `tools.v5.json`, 20 layers, 4-bit local build:

| | Dev (83) strict | Hold-out (37) strict | Wrong calls on off-topic prompts | Median latency | Size |
|---|---|---|---|---|---|
| Untuned base | 58 | 25 | 9/15 dev, 6/15 hold-out | 849 ms | 35 MB |
| Fine-tuned | 80 | 37 | 0/15 both | 804 ms | 63 MB |

The hold-out score is likely inflated (template overlap with training). The open goal is about 30 MB via fewer layers.

## Status

Version 0.1.0. The skills have not yet been run against test prompts. Some figures are estimates and are marked as such inside the skills (for example the layer count that reaches ~30 MB, and the 8-10 epoch suggestion). `needle-in-service-worker` is the least proven: its service-worker lifetime and threading statements are general web-platform behavior to verify on the target browser, not measured results from this project.

## Layout

```
.claude-plugin/plugin.json        plugin manifest
.claude-plugin/marketplace.json   makes this repo installable as a marketplace
skills/<name>/SKILL.md            one folder per skill
```

## Related

- App and harness: `TeaCookie/pwa-vti-bmock` (branch `needle3-finetune-exp1`)
- Voice app using the tuned model: `TeaCookie/pwa-vti`
- Fine-tuning guide: https://cactuscompute.com/blog/finetuning-needle
