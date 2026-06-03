# [The florence 2D/3D hybrid face dataset](zotero://select/library/items/MHCGN8KH)

 **Authors:** Andrew D. Bagdanov, Alberto Del Bimbo, Iacopo Masi
 **Published:** 2011
 **Journal/Conference:** Proceedings of the 2011 joint ACM workshop on Human gesture and behavior understanding
**Citekey:** `bagdanovFlorence2D3D2011`

---
## Abstract
> [!abstract]
>  ```
>  This article describes a new dataset under construction at the Media Integration and Communication Center and the University of Florence. The dataset consists of high-resolution 3D scans of human faces along with several video sequences of varying resolution and zoom level. Each subject is recorded under various scenarios, settings and conditions. This dataset is being constructed specifically to support research on techniques that bridge the gap between 2D, appearance-based recognition techniques, and fully 3D approaches. It is designed to simulate, in a controlled fashion, realistic surveillance conditions and to probe the efficacy of exploiting 3D models in real scenarios.
>  ```

---

%% begin notes %%
## Summary
## Introduction

Face recognition from surveillance imagery is a hard problem: video subjects are uncooperative, lighting is uncontrolled, and resolution varies widely with distance. The Florence 2D/3D Hybrid Face Dataset was created specifically to bridge the gap between purely 2D, appearance-based recognition methods and fully 3D model-based approaches. By pairing high-resolution 3D face scans with matched 2D video sequences captured under realistic surveillance conditions, the dataset enables research into how 3D geometry can augment recognition algorithms that must operate on ordinary camera footage.

## Related Work

The paper positions Florence against the following bodies of work:

| Source | Relevance |
|---|---|
| Zhao et al., *ACM Computing Surveys* 2003 | Comprehensive survey of 2D face recognition algorithms; motivates the need to go beyond appearance-only methods |
| Berretti, Del Bimbo & Pala, *TPAMI* 2010 | Pure 3D face recognition using isogeodesic stripes; represents the "fully 3D" end of the spectrum the dataset is designed to bridge |
| Del Bimbo et al., *CVIU* 2010 | PTZ camera tracking with visual landmark maps; motivates the dataset's PTZ video capture scenarios |
| Fanelli, Gall & van Gool, *CVPR* 2011 | Real-time head pose estimation with random regression forests; one of the explicit downstream tasks the dataset is designed to support |

The dataset occupies the niche between standard 2D image collections (the dominant dataset type at time of writing) and emerging pure-3D databases, filling the largely unexplored 2D–3D intersection.

## Methods & Architecture

Florence is a dataset paper; the "method" is a multi-modal capture pipeline:

**3D acquisition**
- Scanner: 3dMDface System
- Per subject: minimum **3 high-resolution 3D models** — two frontal (train/test split), one left lateral, one right lateral, plus an optional glasses model
- Mesh accuracy: ~$0.2$ mm RMS reconstruction error on average
- Mesh density: ~40,000 vertices, ~80,000 facets
- Texture: stereo image at $3341 \times 2027$ px
- Formats: OBJ, PLY, VRML (mesh + texture)

**2D video acquisition** — three video streams per subject:

| Stream | Resolution | Camera | Zoom levels | Scenario |
|---|---|---|---|---|
| Indoor HD | $1280 \times 720$ | AXIS Q1755 | 4 | Cooperative; controlled frontal lighting; scripted head rotations |
| Indoor PTZ | $704 \times 576$ (4CIF) | AXIS PTZ Q6032-E | 3 | Uncooperative; spontaneous behaviour |
| Outdoor PTZ | $736 \times 544$ | SONY RZ30-P | 3 | Uncooperative; uncontrolled lighting, shadows, highlights |

Scripted head rotations for the HD stream cover six gaze targets: top-right, top-left, middle-right, middle-left, bottom-right, bottom-left. Frame rates: ~20 fps (indoor), ~5–7 fps (outdoor).

**Dataset scale at time of writing:** ~50 adult subjects (October 2011); target of 100+ subjects by Fall 2011.

## Experiments & Results

This is a dataset description paper — no recognition experiments or quantitative benchmarks are reported. The paper does not present accuracy numbers, baselines, or ablations. The only quantitative figures given are capture specifications: mesh RMS error (~$0.2$ mm), vertex count (~40,000), facet count (~80,000), and texture resolution ($3341 \times 2027$ px).

## Conclusion

The authors claim to have designed and begun populating a dataset that uniquely pairs accurate 3D face models with multi-condition 2D video, specifically to support investigation of how 3D information can improve face analysis tasks (pose estimation, recognition) that are classically challenging from 2D imagery alone. The dataset targets realistic surveillance scenarios through the use of PTZ cameras at varying zoom levels and cooperative/uncooperative subject conditions.

## Impact

Florence provides the infrastructure needed to study whether 3D face models can serve as a prior for boosting recognition under the unconstrained, low-resolution, variable-pose conditions typical of real surveillance — a question that remained largely unanswered at the time. Its pairing of sub-millimeter-accuracy 3D scans with matched PTZ video makes it distinctly useful for head pose estimation and face recognition research that requires ground-truth 3D geometry. The dataset is modest in scale (targeting 100 subjects), which limits statistical conclusions but makes it tractable for methods that require per-subject 3D fitting. It does not address large-scale in-the-wild recognition (no internet imagery, no expression variation protocol), so its applicability is narrower than LFW-style benchmarks. In the multi-view and RGB-D pose estimation context, Florence is a natural reference for validating 3D-model-assisted pose pipelines where a scan-quality mesh is available per identity.
%% end notes %%

## Notes & Highlights%% begin obsidian-freeform-notes %%

### Freeform Notes%% end obsidian-freeform-notes %%

### From Zotero

%% Import Date: 2026-06-02T22:02:20.314-04:00 %%
