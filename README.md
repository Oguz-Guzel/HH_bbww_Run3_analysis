![CMS Run 3](https://img.shields.io/badge/CMS-Run%203-blue)

# HH -> WWbb Run 3 analysis

Analysis of the Higgs boson pair-production channel
`HH -> WWbb` with the CMS Run 3 dataset. The workflow is
implemented with the [bamboo analysis framework](https://bamboo-hep.readthedocs.io/).
It produces cutflows, plots, skims, scale factors, and inputs for machine-learning
studies.

## Repository layout

| Path | Purpose |
| --- | --- |
| `python/` | bamboo analysis modules and skimmers |
| `config/` | Analysis, sample, DAS, and batch configuration |
| `data/` | Scale factors, corrections, and other analysis inputs |
| `scripts/` | Utilities for post-processing and systematic studies |

## Prerequisites

- A CMS software environment with CVMFS access (for example, lxplus)
- Python and a fresh bamboo installation
- Access to CMS datasets on the grid
- A valid CMS grid proxy for grid or batch jobs

Install bamboo using its
[fresh-install instructions](https://bamboo-hep.readthedocs.io/en/latest/install.html#fresh-install).
Then install the additional dependencies:

```sh
git clone https://gitlab.cern.ch/cp3-cms/CMSJMECalculators.git
pip install ./CMSJMECalculators
pip install correctionlib
```

Clone this repository next to the bamboo installation:

```sh
git clone https://github.com/Oguz-Guzel/HH_bbww_Run3_analysis.git
cd HH_bbww_Run3_analysis
```

## Environment setup

Run these commands whenever starting a new shell on lxplus or another machine
with CVMFS:

```sh
source /cvmfs/sft.cern.ch/lcg/views/LCG_105/x86_64-el9-gcc11-opt/setup.sh
source <path-to-bamboo>/bamboovenv/bin/activate
export PYTHONPATH="${PYTHONPATH}:${PWD}/python"
```

Before reading files from the grid or submitting batch jobs, create a CMS
VOMS proxy:

```sh
voms-proxy-init --voms cms -rfc --valid 192:00
export X509_USER_PROXY="$(voms-proxy-info -path)"
```

If jobs cannot access grid files, store the proxy at a persistent path:

```sh
voms-proxy-init --voms cms -rfc --valid 192:00 \
  --out "$HOME/private/gridproxy/x509"
export X509_USER_PROXY="$HOME/private/gridproxy/x509"
```

The repository includes `config/cern.ini` for CERN batch execution. Pass it
with `--envConfig` as shown below, or copy its contents to
`~/.config/bamboorc` to make it the default bamboo configuration.

## Running the analysis

### Cutflow and plots

Run the cutflow analysis for either the 2022 or 2023 configuration. Add
`--maxFiles 1` for a quick smoke test using one file per sample.

```sh
bambooRun -m python/cutflowAnalysis.py \
  config/<2022-or-2023>_v12.yml \
  -o ./outputDir/ \
  --envConfig config/cern.ini \
  --distributed driver
```

The configured batch backends include HTCondor, Dask, and Spark. Adjust the
backend and job settings in `config/cern.ini` when needed.

### Machine-learning evaluation

Use the skims in the bamboo results directory as input to an ML workflow.
After producing an ONNX model, evaluate it with:

```sh
bambooRun -m python/mvaEvaluator.py \
  config/<2022-or-2023>_v12.yml \
  -o ./outputDir/ \
  --envConfig config/cern.ini \
  --distributed driver \
  --mvaModel <path-to-model.onnx>
```

Use `--DY_CR` or `--TT_CR` to evaluate the model in the DY or ttbar control
region, respectively.

### Synchronization skims

Produce a synchronization skim with:

```sh
bambooRun -m python/syncSkimmer.py \
  config/2022_v12_sync.yml \
  -o output/syncTest \
  --sync
```

## Scale factors and systematic inputs

If a workflow change can affect the dilepton trigger efficiency, regenerate the
trigger scale factors:

```sh
bambooRun -m python/trigger_eff.py:TriggerEff \
  config/<2022-or-2023>_v12.yml \
  -o ./outputDir/
python scripts/trg_eff_to_json.py --bamboo_output ./outputDir/
```

If a change can affect b-tagging, regenerate the b-tagging reweighting files
and copy the resulting files from the output directory into `data/`:

```sh
bambooRun -m python/btagReweighting.py \
  config/<2022-or-2023>_v12.yml \
  -o ./outputDir/
```

Compute LHE scale-sum weights for muRF variations with:

```sh
python scripts/compute_lhe_scale_sumw.py \
  --years 2022 \
  --max-files 3 \
  --workers 16 \
  --output data/LHEScaleSumw_2022.yaml
```

Replace `2022` with `2023` when producing the corresponding Run 3 input.
