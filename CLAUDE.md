# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A radio-astronomy pipeline for extracting lightcurves from interferometer
data, implemented as a [Stimela](https://stimela.readthedocs.io) recipe. It is
**almost entirely declarative YAML**, not Python — the pipeline is a graph of
Stimela steps that invoke external "cabs" (containerised tools: wsclean,
QuartiCal, breizorro, smops, tricolour, CASA) run through a Singularity backend.
There is essentially no application code to edit; work here means editing recipe
YAML, cab definitions, and parameter files.

The recipe is tailored for MeerKAT L1 visibilities averaged to 1024 channels,
assumes 1GC calibration has already been applied, and picks up from the standard
self-cal loop. It does not handle field selection — the target must already be
extracted.

## Commands

The pipeline runs from **inside the `lightcurve_pipeline/` directory** (paths in
the recipe includes are relative to it):

```
cd lightcurve_pipeline
stimela run lightcurve-pipeline.yml lightcurve-pipeline ms-path=path/to/ms outdir-path=path/to/outputs -pf configs/meerkat-l-band.yml
```

- Stimela validates the recipe and its inputs before every run. Add `-d` after
  `run` (dry run) to validate without running anything. `stimela -C` clears
  Stimela's config cache; it does not validate.
- `-pf <file>` layers a parameter file; repeat `-pf` to stack multiple.
- Add `-s <step>` to run a single step, or `--skip-existing`-style behaviour is
  already baked in via each step's `skip_if_outputs` field.

Installation (Python 3.10–3.12 — pinned `>=3.10, <3.13`). `uv` is required, not
optional: bare `pip` ignores `[tool.uv.sources]` and the install will fail.

```
uv venv   # Honours requires-python when selecting an interpreter.
source .venv/bin/activate
uv pip install -e .
```

Several dependencies (`stimela`, `cult-cargo`, `breifast-tron`) resolve from
**git repos over SSH** (see `[tool.uv.sources]` in `pyproject.toml`), each
tracking a branch rather than a pinned revision: `master` for `stimela` and
`breifast-tron`, and `r0.3.0` for `cult-cargo`. `breifast-tron` is
not public — SSH access to `ratt-ru/breifast` is required, and it is not on PyPI
at all, so a `pip install` cannot fall back to a release. There is no `.venv` in
the repo; create one before installing.

There are no tests, linters, or build steps configured beyond the hatchling
package build.

## Architecture

The entrypoint recipe `lightcurve-pipeline.yml` pulls everything together via
`_include` and defines the ordered `steps`. The pieces:

- **`lightcurve-pipeline.yml`** — top-level recipe. Declares pipeline `inputs`
  (`band`, deep-imaging params like `scale`/`weight`/`size`, `smops-nchan`, and
  threading knobs) and the `steps` list. `assign` sets up derived paths
  (`image-prefix`, `tempdir-path`, log dir).
- **`lightcurve-cabs.yml`** — includes cult-cargo cab definitions (wsclean,
  breizorro, quartical, taql, smops) and defines two inline `python-code` cabs
  (`add-imaging-columns`, `debug-print`).
- **`lightcurve-steps.yml`** — reusable base step templates under `lib.steps`
  (notably `lib.steps.wsclean.base`), pulled into steps via `_use`. These are
  effectively inlined and can read recipe-level variables, so they are **not**
  safely reusable across other recipes.
- **`flagging/meerkat_1k_basic.yml`** — the `meerkat-target-flagging`
  subrecipe (Oxkat-derived), invoked as a nested recipe from the
  `basic-flagging` step. `assign_based_on: band` selects per-band RFI/flag
  ranges (UHF, L, S0–S4). Tricolour strategy config lives in
  `flagging/meerkat_target_narrow.tri.yml`.
- **`(breifast.recipes)tron-pfb.yml`** — the external transient-imaging /
  lightcurve-extraction recipe (named `tron`), invoked by the
  `transient-imaging` step. This is the actual lightcurve producer and comes
  from the private `breifast` package.

### Pipeline flow (the `steps` order)

`basic-flagging` (nested flagging subrecipe, backs up flags first) →
`flag-reset` → `flagsummary-1` → `add-columns-1` (adds imaging columns via
python-casacore) → `image-1` (shallow wsclean to build a mask) → `mask-1`
(breizorro) → `image-2` (masked wsclean) → `smops-1` (upsamples the image-2
model in frequency) → `predict-1` (wsclean predict into MODEL_DATA) →
`selfcal-1` (QuartiCal delay+offset solve, writes CORRECTED_DATA) →
`flagsummary-2` → `image-3` (image CORRECTED_DATA) → `smops-2` → `predict-2`
(deep model into MODEL_DATA) → `transient-imaging` (tron-pfb: subtracts
MODEL_DATA from CORRECTED_DATA, then high-time-resolution imaging + lightcurve
extraction).

Steps reference each other's outputs with Stimela substitutions
(`=recipe.<input>`, `=previous.<output>`, `{steps.<name>.<field>}`). Preserving
these cross-step column/output references is essential when editing.

### Configuration

`configs/meerkat-<band>-band.yml` (UHF, L and S0–S4) are the parameter files
passed via `-pf`. Each sets `band`, the deep-imaging params, and
`transient-imaging.*` keys, which set inputs of the tron-pfb recipe (among them
`primary-beam-band`, which names UHF as `U`). Stimela silently ignores unknown
keys in a parameter file, so check new keys against tron-pfb's own inputs.

To add a band, add a parameter file, extend the `band` choices in both
`lightcurve-pipeline.yml` and the flagging subrecipe, and give the flagging
subrecipe a block for it under `assign_based_on`.

## Conventions

- The pipeline is driven entirely through the `stimela` CLI; there is no Python
  entrypoint.
- Threading is layered: the general `threads` input is the fallback, and the
  more specific `imaging-threads` / `calibration-threads` / `breifast-threads`
  inputs override it where set (via `=recipe.<specific> or recipe.threads`).
