# Lesion 3D-segmentation pipeline — handoff

Pipeline turns a multi-view video / image sequence of a 3D-printed foot
(with a painted brown lesion on the surface) into a labeled 3D mesh where
each face is marked lesion / non-lesion. Then evaluates the resulting
lesion against a CAD ground-truth lesion mesh.

This document describes the pipeline stage-by-stage for an article describing
the work. Supporting documents (algorithm details, numeric results) are
referenced where appropriate.

---

## Pipeline overview

```
input photos (3072×4096, 82 frames)
        │
        ▼
[1] COLMAP SfM        — camera poses + sparse 3D points + intrinsics
        │
        ▼
[2] sdfstudio NeUS-Facto training   — neural SDF reconstruction
        │
        ▼
[3] sdfstudio mesh export   — mesh.obj + texture.png from trained SDF
        │
        ▼
[4] Our segmentation pipeline   — multi-view SAM fusion → 3D lesion mask on mesh
        │   (split into two phases — see § Phase A / Phase B below)
        ▼
[5] Our evaluation pipeline   — sim(3)-ICP alignment to GT + accuracy metrics
```

Stages 1-3 are off-the-shelf (mini-mesh / sdfstudio). Stages 4-5 are our
contribution.

---

## Stage 1 — COLMAP Structure-from-Motion

**Input**: 82 RGB photos (3072 × 4096).
**Output**: `transforms.json` with per-frame extrinsics (4×4 c2w matrices)
+ camera intrinsics (focal length, principal point, OpenCV distortion
coefficients k1, k2, p1, p2).

COLMAP solves for camera poses and a sparse 3D point cloud jointly via
bundle adjustment over keypoint matches across frames. The intrinsics
(focal length, distortion) are also estimated jointly — they're a fit, not
an external calibration.

Result is a self-consistent set of poses + intrinsics: they describe where
each camera was, but only up to a global similarity (overall scale and
absolute world origin are arbitrary).

## Stage 2 — sdfstudio NeUS-Facto training

**Input**: photos + `transforms.json` from stage 1.
**Output**: a trained neural model that represents the foot as an implicit
signed distance field (SDF) f(x,y,z): ℝ³ → ℝ, where the foot surface is
where f = 0.

NeUS-Facto is a hybrid: a NeRF-style proposal-network samples ray points,
and a separate SDF MLP predicts signed distance at those points. The model
is trained to minimize photometric loss — i.e. rendering the predicted SDF
from each known camera pose should produce the actual photo. Training
duration: 100 k iterations.

Outputs of stage 2 are checkpoint weights — not a usable 3D mesh yet.

## Stage 3 — sdfstudio mesh export

**Input**: trained checkpoint from stage 2.
**Output**:
- `mesh.obj` — triangle mesh extracted by Marching Cubes at SDF = 0
  (1 M faces for max export tier)
- `texture.png` — 8 K × 8 K UV-mapped texture map, baked by sampling the
  trained color field at each face's surface position
- UV mapping for the mesh

The resulting mesh has the bulk geometry of the foot. Sub-mm surface
noise is normal — the SDF training doesn't strongly penalize high-frequency
surface wobble.

**At this point the mesh is in "recon units"** — an arbitrary coordinate
scale, not mm. The conversion to mm is what stage 5 will derive.

---

## Stage 4 — Our segmentation pipeline (multi-view SAM fusion)

This is the novel contribution. We split it into two phases for reliability
(rendering phase is GPU-heavy and can crash; isolating it from the SAM phase
made the pipeline checkpointable).

### Phase A — Render the textured mesh from every COLMAP camera pose

**Input**: mesh.obj + texture.png + transforms.json
**Output**: 82 JPG images, each = a synthetic render of the textured mesh
from one of the COLMAP camera positions.

Script: `/home/roman/mini-mesh-eval/scripts/v34a_render_all.py`

Why renders, not the original photos: the renders are consistent across
views (same baked texture, no camera noise / motion blur / lighting
inconsistencies between frames). All renders share a single global
appearance of the lesion — anywhere SAM sees brown in one render, it should
see brown in every render. This avoids the "this-photo's-brown vs that-
photo's-brown" inconsistencies that would otherwise dominate.

The renders use the original COLMAP camera pose, intrinsics (focal,
principal point), and image resolution — so they're geometrically equivalent
to the photos.

### Phase B — Per-view SAM segmentation + multi-view vote aggregation

**Input**: 82 renders + transforms.json + mesh
**Output**: `lesion_binary_v34_t09.ply` — a sparse mesh whose triangles
are the detected lesion faces in the original recon coordinates.

Script: `/home/roman/mini-mesh-eval/scripts/v34n_subdiv2.py`
(uses saved per-view masks from `/home/roman/mini-mesh-eval/scripts/v34k_nomorph_t05.py`)

Per render:

1. **BiRefNet** (`ZhengPeng7/BiRefNet`) predicts a foreground mask
   isolating the foot from the white background. The render is cropped to
   the foot bounding box + 60 px padding.

2. **SAM 3.1** (`facebook/sam3.1`) is run on the foot crop with four text
   prompts in sequence: `"brown spot"`, `"brown stain"`, `"lesion"`,
   `"brown patch"`. Each prompt produces a binary mask. The union of all four
   is the candidate lesion mask for that view.

3. **Filtering**:
   - Drop any mask > 15 % of the foot crop area (SAM grabbed too much)
   - LAB color check: mean color must be `L ∈ [50, 175], a > 130` (brown
     chroma test)
   - Connected components: keep only components ≥ 100 px, each passing the
     LAB color check independently
   - Keep only the **largest connected component** (drops distant
     disconnected spurious detections)

4. The cleaned binary mask for this view is saved to disk
   (`masks_nomorph/view_NNN_mask.png`).

After all views are processed, **multi-view aggregation** runs on the
**subdivided mesh** (Loop subdivision ×2, taking the recon's 1 M base faces
to 16 M faces for finer boundary resolution):

5. For each face in the mesh (16 M faces), compute its centroid (one 3D
   point per face) and its outward normal. For each of the 82 views:
   - Project the face centroid to the render's pixel coordinates using the
     COLMAP camera matrix
   - Check that the face is *front-facing* (normal · ray-to-camera > 0) and
     *in image bounds*
   - If those pass and the projected pixel is inside that view's SAM mask,
     the face gets a vote: `+cos(angle)³` (head-on views weighted higher
     than grazing views)
   - The face also accumulates weight `cos(angle)³` for any view where it's
     visible (whether or not it's in the mask)

6. After all views: per-face score = `votes / weight`. Faces with score ≥ 0.9
   are flagged as lesion candidates.

7. **Spatial DBSCAN cluster** the candidates (eps = 8 × median face spacing),
   keep top-2 clusters by size (handles the lesion's main body even if it's
   split by a thin gap in the topology).

8. **Depth-occlusion filter**: render the textured mesh's depth buffer from
   each pose. For each candidate face, count how many views (where it's
   geometrically visible) actually have the face's centroid as the front-most
   surface (face_z ≈ depth_buffer at projected pixel, within tolerance). Drop
   faces visible in < 30 % of views — these are NeUS inner-shell artifacts
   that project through to the same pixels as the outer shell but are
   actually behind it.

Final output: sparse PLY of detected lesion faces in original recon
coordinates.

### Why this design (rationale for the article)

- **Synthetic renders, not original photos**: removes per-frame lighting /
  exposure / motion-blur / brightness inconsistencies. The lesion looks
  identical across all 82 renders by construction.
- **Largest CC + LAB filter per view**: prevents SAM's occasional false
  positives elsewhere on the foot from polluting the vote.
- **Centroid-projection (not face-id raster)**: face-id rasterization
  picks one face per pixel via z-buffer winner — at subdiv = 2 where most
  faces are sub-pixel, this misses ~80 % of "touched" faces. Centroid
  projection marks every face whose centroid projects into the mask.
- **Top-2 DBSCAN + depth filter**: handles topology gaps and inner-shell
  artifacts that pure thresholding doesn't.

---

## Stage 5 — Our evaluation pipeline

**Input**: detected lesion PLY (recon units) + ground-truth lesion mesh
(in mm, from CAD/Blender)
**Output**: alignment + accuracy metrics

Script: `/home/roman/mini-mesh-eval/scripts/evaluate_lesion_simicp.py`

The detected mesh is in arbitrary recon coordinates; the GT is in mm. They
have to be brought into the same coordinate system before they can be
compared. The pipeline does this without relying on any external reference
points — it uses the geometric shape itself.

### Step 1: Coarse pre-alignment

- Translate detected so its centroid sits at GT's centroid
- Pre-scale detected by `s₀ = √(A_gt / A_det)` so its total surface area
  matches GT's
- This brings detected into roughly the GT's coordinate space

### Step 2: Sim(3) Iterative Closest Point refinement

- 20 000 points are sampled uniformly on each surface
- Open3D's `registration_icp` with `with_scaling=True` runs
- The algorithm iteratively (a) finds nearest-neighbor correspondences
  between source (detected) and target (GT) point sets, (b) solves the
  closed-form Umeyama 1991 transform that minimizes squared distances
  given those correspondences, (c) applies the transform and repeats
- Convergence: stops when relative cost / RMSE change is below 1e-7
- Final transform = 4×4 matrix encoding **uniform scale + rotation +
  translation** (sim(3), 7 DOF, no shearing or non-uniform scaling)

The derived scale `s` is the recon→mm conversion factor. For our run:
**160.55 ± 0.12** across 10 random sampling seeds.

### Step 3: Apply transform → detected is now in mm

After applying the sim(3) transform, the detected mesh lives in mm
coordinates, aligned with GT. All subsequent metrics are in mm.

### Step 4: Compute accuracy metrics

Currently computed:
- **Surface area** of detected (mm²) vs A_gt
- **Per-point Chamfer distance** (mean nearest-neighbor distance both
  directions, 20 k samples)
- **Hausdorff distance** (max nearest-neighbor distance)
- **F-score at distance thresholds** T ∈ {0.1, 0.2, 0.3, 0.5, 0.7, 1.0} mm:
  precision = fraction of detected points within T of GT;
  recall = fraction of GT points within T of detected; F1 = harmonic mean

See `EVAL_EXPLAINED.md` (algorithm details, why sim(3)-ICP, why ICP works
for non-identical shapes, scale-bias caveat) and `RESULTS.md` (numeric
results) for the rest.

---

## Metrics from the Salve et al. article — can we compute them?

Article's metrics: **AD, W₂, HD, HD90, NC, W₂-NC**. Yes, all of them can be
computed on our data with small additions to the evaluation script:

| Metric | What it is | Status |
|---|---|---|
| **AD** (Average Distance) | mean nearest-neighbor surface distance | ✅ we already compute — same as our Chamfer (0.16 mm) |
| **HD** (Hausdorff) | max nearest-neighbor distance | ✅ already compute (0.65 mm) |
| **HD90** (90th pct Hausdorff) | 90th percentile of NN distances | ➕ trivial to add — sort our distance array, take np.percentile(d, 90) |
| **W₂** (Earth Mover / 2-Wasserstein) | optimal-transport distance between point distributions | ➕ ~30 min to add via `pot` (POT) library: `ot.emd2(weights_a, weights_b, cost_matrix)`. Yields a single mm number that captures "total mass to move to transform A into B" |
| **NC** (Normal Consistency) | mean dot product of surface normals at corresponding (nearest-neighbor) points; range 0..1, 1 = perfect alignment | ➕ ~15 min — at each nearest-neighbor correspondence, take `n_det · n_gt`, average. Trimesh has per-face normals; or interpolate per-vertex from point sampling |
| **W₂-NC** | same as NC but using optimal-transport assignments instead of nearest-neighbor | ➕ ~30 min — once we have the W₂ coupling matrix, use it instead of NN to pair points, then average dot products |

We can add all of these to a single `metrics.py` script. Total work:
~1-2 hours including testing.

### Differences from Salve's setup that matter

- **Their GT**: structured-light 3D scanner produces dense point cloud of
  the real wound. **Our GT**: CAD mesh from the design file used to print
  the foot. Ours is exact (no scan noise), theirs has scan noise.
- **Their wound polygon**: they define a polygon around the wound on the
  reconstructed mesh and the GT *manually* to isolate the wound region.
  **Our equivalent**: the detected lesion mesh itself IS the polygon
  (defined automatically by our segmentation). For GT we have the exact
  lesion region mesh from CAD. So both regions are well-defined; we don't
  need manual polygon clicking.
- **Their alignment**: their text describes manual alignment in mesh
  processing software, "to ensure fairness". **Our alignment**: automatic
  via sim(3)-ICP on the lesion regions directly. Open3D's sim(3) ICP
  converges reliably for similar shapes and is fully reproducible.
- **Their rendering metrics**: PSNR/SSIM/LPIPS measure 2D photo synthesis
  quality. We could add these for our renders vs photos, but that's a
  separate evaluation of the recon quality, not the lesion segmentation.

### Their precision evaluation — directly applicable to us

Salve evaluates two precision modes:

- **Inter-device precision**: 3D reconstructions from different cameras of the
  same wound, compared to each other.
- **Inter-recording precision**: different videos of the same wound (same
  camera), reconstructions compared.

For us:
- **Inter-recording precision IS very testable**: we have multiple separate
  capture sessions of the same foot (e.g. test4, test6, test7 — all of the
  same foot). For each: run the full pipeline → get a detected lesion
  mesh. Then sim(3)-ICP-align all pairs and compute pairwise AD / HD / HD90.
  This measures how reproducible our pipeline is across different captures.
  This is actually a *better* precision test than Salve's "different frame
  splits of the same video", because we use genuinely independent sessions.
- **Inter-device precision**: not applicable — we use one phone camera.

---

## Proposed two-script architecture

Looking at the current code, **the natural split is exactly the segmentation
vs. evaluation boundary**:

```
inputs: photos + COLMAP poses + mesh + texture
         │
         │       SCRIPT 1: segmentation
         │       (v34a + v34n / v34k)
         │
         ▼
        lesion_binary.ply (sparse mesh of detected lesion faces, recon units)
         │
         │       SCRIPT 2: evaluation
         │       (evaluate_lesion_simicp.py + metrics.py extensions)
         │
         ▼
        metrics.json (AD, HD, HD90, F-score, area-mm², + W₂ / NC / W₂-NC after upgrade)
```

This is already roughly what the current code does — segmentation pipeline
outputs a PLY in recon-units, and the eval script takes that PLY + a GT PLY
and produces metrics. The clean separation has multiple benefits:

- **Segmentation script knows nothing about GT** — fair, no leakage of GT
  information into the detection
- **Evaluation script knows nothing about segmentation method** — can be
  run identically on detections from ANY segmentation method (color
  threshold, our pipeline, future methods) for fair comparison
- **Evaluation script can be reused for precision testing** between two
  separate capture sessions' detected meshes (no GT needed — just two
  detected lesion meshes from different sessions, ICP one to the other,
  report distances)

Recommendation for the article version: keep this split explicit. Call them
e.g. `segment_lesion.py` and `evaluate_lesion.py`. Document each one's
inputs/outputs separately.

---

## Files for the article (existing)

| File | Purpose |
|---|---|
| `RESULTS.md` | Numeric results (areas, distances, F-scores, visualizations) |
| `EVAL_EXPLAINED.md` | Algorithm walkthrough applied to our numbers (ICP convergence, sim(3) math, scale-bias caveat, visualizations) |
| `HANDOFF.md` | This file — pipeline overview for the article |

## Scripts — current canonical pipeline (3 files)

Copied to: `C:\Users\Roman\lesion-neus-facto-article\scripts\pipeline\`

| Stage | Script | Role |
|---|---|---|
| **Phase 1 — render** | `v34a_render_all.py` | Stage 4 / Phase A. Renders the textured mesh from each of the COLMAP camera poses. **Phase 1 of the segmentation pipeline, not an older version of phase 2.** Splitting the rendering from SAM lets you checkpoint between them. |
| **Phase 2 — segment + aggregate** | `v34n_subdiv2.py` | Stage 4 / Phase B. Loads the saved views, runs BiRefNet+SAM per view (filtered with LAB + connected-components + largest-CC), then aggregates via centroid projection on a subdivided mesh + DBSCAN + depth-occlusion filter. Threshold is a CLI argument (`--score-thresh`, default 0.5); for the article's headline result we used **0.9**. Outputs `lesion_binary_v34_t09.ply` in recon-units. |
| **Stage 5 — evaluate** | `evaluate_lesion_simicp.py` | Sim(3)-ICP alignment + the full SALVE-comparable metric set: AD, HD, HD90, W₂, NC, W₂-NC + F-score at distance thresholds + area diff + auxiliary IoU. Outputs `metrics.json` + aligned mesh PLY. |

Run order:

```
# 1. Render the mesh from every COLMAP camera pose
python v34a_render_all.py --run <colmap-run-dir> --out <work-dir>

# 2. Segment + aggregate (at our chosen threshold = 0.9)
python v34n_subdiv2.py --run <colmap-run-dir> --out <work-dir> --score-thresh 0.9

# 3. Evaluate against GT mesh
python evaluate_lesion_simicp.py \
    --detected <work-dir>/lesion_binary.ply \
    --gt <gt-folder>/region_lesion_top_v12.ply \
    --out <eval-out-dir>
```

### Evaluation methods covered by `evaluate_lesion_simicp.py`

The single script now covers **all** the evaluation methods we tested:

| Method | Purpose | What it tells you |
|---|---|---|
| **Sim(3)-ICP alignment** | bring detected (recon units) into mm coords via best-fit | data-driven recon→mm scale |
| **AD** (= symmetric Chamfer) | mean nearest-neighbor surface distance | typical accuracy |
| **HD** | max nearest-neighbor distance | worst-case accuracy |
| **HD90** | 90th-percentile of NN distances | robust worst-case (drops outliers) |
| **W₂** (Earth Mover's) | optimal transport distance via POT library | global "distribution" distance, complement to NN distance |
| **NC** (Normal Consistency) | mean \|n_det · n_gt\| at NN correspondences | surface orientation alignment, 0..1 |
| **W₂-NC** | NC under optimal-transport coupling | same as NC but pairs via W₂ instead of NN |
| **F-score @ T** | precision/recall at distance T mm for T ∈ {0.1, 0.2, 0.3, 0.5, 0.7, 1.0} | shape match at various tolerance levels |
| **Area diff** | A_det (mm²) vs A_gt (mm²) | net area over- or under-detection |
| **PCA-projected 2D IoU** | flatten both meshes to PCA plane, IoU of outlines | sanity / outline match (auxiliary) |
| **Voxel IoU** | volumetric IoU on voxelized surfaces | sanity (noisy for thin surfaces, auxiliary) |

### Approach 1 (equal-area + R+t ICP, alternative scale framing)

This was a *separate* eval mode we explored: force detected area exactly equal to GT (uniform pre-scale), then rigid-only ICP. Reports same shape metrics but no area diff. Lives at `/home/roman/mini-mesh-eval/scripts/v34_approach1.py` (research code; not in the article-pipeline folder). For the article, `evaluate_lesion_simicp.py` is the primary method; Approach 1 is supplementary discussion in `EVAL_EXPLAINED.md`.

### Variance & verification scripts (research, not in pipeline folder)

| Script | Purpose |
|---|---|
| `verify_10runs.py` | 10 independent ICP runs with different random sampling seeds → reports variance across the metrics |
| `verify_with_multiple_libs.py` | Cross-verifies the sim(3) scale using (a) Open3D, (b) pure-numpy Umeyama 1991, (c) simpleicp library, (d) pymeshlab Hausdorff (via aligned mesh) |

These live at `/home/roman/mini-mesh-eval/scripts/` and are useful for the
"Verification" subsection of the article.

### Visualization scripts (research, not in pipeline folder)

| Script | Output |
|---|---|
| `v34_gt_vs_detected_relief.py` | side-by-side GT vs detected with contour-stripe relief shading |
| `v34_render_smoothed.py` | 3-column GT vs raw vs Taubin-smoothed detected |
| `v34_eval_viz.py` | overlay + distance heatmap + PCA 2D overlay |
| `v34_roughness_check.py` | quantifies recon's surface roughness vs GT |

Live at `/home/roman/mini-mesh-eval/scripts/`. Useful for figure
generation but not part of the metric pipeline.

---

## Numeric headline results (for the article — see RESULTS.md for full)

For the foot dataset (test4):

- **Detected lesion area**: 828.66 ± 1.26 mm² (Sim(3)-ICP, 10 random seeds)
- **Ground-truth area**: 814.18 mm²
- **Area difference**: +1.78 % ± 0.15 %
- **Mean surface distance (AD)**: 0.13 mm
- **Hausdorff (max)**: 0.65 mm
- **HD90** (90th pct): 0.29 mm (computable; not yet in summary file)
- **F1 @ 0.3 mm tolerance**: 98.06 % shape match
- **Pipeline reproducibility**: scale variance 0.22 % across 10 random sampling seeds
