# 🚂 Embedded AI Railway-Sleeper Counter — multimodal vision on the edge

> **Showcase repository.** This system was developed during a paid engineering mission for **AFC Recycling** (industrial recycling of railway sleepers). Source code, trained models and the dataset are the client's property and are **not published**. This page describes the problem, the architecture and the results.

**Goal:** automatically count the railway sleepers (*traverses*) fed into a mobile crusher by an excavator, in real time, with a camera and neural networks running **entirely on embedded hardware** — no cloud, no server.

---

## The challenge

| Constraint | Why it is hard |
|---|---|
| Outdoor, mobile site | The crusher moves from site to site across France: new backgrounds, new lighting every time |
| Camera points **upwards** | The sky is the background → violent lighting changes (overcast ↔ direct sun, hard shadows, glare) |
| Dust & vibrations | Degraded images, need for a robust mounting |
| Intermittent flow | Sleepers arrive in the excavator's bucket, 0 to 3 at a time, often partially hidden |
| Edge only | Raspberry Pi + smart camera (Myriad X VPU) — every millisecond counts |

## Architecture

```
            ┌─────────────┐
 RGB  ────► │ Expert CNN  │──┐
            └─────────────┘  │     ┌───────────────┐     ┌──────────────────────┐
 Mono ────► │ Expert CNN  │──┼───► │ "Chief" meta- │ ──► │ Temporal counting    │ ──► LCD display
            └─────────────┘  │     │ network       │     │ logic (voting +      │     + remote
 Depth ───► │ Expert CNN  │──┘     │ (fusion)      │     │ hysteresis)          │     monitoring
            └─────────────┘        └───────────────┘     └──────────────────────┘
      stereo/depth camera           Raspberry Pi + on-camera VPU (OpenVINO .blob)
```

1. **Data acquisition tool** — custom software streaming RGB, mono and depth simultaneously, saving and pre-labelling images on site.
2. **Data cleaning & auditing tool** — review and fix mislabelled images; class balancing and controlled data augmentation (≈ 13 800 training images, ~4 500 per class).
3. **Three expert CNNs** (one per modality) + a **"Chief" meta-network** that fuses their probability vectors.
4. **Edge deployment** — Keras `.h5` → ONNX → OpenVINO `.blob` running on the camera's VPU, multi-threaded pipeline on the Raspberry Pi (~22 FPS).
5. **Counting logic** — asymmetric confidence thresholds, majority vote over a sliding window of 10 frames, hysteresis state machine so a slow sleeper is counted only once.
6. **Remote operations** — 4G link, VNC + VS Code remote for diagnostics, re-training and over-the-air model updates from Lille.

## Results

**Offline evaluation (test set, v2):**

| Model | Accuracy |
|---|---|
| RGB expert | 95.99 % |
| Mono expert | 96.50 % |
| Depth expert | 93.04 % |
| **Chief (fusion of the 3)** | **97.51 %** |

The Chief corrects **48.6 %** of the RGB expert's errors and **78.8 %** of the depth expert's errors.

**On-site validation (real counting vs. system):** iterative calibration of the temporal logic → **71 % → 77 % → 90.9 %** counting accuracy, with a deliberately *safe* bias (slight over-counting rather than missing a sleeper).

## What went wrong — and how it was fixed

- **97 % in the lab, 60 % on site.** Root cause: the image → tensor pre-processing on the Pi differed from the training pipeline after the `.h5 → .blob` conversion. Fixed by aligning the input vectors. *Lesson: always validate the converted model against the original on the same inputs.*
- **False positives under blue sky.** The initial dataset was collected on a grey day. Diagnosed remotely by injecting known-good images into the camera, then fixed by a targeted remote acquisition + re-training + OTA deployment.
- **Unstable "3 sleepers" class.** Only 62 genuine examples; augmentation produced near-clones and the counter jumped by +3. The class was removed in v2 for reliability.

## Skills demonstrated

`Computer vision` · `CNN / Keras / TensorFlow` · `Multimodal fusion (RGB, mono, depth)` · `Data acquisition, labelling & augmentation` · `Model conversion (ONNX, OpenVINO, TFLite)` · `Edge AI on Raspberry Pi + VPU` · `Real-time multithreading` · `State-machine counting logic` · `Remote deployment over 4G` · `Industrial client project, end to end`

## Next steps (proposed to the client)

Re-position the camera towards the crusher inlet · object detection + tracking (YOLO-type) instead of frame classification · active learning on low-confidence sequences · training the client's technicians to collect and audit data.

---

**Teddy Thamrong** — Élève-ingénieur Arts et Métiers (expertise robotique & IA) · recherche un stage de fin d'études en IA à partir de septembre 2027.
*Code available for discussion during an interview, subject to the client's agreement.*

© 2026 Teddy Thamrong — all rights reserved. No license is granted.
