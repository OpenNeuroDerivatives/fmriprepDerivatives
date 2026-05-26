# opinions for fmriprep runs in this superdataset

This repo seeds a curated datalad superdataset of fmriprep runs that follow the opinions documented here. Runs that meet these opinions live here; less-opinionated runs are welcome in [`OpenNeuroDerivatives`](https://github.com/OpenNeuroDerivatives) instead.

**Scope**: We cannot cover every consumer use case, we are aiming to produce data that will be useful for the "typical" case.

## Pipeline shape

We're standardizing on a **staged pipeline** rather than a single `fmriprep` invocation.

0. **MRIQC** (gate; should succeed before any fmriprep stage).
1. **`fmriprep --anat-only` + `--output-spaces`** — produces the full anatomical BIDS-derivatives scaffold (bias-corrected T1w, brain mask, segmentation, T1w↔MNI transforms in both NLin2009cAsym and NLin6Asym, surfaces, fsLR/fsaverage registration). Also produces the FreeSurfer subjects-directory, which is split out as a separate datalad subdataset mounted at `sourcedata/freesurfer/` so consumers can fetch it independently. Slow (~hours wall-clock, dominated by FreeSurfer's `recon-all`); low RAM; no BOLD touched.
2. **`fmriprep --level minimal`** — adds BOLD-side transforms (HMC, BOLD→T1w coreg, SDC). No confounds, no resampling, no BOLD outputs in any template space. Smallest fmriprep BOLD derivative.
3. **`fmriprep --level resample`** — the *intended* target stage: derive denoising **regressors** (motion params, framewise displacement, WM/CSF means, aCompCor components) into `_desc-confounds_timeseries.tsv`, without writing 4D BOLD in any template space. **Today this is byte-identical to `minimal`** — resample does not yet emit confounds; currently those come out only of `--level full`. fmriprep plans to add the confounds to resample (Discussed with Chris 2026-05-19). Until that lands we run `minimal` (Stage 2), which is identical, so we can upgrade in place once resample gains confounds.
4. **`fmriprep --level full`** (optional) — additionally writes the 4D resampled BOLD per output space (and, today, the confounds). ~5-8× storage cost over Stage 3. Skip unless explicitly needed.

**Stage outputs share one derivative dataset per (raw_dataset, fmriprep_version) pair.** Successive stages are sequential `datalad run` commits on the same dataset, not separate datasets. Within that, the **FreeSurfer subjects-directory** (under `sourcedata/freesurfer/` in the fmriprep derivative) is split out as its **own datalad subdataset** so consumers can fetch just the FreeSurfer side without the rest, or vice versa.

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

Stage 1 uses `--anat-only` instead of `--level`.

### Flags to include

| Flag | Why |
|---|---|
| `--output-spaces MNI152NLin2009cAsym:res-2 MNI152NLin6Asym:res-2` | Both modern (NLin2009cAsym) and HCP/AROMA-lineage (NLin6Asym) volumetric. `res-2` (2mm) is the de-facto fMRI analysis standard; `res-native` is a space-saving hack, not a recommendation. |
| `--cifti-output 91k` | HCP-standard grayordinate density. Implicitly pulls in fsLR + surface registration. |
| `--random-seed 12345` | Reproducibility. fmriprep auto-generates if unset; explicit value enables bit-identical reruns. |
| `--skull-strip-fixed-seed` | Reproducibility of skull-stripping specifically (Atropos has stochastic init). |
| `--use-syn-sdc warn` | SyN-based SDC fallback when no fieldmap is present; log when used. |
| `--me-output-echos` | No-op on single-echo data. On multi-echo data, ships per-echo BOLD so downstream TE-dependent denoising (e.g. `tedana`) stays possible. Kept on as a hedge — consumers may not run tedana but might want the option later; the cost is duplicated raw echoes on multi-echo datasets. |
| `--md-only-boilerplate` | Skip PDF render of methods text. Markdown is sufficient and faster. |
| `--skip-bids-validation` | Assume input is validated upstream (in our MRIQC gate). |
| `--notrack` | Disable telemetry. |
| `--level` | Picks pipeline stage. See pipeline shape above. |

### Flags that should not be set

- `--ignore slicetiming` — leave unset. fmriprep is data-driven: it corrects if `SliceTiming` is in the BIDS JSON sidecar, skips otherwise. **TODO: see open questions**

### Conditional flags (per-dataset)

These depend on per-dataset state and must be set at runtime, not as part of the recommended invocation.

- `--skull-strip-t1w {force,skip}` — `force` if the dataset has not been skull-stripped, `skip` if it has. fmriprep's built-in detection is unreliable, so this is gated on manual verification (see pre-run gates below). A reliable detector would let fmriprep choose automatically; there's upstream interest in adding one if a good algorithm turns up. **TODO: see open questions**

### Optional flags

Left to the executor or to per-dataset judgment.

- `--track-carbon`, `--stop-on-first-crash`, `--resource-monitor` — executor concerns.
- `--bids-filter-file` — per-dataset.

## Pre-run gates

Before invoking fmriprep on any dataset:

1. **MRIQC must succeed first.** fmriprep runs are gated on a successful MRIQC run.
2. **Defacing + skull-strip status verified.** A manually-maintained Google Sheet tracks which datasets have been checked for face presence (defacing) and prior skull-stripping. Refer to it before any run; if the dataset isn't listed, get it checked first. Operating on undefaced data we don't have permission to redistribute is not allowed, so an unverified dataset is a hard stop. **TODO: see open questions**

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
ds005374_fmriprep-25.2.5              # the fmriprep derivative (all stages share this)
ds005374_freesurfer-7.3.2             # FreeSurfer subjects-dir, shipped as subdataset
                                      # mounted under ds005374_fmriprep-25.2.5/sourcedata/freesurfer/
ds005374_fmriprep-25.2.5+austin1      # deliberate flavor variant (alternate config)
```

## Versions and provenance

- **fmriprep version**: pinned to the **25.2 LTS** series (currently 25.2.5). fmriprep cuts an LTS roughly every 5 years (25.2 is the first since 2022; next is expected ~30.2). Holding one version across the whole corpus keeps outputs comparable — fixing pipeline + version is a known reproducibility lever (cf. Milham et al. on pipeline-induced variability). 25.2 is also the latest release today. Encoded in the dataset name so multiple versions can still coexist.
- **FreeSurfer version**: whatever fmriprep's container bundles.
- **Templates**: come from TemplateFlow; whatever version fmriprep pulls. Document in `dataset_description.json`.

## Open questions

1. **Storage delta** between `minimal` and `full` in concrete bytes — rough numbers for one typical subject + run. (`minimal` and `resample` are identical today, so the only real delta is minimal → full.)
2. **`--use-syn-sdc warn`: fall back to SyN-based SDC when no fieldmap is present?** With a fieldmap, fmriprep applies it by default; many OpenNeuro datasets lack one, so the choice is approximate SDC everywhere vs. no SDC on those subjects. `minimal` only *outputs* the SDC warp, leaving application to the consumer — but the warp is nonlinear and cannot be un-applied once baked into resampled 4D (`full`), so the decision only bites when shipping `full`. Open:
   1. Is approximate SyN-SDC better than no SDC for typical downstream analysis on OpenNeuro datasets?
   2. Does mixing fieldmap-SDC, SyN-SDC, and no-SDC outputs introduce comparability artifacts consumers need to know about?
   3. Is the `warn` log level enough for consumers to know per-subject which SDC method was used, or should this surface in `dataset_description.json` or a sidecar?
3. **Slice timing correction (STC): when is it baked in, and is there a consumer escape hatch?** Leaving `--ignore slicetiming` unset applies STC when `SliceTiming` is in the sidecar. STC has **zero impact on `minimal`** (it only affects the coregistration reference), so it's moot for the current minimal target; it matters once we ship resampled BOLD. Current lean: follow the fmriprep default (apply when metadata present); simulations suggest the effect is negligible below ~0.5s TR. Open:
   1. At what stage is STC baked into shipped outputs?
   2. Are the shipped spatial transforms still valid against non-STC'd raw BOLD, or are they entangled with STC?
4. Workflow for the manual defacing/skull-strip review — how/where to record it, and how to scale it beyond a single reviewer.
5. DataLad remake special remote: for shipping minimal derivatives + recomputing resample on demand.

## Related work

- Felix's bootstrap pipeline: https://cerebra.fz-juelich.de/f.hoffstaedter/bootstrap_fMRIprep
- Joe's recent reproman-driven runs: see any `OpenNeuroDerivatives/ds*-fmriprep` repo with `.reproman/jobs/local/`
- fmriprep docs: https://fmriprep.org/en/stable/usage.html
- BIDS derivatives spec: https://bids-specification.readthedocs.io/en/stable/derivatives/
