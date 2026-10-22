# Photo Analyzer and Corrector

`Photo_corrector.ipynb` is a Google Colab notebook that analyzes a photo's visual quality, applies selected corrections, and produces a report. It aims to leave acceptable images alone and accepts a correction only when the checks show a clear improvement.

## Features

- Measures brightness and exposure, contrast, shadow/highlight clipping, blur, and noise.
- Estimates image tilt and detects possible perspective skew in prominent quadrilateral subjects.
- Optionally corrects lens distortion when a compatible camera calibration file is provided.
- Applies guarded corrections such as gamma brightness adjustment, contrast stretching, CLAHE, denoising, rotation, perspective warping, and undistortion.
- Compares before and after quality scores and rejects corrections that do not improve the score or make another check fail.
- Saves a corrected image, a CSV metrics report, and a visual before/after report.
- Includes batch processing and labelled-photo evaluation helpers for tuning thresholds.

## Requirements

The notebook is designed to run in Google Colab. Its main dependencies are:

- Python 3
- OpenCV (`opencv-python`)
- NumPy
- Matplotlib
- pandas

The notebook imports `google.colab.files` to upload images. Run it in Colab, or adapt the upload cells if running in a local Jupyter environment.

## Quick start

1. Open `Photo_corrector.ipynb` in Google Colab (or upload the notebook to a Colab session).
2. Run the notebook cells in order. The upload prompt near the beginning asks you to choose a photo.
3. Review the analysis tables and visualizations as the notebook runs.
4. The end-to-end entry point processes the uploaded photo and saves results in the `output/` folder.

Supported input formats in the processing helpers are JPEG and PNG. Images are analyzed on a copy resized to at most 800 pixels wide; accepted operations are then replayed on the original-size image before saving.

## Results

For an input such as `photo.jpg`, the processing function saves files similar to:

```text
output/
├── photo_fixed.jpg       # corrected image (or an unchanged copy if no fix is accepted)
├── photo_report.csv      # per-check metrics and actions
└── photo_report.png      # before/after images and histograms
```

The notebook also includes optional batch helpers. They read images from `photos/`, save per-image results under `output_batch/`, and can create a contact sheet comparing the original and processed photos.

## How the analyzer works

The notebook computes several metrics on a working-size image, classifies each result as `ok`, `borderline`, or `fail`, and combines weighted check scores into an overall score. Borderline results are reported but do not trigger a correction. Clear failures can trigger one or more correction stages; after each stage, the notebook re-runs the checks and uses safeguards before accepting the output.

Thresholds, margins, correction strengths, and score weights are defined in notebook settings cells. Their values are starting points and may need calibration for a particular photo collection. Blur and clipping are analyzed and reported; the notebook's problem labels identify those as flag-only checks rather than correction targets.

## Batch evaluation and tuning

The later notebook cells support evaluation on a set of example photos:

1. Upload sample images into the `photos/` folder.
2. Fill in the `LABELS` dictionary with known issues for each image. Supported labels include `dark`, `bright`, `lowcontrast`, `clipped`, `blurry`, `noisy`, `tilted`, and `skewed`.
3. Run the evaluation and scoring cells to compare detected problems with labels and inspect false alarms on images marked as good.
4. Review suggested threshold adjustments. The notebook requires explicitly enabling its `APPLY` setting before suggestions are written into the active limits.
5. Run the batch processing and acceptance-gate cells to review full-resolution outputs.

## Optional lens calibration

Lens correction is only attempted when a calibration file is supplied to `process_photo(path, calib=...)`. Without one, the lens-distortion step is skipped. Use calibration data that matches the camera and image setup; an unsuitable calibration can produce an unsuitable correction.

## Notes

- Keep original photos; the notebook writes processed copies into output folders.
- Automatic image-quality checks are heuristic. Inspect the saved image and report, especially for perspective correction, denoising, and contrast enhancement.
- The notebook contains synthetic-damage and replay checks intended to help evaluate behavior and consistency.
