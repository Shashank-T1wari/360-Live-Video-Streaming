# Future FoV (Field-of-View) Prediction for 360° Video — D-GAN, single video/user

Predicts which tiles of a 360° video will be in a viewer's field of view a
fraction of a second into the future, using their recent head-movement
history plus the video's saliency/motion content. This notebook is the
single-video/single-user version of the pipeline (`sport` / `user50`) — the
pooled, multi-video/multi-user version lives in a separate notebook
(`FoV_prediction_pooled`).

---

## 1. What we built

### 1.1 Why D-GAN
Adapts **D-GAN** (Saxena & Cao, *"Multimodal Spatio-Temporal Prediction with
Stochastic Adversarial Networks,"* ACM TIST 2022) from taxi-demand-grid
prediction to FoV-tile prediction:

| D-GAN (original) | This notebook |
|---|---|
| Spatial grid of city zones | 20×10 tile grid over the equirectangular frame (192×192 px tiles) |
| Predicted quantity: taxi demand count | Tile occupancy (binary: does this tile overlap the FoV) |
| External factors: PoI / weather / weekday | Saliency + motion maps (genuinely per-tile, time-varying) |
| Temporal resolutions: hour / day / week | Recent-history windows: short / medium / long (0.17s / 0.5s / 1.5s) |
| Sample multiple plausible demand futures | Sample multiple plausible gaze futures |

Grid size and window alignment were verified directly against the real
uploaded files: `sport_saliency.mp4` / `sport_motion.mp4` (3840×1920, 30 fps,
1800 frames → exactly 20×10 tiles at 192 px/tile) and the tile-ID range in
`sport_user50_tile.csv`.

### 1.2 Data pipeline
- Parses `sport_user50_orientation.csv` (per-frame yaw/pitch/roll) and the
  **ragged** `sport_user50_tile.csv` (variable tile-ID list per frame) into a
  dense `(1800, 10, 20)` binary occupancy array.
- Extracts and tile-pools `sport_saliency.mp4` / `sport_motion.mp4`
  frame-by-frame (average-pools each 192×192 block to one value per tile),
  **caching to `.npy`** so re-runs are instant.
- Builds multi-resolution sliding windows: short/medium/long occupancy
  history → future occupancy target, plus saliency/motion history as
  auxiliary content input.
- Splits **chronologically** (70% train / 15% val / 15% test along the single
  60-second session) — a random shuffle would leak future frames into
  training since this is one continuous viewing session.

### 1.3 Model
Same architecture as the pooled notebook (unchanged, reused module):
- Five self-attention ConvLSTM encoder branches (tile short/medium/long +
  saliency + motion history) fusing into a shared VAE latent `(mu, logvar)`.
- Generator/decoder with two heads: future tile-occupancy probability
  (sigmoid) + auxiliary saliency/motion reconstruction.
- Discriminator on `(occupancy map, latent)` pairs, least-squares GAN loss.
- **BCE** (not L2) for the occupancy head, since the target is a 0/1 mask.
- **KL annealing** over `kl_anneal_steps=60`: early testing showed the KL
  term collapsing to ~0 within ~30 steps (posterior collapse — the model
  learns to ignore the latent). Ramping KL weight up from 0 measurably
  helped (reconstruction loss reached ~0.37 vs. plateauing at ~0.46 without
  annealing, in early testing).

### 1.4 Evaluation
Two baselines, standard in the FoV-prediction literature:
- **Persistence** — repeat the last observed occupancy frame across the
  whole future horizon.
- **Random** — score tiles randomly (establishes the floor).

Two evaluation modes:
- **Fixed threshold (probability ≥ 0.5)** — the standard sigmoid cutoff.
- **Top-K (K = mean true tile count)** — threshold-free, matches how a real
  tile-streaming system allocates a fixed tile budget; what most tile-based
  FoV papers actually report.

---

## 2. What running this notebook showed

**Note:** this uploaded copy of the notebook has not been executed (no saved
cell outputs) — the numbers below are the results the notebook's own Section
7 narrates from running this exact pipeline on `sport`/`user50` for ~5
epochs. Re-run the notebook to regenerate them and confirm.

- **Fixed 0.5 threshold**: predicted probabilities never crossed 0.5 (max ≈
  0.46) after ~5 epochs, so precision/recall/F1 read as 0. This is a
  **calibration** problem, not a "model learned nothing" problem — mean
  predicted probability (0.161) matched the true occupancy rate (0.162)
  almost exactly, i.e. the model learned the correct *base rate* but hadn't
  sharpened its confidence enough to cross a fixed cutoff.
- **Top-K evaluation** (the fairer comparison): model scored **F1 ≈ 0.67,
  IoU ≈ 0.51**, well above **random** (F1 ≈ 0.16, IoU ≈ 0.09) — confirming
  real, non-trivial spatial structure was learned.
- The model did **not** beat the **persistence baseline** (F1 ≈ 0.93, IoU ≈
  0.88) within this training budget. This particular clip has slow, smooth
  head motion, so "assume no movement" is an unusually strong baseline —
  consistent with patterns reported in published FoV-prediction work at
  sub-second horizons on slow-motion content.
- Training loss (occupancy BCE) dropped from ~0.54 → ~0.22–0.28 over ~370
  steps (~5 epochs), noisy and still plateauing — not yet converged.

**Bottom line:** the full pipeline runs correctly end-to-end on real data,
and the model learns genuine (better-than-random) spatial structure — but at
this training budget (single video/user, ~5 epochs) it doesn't yet beat the
trivial persistence baseline. That's an honest, expected checkpoint result
for a single-session model, not a final result.

---

## 3. Changes / next steps

In the order the notebook's own Section 10 suggests, roughly by expected
impact:

1. **Train for more epochs.** Occupancy loss was still improving at epoch 5
   and this config trains in a few minutes on CPU — `epochs=15–20` is the
   first, cheapest thing to try.
2. **Pool more videos/users.** This notebook currently pools only *windows
   within one session* (since only one video/user file set was provided).
   Point `build_fov_dataset` at multiple `(video, user)` pairs and combine
   their windows (chronological split within each session, then concatenate)
   to exercise the actual pooled design and expose the model to more diverse
   head-motion patterns. (This is exactly what the separate
   `FoV_prediction_pooled` notebook does.)
3. **Calibrate the decision threshold** on the validation set (e.g. pick the
   threshold that maximizes F1, or just report top-K) instead of a fixed
   0.5 — Section 7 shows a fixed threshold hides real learned signal.
4. **Add a continuous yaw/pitch/roll regression head**, alongside or instead
   of tile occupancy, using the calibrated orientation columns directly —
   more precise than a discretized grid, at the cost of the direct
   "which tiles to stream" interpretation.
5. **Feed future (not just historical) saliency/motion.** Unlike future
   viewer orientation, the video content is known in advance in a real
   streaming pipeline (it's pre-recorded) — using *upcoming* frames'
   saliency/motion as an additional input is a realistic enhancement this
   notebook doesn't yet use.

---

## 4. How to reproduce / where to look

- Data pipeline: `parse_orientation_csv`, `parse_tile_csv`,
  `extract_pooled_video`, `build_multi_resolution_fov_windows`,
  `build_fov_dataset` (Section 2).
- Model: `FoVDGAN` class — encoder (`build_fov_correlation_network`),
  generator (`build_fov_generator`), discriminator (`build_discriminator`)
  (Section 3).
- Training loop + loss curves: Sections 5–6.
- Evaluation (fixed-threshold + Top-K, vs. persistence/random baselines) and
  the honest discussion of what the numbers mean: Section 7.
- F1-vs-horizon curve and qualitative predicted-vs-actual heatmaps: Sections
  7–8.
- Multiple plausible futures (stochastic latent sampling) + union-of-samples
  recall: Section 9.
- Limitations and prioritized next steps: Section 10 (mirrored in Section 3
  of this README).
