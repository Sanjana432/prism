# PRISM · Minute Change Lab

An executable research workbench for controlled **small** document-rendering changes. It includes image generation, native-resolution comparison, blind annotation, human calibration, grouped splitting, Qwen inference, QLoRA training, and held-out evaluation.

**Status:** the browser workbench and CPU pipeline are implemented and tested. Generated datasets are development/pilot material with **empty human scores**. No Qwen training run, human study, or high-accuracy claim has been completed. GPU execution uses the included free-Colab notebook. The research target remains approximately 85 approved scans + 15 authored originals, 10 variants each, and four independent annotators.

## Start locally

Python 3.10–3.12 and Node 18+ for JavaScript tests:

```powershell
python -m pip install -r requirements.txt
python -m prism serve --port 8847
```

Open `http://127.0.0.1:8847`. The published static site uses the same browser-local application. Uploaded images are processed in the browser; annotations and generated releases persist in IndexedDB. Export a workspace backup before clearing browser storage. Reviewers export JSONL files to combine their annotations; this is not a shared multiuser database.

## What is ready

- Six controlled edit families: font family, font size, font colour, inpainting colour, horizontal position and line spacing.
- Lossless generation at native resolution, 4× supersampled text, one region changed per diagnostic release, exact edit metadata, and unchanged controls.
- Browser comparison, overlay, heatmap, full-resolution zoom, region crops, direct image upload, PNG/JSONL ZIP export.
- Blind task/image export and twelve independent scores, no prefilled labels, reviewer IDs, notes and detection judgements.
- Four-rater consensus, mean/std, train-only isotonic severity calibration. Font-family labels are categorical, not ordinal.
- 70/15/15 grouping by original and exact source hash; validation-only prompt selection; no synthetic test ground truth.
- Strict twelve-score JSON inference, one retry, raw invalid output logs; deterministic category/overall scores and release ranking.
- MAE, RMSE, PLCC, SROCC, original-level bootstrap, best-variant selection, detection controls and conservative sensitivity thresholds.
- Qwen2.5-VL-3B / NF4 QLoRA with rank 16, alpha 32, batch 1, gradient accumulation 8, cosine LR, 2–3 epochs, assistant-only loss and best validation checkpoint.

## Generate your images

### Immediate, editable multilingual templates

```powershell
python -m prism generate --output data/generated --originals 20 --variants 25
python -m prism audit data/generated/pairs.jsonl --root data/generated
```

Each original yields one unchanged control plus four settings for each of six edit types. `--variants 10` makes a smaller main-study-shaped set; it does **not** turn these repetitive authored fixtures into a representative research corpus. Use diverse source documents for the final study.

Add `--mixed` to generate 1–3-knob combinations, e.g. `--originals 100 --variants 10 --mixed`. Mixed edit dictionaries are recorded exactly and have no invented aggregate severity. They require direct human labels; single-knob calibration is never added together to fabricate mixed-artifact truth.

Browser generation creates the same kinds of controls with its own documented canvas renderer; byte outputs and font rasterization can vary by OS/browser. Never mix renderer versions without recording their provenance. A reconstructed scan region is a new candidate baseline and must be reviewed before study use.

### Real scans, OCR and translation

Colab dependencies include Tesseract, OpenCV and optional local Marian EN→FR/ES translation. Translation downloads a public model and runs in your runtime; it does not transmit document text to an external translation API.

```bash
python scripts/prepare_scan.py ocr input.png data/draft.json
python scripts/prepare_scan.py translate data/draft.json data/translated.json
# Review OCR, boxes, translated text and fonts in translated.json.
# Mark document approved:true and each region approved:true only after review.
python scripts/prepare_scan.py generate data/translated.json data/scan_001
```

The generator inpaints only a foreground text mask, restores all unmasked original pixels, fits translated paragraphs, and rejects invalid/overlapping regions and baseline overflow. It preserves a clean background image. OpenCV inpainting is deterministic but can still produce visible texture defects; inspect the reconstructed baseline at native resolution. A diffusion model is deliberately unnecessary for this small-change task.

On Windows, `scripts/ocr_windows.ps1` uses installed Windows OCR. `scripts/build_scan_pilot.py` demonstrates generation from existing source scans with saved OCR. Those pilot samples are same-language English reconstructions and explicitly **unreviewed**. They are not approved French/Spanish references.

The supplied screenshot examples are included for visual inspection. Their exact baseline provenance and JPEG history are unverified. The 600 supplied metadata records are preserved; their original `real_samples` assets were absent, and all 7,200 metric values were null.

## Full research workflow

```bash
# 1. Validate assets, then split by original before learning anything.
python -m prism audit data/generated/pairs.jsonl --root data/generated
python -m prism split data/generated/pairs.jsonl --output data/split.jsonl

# 2. Test + validation + 300 training-calibration pairs, four humans each.
python -m prism tasks data/split.jsonl --root data/generated --output data/annotation
# Give only data/annotation/blinded to reviewers, not private_assignment_key.json.
# Import blinded/tasks.jsonl and its image folder in the site's Annotation studio.

# 3. Combine reviewer exports; use the private key to recover original pair IDs.
python -m prism consensus data/all_annotations.jsonl --key data/annotation/private_assignment_key.json --output data/consensus.jsonl
python -m prism calibrate data/split.jsonl data/consensus.jsonl --labelled-output data/labelled.jsonl

# 4. GPU steps: run in Colab. Test baseline is read-only; do not tune from it.
python -m prism infer data/split.jsonl --root data/generated --output outputs/baseline.jsonl --split test --probe
python scripts/select_prompt.py data/split.jsonl data/consensus.jsonl --root data/generated
# Read selection.json and pass its frozen prompt/resolution below.
python -m prism train data/labelled.jsonl --root data/generated --prompt P2 --resolution 1280
python -m prism infer data/split.jsonl --root data/generated --adapter outputs/qlora/best --output outputs/finetuned.jsonl --probe

# 5. Actual human agreement, not agreement with rendering formulas.
python -m prism evaluate data/split.jsonl data/consensus.jsonl outputs/baseline.jsonl --output reports/baseline.json
python -m prism evaluate data/split.jsonl data/consensus.jsonl outputs/finetuned.jsonl --output reports/finetuned.json
```

The P2/1280 arguments above are examples, not a claimed selected optimum. The notebook reads the actual selection file. All computations can be resumed from saved manifests and exports. Save outputs to Google Drive between free Colab sessions.

## Interpreting output

`prism.vision.compare` sees a **clean localized baseline + release** and returns physical evidence. `prism.vlm` sees **Original first + Release second** for the twelve quality predictions. Its optional separate detection probe sees baseline + release. Generator severity, target scores and the known artifact ROI never enter the VLM quality prompt. P4 uses a fixed central crop selected without defect metadata.

Pixel change is not the same as perceptual detection. A one-level RGB difference may be measurable but invisible. JPEG noise and page misregistration can overwhelm a minute-change signal. The tool refuses to silently resize mismatched pages. Use lossless, aligned pairs and negative controls. Font family cannot be reliably named from simple pixel geometry.

The twelve canonical keys match the supplied metadata. Scores use [0,1], higher is better. All category weights are 1/3; each category contains four metrics. Semantic quality remains unscored until humans or an actual VLM score it. Null correlations mean mathematically undefined, not zero performance.

## Verification

```bash
python -m unittest discover -s tests -v
node --test tests/engine.test.mjs
python scripts/verify_delivery.py
```

See `reports/verification.json` for executed checks and `docs/reference-audit.md` for the attachment audit. GPU model code is syntax-checked but cannot be claimed runtime-tested until run on a CUDA machine with real labels. The notebook makes those requirements explicit and refuses to train on empty labels.

## Key files

| Location | Purpose |
|---|---|
| `site/dist/` | Browser app, demo assets, downloadable toolkit |
| `prism/generation.py` | Deterministic authored release generator |
| `prism/localization.py` | OCR, draft translation, scan rendering |
| `prism/annotation.py` | Consensus and human calibration |
| `prism/dataset.py` | Validation, split and blind tasks |
| `prism/vlm.py`, `prism/training.py` | Real optional GPU inference/training |
| `prism/evaluation.py` | Human agreement and minute-change audit |
| `PRISM_Colab.ipynb` | Colab setup and end-to-end execution |
| `configs/study.json` | Frozen research targets and safeguards |

Model implementation follows [Qwen2.5-VL documentation](https://huggingface.co/docs/transformers/v4.50.0/model_doc/qwen2_5_vl) and [PEFT quantization guidance](https://huggingface.co/docs/peft/v0.18.0/developer_guides/quantization). See the downloaded dependencies' own licences and model cards before redistribution. No commercial font files are bundled; system fonts are used at render time.

