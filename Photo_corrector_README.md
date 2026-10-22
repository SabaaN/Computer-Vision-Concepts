# Photo Analyzer and Corrector

`Photo_corrector.ipynb` is an image-quality analysis and correction workflow built with OpenCV. It measures common photo issues, explains the measurements in tables and plots, and applies selected corrections only when its checks indicate a clear problem. It is designed to preserve good images and to make the processing decisions reviewable.

## What it does

The notebook analyzes each image across these quality dimensions:

| Check | What it measures | Typical interpretation |
| --- | --- | --- |
| Brightness | Mean grayscale brightness | Identifies images that are too dark or too bright |
| Contrast | Difference between the 2nd and 98th grayscale percentiles | Identifies a narrow tonal range without letting a few extreme pixels dominate |
| Clipping | Share of near-black and near-white pixels | Reports lost shadow or highlight detail |
| Blur | Noise-adjusted Laplacian sharpness over image tiles | Identifies soft images while allowing a sharp subject against a soft background |
| Noise | Estimated noise level from flatter, unclipped areas | Identifies visible Gaussian-like noise |
| Impulse noise | Isolated extreme black/white pixels relative to a local median | Detects salt-and-pepper speckles separately |
| Tilt | Dominant direction of strong image edges and confidence in that estimate | Finds modest rotation when enough edges agree |
| Perspective | Shape and corner-angle error of a prominent edge-supported quadrilateral | Finds a likely skewed page, screen, or other four-sided subject |
| Lens distortion | Predicted corner displacement using camera calibration | Runs only when a compatible calibration file is supplied |

Each check is marked `ok`, `borderline`, or `fail`. Borderline results are shown in the report but do not normally trigger a correction. A weighted overall score summarizes the checks; it is a project-specific quality indicator, not a universal photo-quality standard.

## Corrections

The current full pipeline evaluates and processes geometry before appearance:

1. **Lens undistortion**, when calibration is supplied and predicted distortion is clearly above the configured threshold.
2. **Perspective correction**, when a large convex quadrilateral has sufficient support from detected edges and a clear corner-angle error.
3. **Tilt correction**, when enough strong edges agree on a rotation and the estimated tilt clearly exceeds the limit. The rotated image is cropped to remove empty corners.
4. **Exposure and contrast correction.** Gamma adjustment changes lightness to target a configured brightness. Percentile stretching is tried for low contrast; CLAHE is a secondary option if stretching does not resolve the issue.
5. **Denoising.** A median filter handles isolated impulse noise. OpenCV non-local-means denoising handles estimated Gaussian-like noise, with a strength based on the measurement.

Clipping and blur are analyzed and reported, but the notebook does not attempt to reconstruct lost highlights/shadows or sharpen blurry images. It avoids destructive sharpening and does not treat these checks as directly fixable.

### Safeguards

- Processing runs first on a working copy no wider than 800 pixels. Small images are not enlarged.
- A dead zone around each threshold keeps borderline photos from being changed unnecessarily.
- Each correction is measured again. The notebook can reject a candidate if it does not improve the score enough, makes a previously acceptable check fail, produces implausible geometry, or cannot verify the intended result.
- The combined pipeline checks the final score as well, so an individually accepted stage can still be undone if the full chain makes the result worse.
- Accepted operations and their parameters are recorded. The notebook replays those operations at original resolution and verifies the resulting image.
- If no correction is accepted, the saved “fixed” file is copied from the original input so its image bytes remain unchanged.

## Requirements and environment

The notebook is written for **Google Colab** and uses Colab's upload dialog (`google.colab.files`). The imports include:

- Python 3
- OpenCV (`cv2`)
- NumPy
- Matplotlib
- pandas

Colab typically provides these packages. For a local Jupyter environment, install any missing packages and replace the Colab upload cells with local file paths or a local upload mechanism. The processing functions use OpenCV BGR arrays internally; Matplotlib display code converts them to RGB.

## Run one photo

1. Open `Photo_corrector.ipynb` in Google Colab.
2. Run the notebook cells in order so the functions and settings are defined.
3. At the first upload prompt, choose an image. The notebook loads it, checks the image dimensions, and prepares an analysis copy.
4. Review the intermediate metric tables, histograms, and before/after previews.
5. Run the final processing cell. It calls `process_photo(path)` and saves the full-resolution result and reports in `output/`.

The main entry point is:

```python
result = process_photo(
    path,
    out_dir="output",
    calib=None,
    show=True,
    save=True,
)
```

`path` is the input image path. `out_dir` selects the output folder. Set `show=False` to close the report figure instead of displaying it, and `save=False` to skip writing the result files. `calib` can be a path to a supported calibration `.npz` file; without it, lens distortion correction is skipped.

### Supported images

The batch and output helpers recognize `.jpg`, `.jpeg`, and `.png` extensions. Images are loaded in color. Grayscale files are converted to a three-channel color image by OpenCV's color-read mode; alpha transparency is not retained. Large images are reduced proportionally for analysis, then accepted operations are replayed on the original-size image. JPEG output uses the notebook's configured quality; PNG output is written as PNG.

## Output files

For an input named `photo.jpg`, the single-photo run writes files similar to:

```text
output/
├── photo_fixed.jpg       # corrected image or byte-identical copy when no fix is accepted
├── photo_report.csv      # metrics, statuses, scores, and actions
└── photo_report.png      # before/after preview and grayscale histograms
```

The CSV includes one row per quality check and an overall-score row. The plot provides a visual comparison and grayscale histograms to help interpret tonal changes. `process_photo` also returns a Python dictionary containing the pipeline result, report DataFrame, generated file paths, applied operations, image dimensions, scale factor, and processing time.

## Batch evaluation and tuning

The notebook includes cells for building a small labelled evaluation set and processing multiple photos:

1. Upload multiple photos when prompted. The notebook stores them under `photos/`.
2. Run the cell that prints a `LABELS` template, then fill in the known issues for each file. For example:

   ```python
   LABELS = {
       "kitchen.jpg": ["dark", "noisy"],
       "park.jpg": [],
       "document.jpg": ["skewed"],
   }
   ```

3. Run the evaluation cells to compare labels with detected failures, inspect borderline cases, and review the raw metrics. The scoring helper reports problems found, false alarms, and missed labels. It also identifies labelled-good photos that were flagged or changed.
4. Review suggested limits. Suggestions are only applied when the labelled and unlabelled metric ranges can be separated; set `APPLY = True` in the relevant cell to write those suggestions into the active limits.
5. Run the full batch processor. It writes each image's corrected copy and reports under `output_batch/`, then creates a contact sheet for visual review.
6. Review the acceptance gates at the end of the notebook. They check the labelled sample size, changes to known-good images, problem detection rate, score regressions, and agreement between working-size analysis and full-resolution replay.

The labels are user-supplied ground truth. Results from a small or unrepresentative set should not be treated as reliable performance estimates for other cameras or photo collections.

## Optional camera calibration

Lens correction requires a NumPy `.npz` file containing:

- `K`: the camera intrinsic matrix
- `dist`: distortion coefficients
- `size`: (width, height) at which the calibration was made

Pass its path as `calib="camera_calibration.npz"` to `process_photo`. The notebook rescales the camera matrix for the working or full image size, but skips correction when the calibration aspect ratio does not match. It also checks whether the predicted distortion is substantial and whether the undistorted crop retains enough image area. Use calibration made for the same camera and compatible capture setup.

## Settings and limitations

Thresholds, margins, score weights, target brightness, denoising strengths, edge-confidence requirements, and geometric constraints are collected in settings cells near the relevant analysis stages. They are editable starting values, not universal thresholds. Tune them against representative, manually labelled photos and inspect the output visually.

Automatic checks can confuse scene content with image defects. For example, a deliberately dark scene may be labelled underexposed, a shallow-depth-of-field image may contain a soft background, and a building's intended perspective may resemble skew. Geometric correction can crop content; denoising and contrast enhancement can remove detail or amplify artifacts. Keep the original images and review processed results before using them as final deliverables.

The notebook also contains synthetic-damage experiments and replay checks. These help explore known changes such as added blur, noise, or rotation, and check that full-resolution replay follows the analysis decisions. They do not replace testing on a diverse real-world image collection.
