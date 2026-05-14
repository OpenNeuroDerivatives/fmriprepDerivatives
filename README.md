# fmriprep derivatives — opinionated standard for OpenNeuro

A community-maintained reference for **how to run fmriprep across OpenNeuro datasets** so derivatives are comparable across producers (Joe, Felix, Austin/Yarik, …) and useful to downstream consumers (META, individual researchers, AI pipelines).

This repo is **opinions, not data.** Discussion happens via PRs and line-comments on the documents here.

Status: **WIP draft, seeded from the 2026-05-13 meeting between Felix Hoffstaedter, Joe Wexler, Austin Macdonald, and Yarik Halchenko.** Chris Markiewicz (fmriprep) was sick and missed the meeting; some questions are pending his input.

Meeting recording: <https://dartmouth.zoom.us/rec/play/OXvY_H8O2Kuya3HOWgpmtNRjw9auoZz4sCYkxS_P3GWG3kYnmHNN_9w5PwO3-SbT9pa3Ex8OhIeILeqQ.DGFKhh22rtjOcDoT?autoplay=true&startTime=1778698976000>

## Pipeline shape

We're standardizing on a **staged pipeline** rather than a single `fmriprep` invocation.

0. **MRIQC** (gate; must succeed before any fmriprep stage).
1. **`fmriprep --anat-only` + `--output-spaces`** — produces the full anatomical BIDS-derivatives scaffold (bias-corrected T1w, brain mask, segmentation, T1w↔MNI transforms in both NLin2009cAsym and NLin6Asym, surfaces, fsLR/fsaverage registration). Also produces the FreeSurfer subjects-directory, which is split out as a separate datalad subdataset mounted at `sourcedata/freesurfer/` so consumers can fetch it independently. Slow (~hours wall-clock, dominated by FreeSurfer's `recon-all`); low RAM; no BOLD touched.
2. **`fmriprep --level minimal`** — adds BOLD-side transforms (HMC, BOLD→T1w coreg, SDC). No confounds, no resampling, no BOLD outputs in any template space. Smallest fmriprep BOLD derivative.
3. **`fmriprep --level resample`** — applies the composed transform chain internally to derive denoising **regressors** (motion params, framewise displacement, WM/CSF means, aCompCor components), written to `_desc-confounds_timeseries.tsv`. **Does NOT itself denoise the BOLD** — that's a downstream step. Probably does NOT serialize the 4D resampled BOLD in template space either; **TODO: confirm with Chris.**
4. **`fmriprep --level full`** (optional) — additionally writes the 4D resampled BOLD per output space. ~5-8× storage cost over Stage 3. Skip unless explicitly needed.

**Stage outputs share one derivative dataset per (raw_dataset, fmriprep_version) pair.** Successive stages are sequential `datalad run` commits on the same dataset, not separate datasets. Within that, the **FreeSurfer subjects-directory** (under `sourcedata/freesurfer/` in the fmriprep derivative) is split out as its **own datalad subdataset** so consumers can fetch just the FreeSurfer side without the rest, or vice versa.

> **TODO (block on Chris): clarify `--level resample` semantics.** What exactly does it emit? Confounds TSV — yes. 4D timeseries in template space — Felix believes no, Joe and Felix both uncertain. This is the most important open question; the answer determines whether `--level resample` is a sufficient "good default" or whether `--level full` is needed for downstream-ready derivatives.

## Recommended fmriprep invocation

Applied at each stage. Variables in `<angle brackets>`.

```bash
fmriprep <bids_dir> <output_dir> participant \
  --participant-label <subid> \
  --output-spaces MNI152NLin2009cAsym:res-2 MNI152NLin6Asym:res-2 \
  --cifti-output 91k \
  --random-seed 12345 \
  --skull-strip-fixed-seed \
  --use-syn-sdc warn \
  --me-output-echos \
  --md-only-boilerplate \
  --skip-bids-validation \
  --notrack \
  --fs-license-file <path> \
  --level <minimal|resample|full> \
  --nthreads <N> --omp-nthreads <M> --mem-mb <MB>
```

Stage 1 uses `--anat-only` instead of `--level`. recon-all runs inside that invocation; you do not invoke `recon-all` separately.

### Why each flag

| Flag | Why |
|---|---|
| `--output-spaces MNI152NLin2009cAsym:res-2 MNI152NLin6Asym:res-2` | Both modern (NLin2009cAsym) and HCP/AROMA-lineage (NLin6Asym) volumetric. `res-2` (2mm) is the de-facto fMRI analysis standard; `res-native` is a space-saving hack, not a recommendation. |
| `--cifti-output 91k` | HCP-standard grayordinate density. Implicitly pulls in fsLR + surface registration. |
| `--random-seed 12345` | Reproducibility. fmriprep auto-generates if unset; explicit value enables bit-identical reruns. |
| `--skull-strip-fixed-seed` | Reproducibility of skull-stripping specifically (Atropos has stochastic init). |
| `--use-syn-sdc warn` | SyN-based SDC fallback when no fieldmap is present; log when used. Matches Joe's practice. **TODO: confirm with reviewers** — Felix prefers not to override fmriprep defaults. |
| `--me-output-echos` | No-op on single-echo data. On multi-echo data, ships per-echo BOLD so downstream tools like `tedana` can do TE-dependent denoising. |
| `--md-only-boilerplate` | Skip PDF render of methods text. Markdown is sufficient and faster. |
| `--skip-bids-validation` | Assume input is validated upstream (in our MRIQC gate). |
| `--notrack` | Disable telemetry. |
| `--level` | Picks pipeline stage. See pipeline diagram above. |

### Flags that should not be set

- `--ignore slicetiming` — leave unset. fmriprep is data-driven: it corrects if `SliceTiming` is in the BIDS JSON sidecar, skips otherwise.

### Optional flags

Left to the executor or to per-dataset judgment.

- `--track-carbon`, `--stop-on-first-crash`, `--resource-monitor` — executor concerns.
- `--bids-filter-file` — per-dataset.

### Conditional flags (per-dataset)

These depend on per-dataset state and must be set at runtime, not as part of the recommended invocation.

- `--skull-strip-t1w {force,skip}` — `force` if the dataset has not been skull-stripped, `skip` if it has. Auto-detection is unreliable; both Joe and Felix gate this on manual verification (see pre-run gates above). **TODO:** standardize a machine-readable per-dataset metadata file recording skull-strip status so this can be set automatically.

## Pre-run gates

Before invoking fmriprep on any dataset:

1. **MRIQC must succeed first.** Both Joe and Felix gate fmriprep on a successful MRIQC run.
2. **Defacing + skull-strip status verified.** Joe maintains a Google Sheet tracking which datasets have been manually checked for face presence (defacing) and prior skull-stripping. Refer to it before any run; if the dataset isn't listed, get it checked first.
   - **TODO:** add link to Joe's spreadsheet here once shared.
   - **TODO:** consider whether parts of this verification can be automated (face detection, intensity-distribution heuristics for skull-stripped vs not).

## Output dataset naming

Per BIDS derivatives spec (clarified by Yarik):

```
<dataset-id>_<pipeline>-<flavor>
```

- **Underscore** separates entities; **dash** separates pipeline name from flavor.
- **`+`** chains flavors.
- Don't bake executor (BABS / reproman / bash-heredoc / etc.) into the name — different executors should produce machine-precision-identical output for the same fmriprep config.

Examples:

```
ds005374_fmriprep-25.1.4              # the fmriprep derivative (all stages share this)
ds005374_freesurfer-7.3.2             # FreeSurfer subjects-dir, shipped as subdataset
                                      # mounted under ds005374_fmriprep-25.1.4/sourcedata/freesurfer/
ds005374_fmriprep-25.1.4+austin1      # deliberate flavor variant (alternate config)
```

**Stages are not separate datasets.** Each stage (anat-only → minimal → resample → optional full) is a sequential `datalad run` commit on the same `ds<id>_fmriprep-<version>` dataset. Consumers fetch the parts they want via `datalad get` patterns and git-annex `wanted` expressions (see "Distribution" TODO below).

> **TODO:** confirm the structural choice — single derivative dataset with sequential `datalad run` commits, FreeSurfer as nested subdataset under `sourcedata/freesurfer/`. Felix's Stage 1 invocation produces both kinds of outputs; we just commit them to two different datasets.

## Versions and provenance

- **fmriprep version**: latest stable at time of run. Encoded in the dataset name so multiple versions coexist.
- **FreeSurfer version**: whatever fmriprep's container bundles.
- **Templates**: come from TemplateFlow; whatever version fmriprep pulls. Document in `dataset_description.json`.

## Open questions (block on Chris)

1. **`--level resample` semantics** — break into three sub-questions:
   1. Does it write the `_desc-confounds_timeseries.tsv` (motion params, FD, WM/CSF, aCompCor, etc.)? *Felix says yes; we believe yes with high confidence.*
   2. Does it write resampled 4D BOLD to disk in any space — T1w-native, template, or both? *Felix believes no; uncertain. This is the load-bearing question for whether Stage 3 is a sufficient "good default."*
   3. What's the on-disk storage delta vs `--level minimal`? If (b) is "no," delta should be small (just adding the confounds TSV). If (b) is "yes," the delta is large.
2. **Storage delta** between stages 2 → 3 → 4 in concrete bytes — rough numbers for one typical subject + run.
3. **`--use-syn-sdc warn`** — keep it (Joe) or drop it (Felix)?

## Open questions (block on us)

- Naming for the staged subdatasets (stage as flavor suffix vs. separate entity).
- Whether the FreeSurfer subdataset is its own top-level dataset or a subdataset of the fmriprep derivative.
- Workflow for the manual defacing/skull-strip review — how to make Joe's verification scale beyond one person.

## Related work

- Felix's bootstrap pipeline: https://cerebra.fz-juelich.de/f.hoffstaedter/bootstrap_fMRIprep
- Joe's recent reproman-driven runs: see any `OpenNeuroDerivatives/ds*-fmriprep` repo with `.reproman/jobs/local/`
- BABS: https://github.com/PennLINC/babs
- DataLad ReMake (Felix exploring): for shipping minimal derivatives + recomputing resample on demand
- fmriprep docs: https://fmriprep.org/en/stable/usage.html
- BIDS derivatives spec: https://bids-specification.readthedocs.io/en/stable/derivatives/

## Contributors

- **Austin Macdonald** (Dartmouth) — current maintainer of this doc; runs mechababs over OpenNeuro at scale
- **Felix Hoffstaedter** (FZ Jülich) — long-running OpenNeuro preprocessing on cerebra
- **Joe Wexler** — OpenNeuroDerivatives org, prior + current fmriprep runs there
- **Yarik Halchenko** (Dartmouth) — BIDS naming, datalad infrastructure, META coordination
- **Chris Markiewicz** (Stanford / fmriprep) — pending input on the open questions
