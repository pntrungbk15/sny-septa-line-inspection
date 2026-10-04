# Technical Overview

Technical notes on the **portfolio implementation**. The anomaly models of the production system (MetaUAS,
PatchCore, MemSeg) are described in [production_context.md](production_context.md); the demo deliberately uses a
weight-free model so that nothing trained on production data or on licence-restricted datasets is involved.

## Anomaly detection (demo model)

A PaDiM-style patch-statistics model, learned from good fabric only:

1. **Illumination normalisation.** Local mean and variance normalisation removes slow lighting changes.
2. **Filter bank.** Ten responses per pixel: smoothed intensity, gradient energy at two scales, Laplacian of
   Gaussian at two scales, local contrast, Hessian components and |LoG|.
3. **Patch features.** Responses pooled over 12 px patches every 6 px (mean and mean absolute value), giving 20
   features per patch.
4. **Fit.** One Gaussian with a shrunk covariance over the patches of 16 generated good sections.
5. **Score.** The Mahalanobis distance of each patch is interpolated back to pixels as a heatmap. The pixel
   threshold sits above a high percentile of the good maps; the **image score is the mean of the top-k map values**,
   the same top-k idea used by the production system.

Fitting takes a few seconds and needs no pretrained weights. It suits stationary textures like knit and coating; real
fabric would call for a deep feature extractor, which the backend interface allows.

## From heatmap to defect

| Step | Purpose |
|---|---|
| Threshold the heatmap | Anomaly mask |
| Connected components | Objects above a minimum area (e.g. ≥ 0.3 mm²) |
| Classification | Nearest-prototype classifier over elongation, orientation, contrast sign and fill ratio, with softmax confidences (slub, knot, ladder, scratch, lint) |
| Confidence filter | Removes weak objects before the rules |
| Measurement | Area in mm², length in mm, from each camera's mm/px calibration |

## Classical measurement

- **Edge caliper (width and edges).** Sub-pixel edge positions on every row, a robust line fit per edge; fabric
  width, edge deviation from the fitted line, and web skew.
- **Colour (shade and stains).** CIEDE2000 colour difference to the product's reference shade, per zone; stains are
  connected regions above a ΔE threshold, measured in mm².
- **Backlight (holes).** Transmitted light thresholded, opened, and measured as hole count and diameter.
- **Label and lot code.** Template matching locates a fixture mark; a light OCR reads the lot line by normalised
  correlation against the label printer's font, with a confidence discounted by the margin to the runner-up glyph.
- **Print registration.** ORB keypoints matched to the golden artwork with a ratio test; a RANSAC similarity
  transform gives offset in mm and angle in degrees.
  The synthetic golden artwork used by the demo: [golden-print-reference.png](../assets/demo/golden-print-reference.png).

## Rules engine

- Operators: min, max, range, tolerance, count, text pattern, equality with the production order.
- Filters: region (only objects inside a named ROI), class, minimum confidence.
- Aggregation: worst object, sum, count.
- Severity: minor, major, critical; the GOOD / NG node decides which severities reject.
- Logic: AND / OR groups.
- Output: one human-readable sentence per rule with the measured value and the limit.

## Inference backend

Segmentation, detection and classification go through one backend interface. The default classical backend is
deterministic and weight-free; an ONNX backend runs a user-supplied segmenter or classifier through ONNX Runtime.
Nothing is bundled.

## Verification on synthetic data

Because defects are injected, ground truth is known. The portfolio project's checks confirm that every factory
pipeline returns the injected verdict on fixed seeds and that reported objects overlap the injected defect boxes,
that measured width and print angle agree with the rendered values, and that the colour-difference implementation
reproduces published reference pairs. These are correctness checks on synthetic data, **not** accuracy figures for a
real line.

## Technologies

Python, OpenCV, NumPy, SQLite, PyQt5 with FluentQt (the author's Fluent-style component library); ONNX Runtime
optional.
