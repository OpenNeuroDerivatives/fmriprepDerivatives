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
| `--use-syn-sdc warn` | SyN-based SDC fallback when no fieldmap is present; log when used. **TODO: see open questions** |
| `--md-only-boilerplate` | Skip PDF render of methods text. Markdown is sufficient and faster. |
| `--skip-bids-validation` | Assume input is validated upstream (in our MRIQC gate). |
| `--notrack` | Disable telemetry. |
| `--level` | Picks pipeline stage. See pipeline shape above. |

### Flags that should not be set

- `--ignore slicetiming` — leave unset. fmriprep is data-driven: it corrects if `SliceTiming` is in the BIDS JSON sidecar, skips otherwise. **TODO: see open questions**

### Conditional flags (per-dataset)

These depend on per-dataset state and must be set at runtime, not as part of the recommended invocation.

- `--skull-strip-t1w {force,skip}` — `force` if the dataset has not been skull-stripped, `skip` if it has. Auto-detection is unreliable; both Joe and Felix gate this on manual verification (see pre-run gates above). **TODO: see open questions**

### Optional flags

Left to the executor or to per-dataset judgment.

- `--track-carbon`, `--stop-on-first-crash`, `--resource-monitor` — executor concerns.
- `--bids-filter-file` — per-dataset.
- `--me-output-echos` — only when planning to run `tedana` (or other TE-dependent denoising) downstream. No-op on single-echo data; on multi-echo data it ships per-echo BOLD, which otherwise just duplicates the raw echoes at storage cost. tedana is not a set-and-forget step (it needs per-dataset human inspection of the ICA), so this is off by default.

## Pre-run gates

Before invoking fmriprep on any dataset:

1. **MRIQC must succeed first.** Both Joe and Felix gate fmriprep on a successful MRIQC run.
2. **Defacing + skull-strip status verified.** Joe maintains a Google Sheet tracking which datasets have been manually checked for face presence (defacing) and prior skull-stripping. Refer to it before any run; if the dataset isn't listed, get it checked first. **TODO: see open questions**

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

1. **`--level resample` semantics** What gets written by this step?
   1. Does it write the `_desc-confounds_timeseries.tsv`
   2. Does it write resampled 4D BOLD to disk in any space?
2. **Storage delta** between stages 2 → 3 → 4 in concrete bytes — rough numbers for one typical subject + run.
3. **`--use-syn-sdc warn`: fall back to SyN-based SDC when no fieldmap is present?** When a fieldmap is present in the BIDS dataset, fmriprep uses it by default. Many OpenNeuro datasets lack fieldmaps, so the choice is between approximate SDC everywhere or no SDC on those subjects. Sub-questions:
   1. Is approximate SyN-SDC better than no SDC for typical downstream analysis on OpenNeuro datasets?
   2. Does mixing fieldmap-SDC, SyN-SDC, and no-SDC outputs introduce comparability artifacts that downstream consumers need to be aware of?
   3. Is the `warn` log level sufficient for consumers to know per-subject which SDC method was used, or should this surface in `dataset_description.json` or a sidecar?
4. **Slice timing correction (STC): when is it baked in, and is there a consumer escape hatch?** Leaving `--ignore slicetiming` unset means fmriprep applies STC when `SliceTiming` is in the BIDS sidecar.
   1. At what stage is STC baked into shipped outputs?
   2. Are the shipped spatial transforms still valid against non-STC'd raw BOLD, or are they entangled with STC?
5. Workflow for the manual defacing/skull-strip review, how/where to record this, and how to make Joe's verification scale beyond one person.
6. DataLad remake special remote (Felix exploring): for shipping minimal derivatives + recomputing resample on demand

## Related work

- Felix's bootstrap pipeline: https://cerebra.fz-juelich.de/f.hoffstaedter/bootstrap_fMRIprep
- Joe's recent reproman-driven runs: see any `OpenNeuroDerivatives/ds*-fmriprep` repo with `.reproman/jobs/local/`
- fmriprep docs: https://fmriprep.org/en/stable/usage.html
- BIDS derivatives spec: https://bids-specification.readthedocs.io/en/stable/derivatives/
