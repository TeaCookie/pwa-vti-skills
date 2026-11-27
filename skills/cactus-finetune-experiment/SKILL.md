---
name: "cactus-finetune-experiment"
description: "Run and evaluate a Cactus finetune experiment in WSL: train, build .cact, make manifest, browser-test raw output, compare to baseline, then shrink via layer depth. Use when the user trains or evaluates Needle models."
---

# Finetune, build, evaluate

Prereqs: tools and eval cases (ondevice-tool-surface) and `train.jsonl` (ondevice-training-data). Training runs in WSL on the user's GPU (the Windows runtime is CPU only). Give the user copy-paste WSL commands to run in the project directory; you cannot train in a sandbox. Put each experiment on its own branch with a README runbook.

## Commands (experiment 1 shape)
```
cactus finetune finetune/data/train.jsonl --epochs 20 --batch-size 16 --lr 1e-4 --lora-rank 32 --lora-alpha 64 --max-len 2048 --val-split 0.1 --seed 42 --checkpoint-dir finetune/checkpoints --out finetune/checkpoints/NAME_adapter.safetensors 2>&1 | tee finetune/results/NAME_train.log
cactus build --lora finetune/checkpoints/NAME_adapter.safetensors --out finetune/models/NAME.cact
node finetune/make_manifest.mjs finetune/models/NAME.cact   # writes needle/assets.NAME.json with ?v=<sha12>
python3 -m http.server 8000   # open test.html?tools=...&cases=...&manifest=assets.NAME.json
```
Then Load model, Run all cases, Save results (JSON) for dev and hold-out, with the new manifest and with no manifest (untuned baseline).

## Expectations
- A 20-layer local build is about 63 MB (4-bit local quantization). `cactus build` drops the confidence head, so confidence is None; do not use a confidence sweep on tuned models.
- Training on a ~6 GB card took 3+ hours for 20 epochs at batch 16 on ~2000 examples. Validation loss bottomed out around epoch 8 and drifted up while training loss hit ~0, so 8-10 epochs is enough and about a third of the time.
- The `accuracy N/M held-out calls exact` line scores the quantized weights that ship.

## Analyzing results
Compute from the saved JSONs: strict/lenient/names scores, wrong calls on off-topic prompts, error/truncation counts, median and p90 latency, model size. Compare to the untuned baseline per group, list every failure with expected vs got, and check the gates agreed earlier. Reference result: untuned 58/83 dev and 25/37 hold-out; exp1 80/83 and 37/37, 0/15 wrong off-topic calls, 63 MB, median 804 ms.
- Say plainly what the numbers do not show: hold-out contamination, noise (+/-5 on 83, +/-7.5 on 37), no run-to-run variance measured.
- Read the raw `reasoning` strings. They reveal whether the model is deriving arguments or pattern-matching.

## Next experiments (one variable each)
1. Retrain at fewer epochs (8-10), then build the depth ladder with `cactus build --layers N` (e.g. 20, 14, 10, 8) toward ~30 MB (about 8-9 layers at 4 bits is an estimate). Unknown whether shallow slices trained locally keep accuracy; the ladder measures it. Consider training at the target depth if slices degrade.
2. A sealed hold-out written by the user, and a variance check (repeat training with another seed).
3. Only after a good raw threshold: add pre-model layers (domain gate, field cleanup) and re-measure against the raw numbers.

## Practicalities
- The same cactus wasm runtime as the user's pwa-vti works; a tuned .cact loads through the same call sequence.
- Large files: .cact and .safetensors via git LFS; push the branch with data, logs, results and manifest so the next session can analyze them.
- Do not start the next experiment before the user confirms; give exact commands and a small compare script for result JSONs.