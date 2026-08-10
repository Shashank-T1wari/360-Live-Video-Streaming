# Pooled FoV (Field-of-View) Prediction — 360° Video Dataset

Predicting a viewer's future tile-occupancy heatmap (which tiles of a 360° video
will be in their field of view) from their recent head-movement history plus
video saliency/motion content, pooled across multiple videos and viewers.

---

## 1. What we built

### 1.1 Data pipeline (new in this notebook)
Generalized a single-video/single-user FoV notebook to the **full dataset**,
using the folder structure:

```
360dataset/
  content/
    saliency/videoname_saliency.mp4
    motion/videoname_motion.mp4
  sensory/
    orientation/videoname_userXX_orientation.csv
    tile/videoname_userXX_tile.csv
```

Key additions:
- **`discover_trajectories`** — auto-finds every (video, user) pair on disk
  that has a complete matching set of files, instead of assuming one fixed
  pair.
- **Mixed-resolution handling** — each video's native tile grid is computed
  from its own resolution, then every video is resized to one shared
  `grid_h x grid_w` grid so a single model can train across videos of
  different native sizes.
- **Trajectory-level train/val/test split (70/15/15)** — whole (video, user)
  pairs go entirely into one split, never split within a trajectory, so the
  model is evaluated on video/viewer combinations it has never seen. This is
  the correct test for a pooled/general predictor (as opposed to splitting
  chronologically within one clip).
- **Per-video caching** — saliency/motion extraction is expensive, so it's
  computed once per video and reused across every user who watched that
  video.

### 1.2 Model
Same D-GAN-style architecture as the earlier single-trajectory notebook
(unchanged, already validated there):
- Three ConvLSTM encoder branches over **short / medium / long recent-history
  windows** (15 / 30 / 60 frames ≈ 0.5 s / 1.0 s / 2.0 s at 30 fps) of tile
  occupancy, plus two more branches over saliency and motion history.
- Self-attention (SAGAN-style) inside each encoder branch.
- Branches fuse into a shared latent `(mu, logvar)` — a VAE bottleneck.
- Generator decodes the sampled latent into (a) a future tile-occupancy
  probability map, `predict_horizon=15` frames ahead, and (b) an auxiliary
  reconstruction of the saliency/motion history (keeps the latent
  content-rich).
- A discriminator adversarially pushes generated occupancy maps to look
  realistic (least-squares GAN loss).
- Loss = BCE (occupancy reconstruction) + MSE (content reconstruction) +
  annealed KL + adversarial term.

### 1.3 Evaluation
- **Primary metric: Top-K F1 / IoU**, threshold-free — K set to the average
  true number of occupied tiles, standard for tile-based FoV work.
- **Secondary metric: fixed 0.5-threshold** precision/recall/F1/IoU/accuracy,
  included for transparency (known to read low even when the model has
  learned real structure, due to probability calibration/sharpness).
- **Baselines**: persistence (repeat the last observed frame for the whole
  horizon) and random scoring.
- Evaluated on **entirely unseen (video, user) trajectories** — a genuine
  generalization test, not just a held-out time slice.

---

## 2. What we ran, and what came out

**Run config**: 6 videos, ≤5 users/video → 30 trajectories discovered → 21
train / 4 val / 5 test. 5 epochs, CPU only (no GPU detected).

**Runtime**: ~2.2–3 hours *per epoch* (~13–14 hours total) — CPU-bound, no
GPU available in this environment.

**Headline result — Top-K F1, mean over 15-frame horizon:**

| | Model | Persistence baseline | Random |
|---|---|---|---|
| F1 | **0.625** | **0.886** | 0.171 |
| IoU | 0.455 | 0.796 | — |

**Interpretation:**
- The model clearly beats random (0.625 vs 0.171), so it has learned a real,
  above-chance spatial prior — roughly *where* the viewer's gaze cluster is.
- It **loses to the persistence baseline at every horizon**, including +1
  frame. "Assume the future looks like the last observed frame" currently
  outperforms the trained model. Gaze in 360° video is strongly
  autocorrelated over sub-2-second windows, so this is a real, hard baseline
  to beat — but it means the model has not yet demonstrated it captures
  head-movement *dynamics*, only coarse position.
- One encouraging pattern: the gap to persistence **narrows with horizon**
  (persistence F1 falls 0.926→0.843 from +1 to +15 frames, while model F1
  rises 0.578→0.643) — i.e., persistence degrades faster than the model does
  as you look further ahead. This suggests the model may become relatively
  more useful at longer horizons, though this wasn't confirmed with a
  dedicated plot (see Section 3 below).
- Qualitative predictions are a **soft, blurry blob** centered near the
  viewer's current position, and this blob barely changes shape from +1 to
  +15 frames — consistent with "predicting position, not motion."
- Sampling the latent multiple times (stochastic futures) produced very
  similar high-confidence outputs each time — the VAE latent is not yet
  representing meaningfully different plausible futures.
- Train BCE loss fell steadily (0.33 → 0.25) while **val BCE flattened/rose
  after epoch 2** (0.359 → 0.345 → 0.366 → 0.348 → 0.382) — the model is
  overfitting to the pooled training trajectories well before epoch 5.

**Bottom line:** the data pipeline (discovery, mixed-resolution handling,
trajectory split, caching) worked correctly end-to-end on the real dataset —
the earlier "untested orchestration code" concern turned out fine. The model
itself, in this configuration, is currently **weaker than doing nothing**
for short-horizon FoV prediction. That's the finding to act on.

---

## 3. Changes / improvements to make next

Roughly in priority order:

1. **Make beating the persistence baseline the actual target.**
   Nothing in the current loss rewards the model for outperforming
   persistence. Try predicting the *delta* from the last observed frame
   instead of absolute occupancy, or add a loss/metric term that only
   rewards improvement over persistence. This reframes training toward the
   genuinely hard part — motion — instead of the easy part — position.

2. **Address overfitting before scaling up.**
   Val loss already flattens by epoch 2 with only 5 epochs run. More epochs
   alone won't help. Try: more dropout / weight decay, a smaller latent
   dim / fewer ConvLSTM filters (model is already small — 93K / 1.8M / 19K
   params for encoder/generator/discriminator — so overfitting is more
   about too little trajectory diversity than too much capacity), and early
   stopping on val loss (current run's best checkpoint is ~epoch 2, not 5).

3. **Get GPU compute.**
   ~3 hrs/epoch on CPU is the main blocker to iterating at all. A modest GPU
   would likely bring this to minutes/epoch, enabling the 20–50+ epochs a
   GAN-style model typically needs to stabilize, plus real hyperparameter
   sweeps.

4. **Finish running the notebook's own diagnostic cells.**
   The F1/IoU-vs-horizon plot and the fixed-threshold
   precision/recall/IoU/accuracy + predicted-probability-range cells were
   written but never executed in this run. Running them will quantify the
   "blurriness"/calibration issue seen qualitatively, and confirm/deny the
   narrowing-gap-with-horizon pattern noted above.

5. **Break down test performance by generalization type.**
   Split held-out results into "video seen in training (different user)" vs.
   "video never seen at all." With only 6 videos / 21 train trajectories,
   part of the current 0.625 F1 could be propped up by video content the
   model already saw, even though the *viewer* is new — this materially
   changes how to interpret "generalization to unseen trajectories."

6. **Scale up data once 1–5 are addressed.**
   `max_videos=6, max_users_per_video=5` were CPU-feasibility caps, not
   dataset limits (10 videos were discovered on disk). 21 training
   trajectories is thin for a pooled/general model. Raise these caps after
   the baseline-aware loss, regularization, and GPU compute are in place —
   not before, since more data won't fix the current beats-persistence
   problem on its own.

7. **Sanity-check against a simpler model.**
   Given persistence alone already scores F1 = 0.886, try something between
   "persistence" and the full VAE-GAN — e.g. a plain ConvLSTM regressor
   trained with BCE only (no VAE/GAN terms), or a simple linear
   extrapolation of the gaze centroid. If a much simpler model closes most
   of the gap to persistence, that indicates the VAE/GAN machinery should be
   added back incrementally once the fundamentals work, rather than trained
   all together from scratch.

8. **Longer-term extensions** (lower priority, from the notebook's own
   "next steps"):
   - Longer prediction horizons.
   - Continuous yaw-pitch-roll head-pose prediction instead of discrete tile
     occupancy.

---

## 4. How to reproduce / where to look

- Data pipeline: `discover_trajectories`, `_process_trajectory`,
  `build_pooled_dataset` (Section 2 of the notebook).
- Model: `FoVDGAN` class — encoder (`build_fov_correlation_network`),
  generator (`build_fov_generator`), discriminator (`build_discriminator`)
  (Section 4).
- Training loop + train/val curves: Section 5.
- Evaluation (Top-K + fixed-threshold metrics vs. persistence/random
  baselines): Section 6.
- Qualitative visualizations (prediction vs. ground truth, stochastic
  futures): Sections 7–8.
- Known open items / next steps: Section 9 (and Section 3 of this README).
