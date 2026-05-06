# HPA Usage in Public GitHub Repositories

This repository contains the research artifacts used to mine, classify, and analyze Kubernetes Horizontal Pod Autoscaler (HPA) configuration files from public GitHub repositories.

The artifact is organized as a three-notebook pipeline. Each notebook can be executed independently when the required intermediate data files are already available in `data/`, but the recommended execution order is from Notebook 1 to Notebook 3.

## Repository Structure

```text
.
├── 1_mine_hpa_usage.ipynb
├── 2_classify_hpa_minimal_version.ipynb
├── 3_msr_repository_analysis_minimal_version.ipynb
├── data/
├── feature-models/
├── research/
├── utils.py
├── requirements.txt
├── LICENSE
└── README.md
```

### Main directories and files

| Path | Description |
|---|---|
| `1_mine_hpa_usage.ipynb` | Mines candidate HPA files from GitHub and prepares the raw/intermediate mining outputs. |
| `2_classify_hpa_minimal_version.ipynb` | Parses and classifies HPA manifests, extracts configuration fields, computes HFRS-related features, and generates classification figures. |
| `3_msr_repository_analysis_minimal_version.ipynb` | Performs repository-level MSR analyses, including activity, version distribution, HFRS, sensitivity checks, risk indicators, and temporal co-evolution. |
| `data/` | Curated CSV files required by the minimal notebooks. |
| `feature-models/` | Feature model diagrams and DOT files used to document HPA API-version capabilities. |
| `research/` | Supporting technical notes and documentation snapshots used during the study. |
| `utils.py` | Small utility functions used by the mining/classification workflow. |
| `requirements.txt` | Python dependencies required to run the notebooks. |

## Execution Modes

The artifact supports two execution modes.

### Mode A: Reuse the curated data files

This is the recommended mode for artifact evaluation and paper review. The curated CSV files are already available in `data/`, so reviewers can execute Notebooks 2 and 3 without re-mining GitHub.

Recommended order:

```text
2_classify_hpa_minimal_version.ipynb
3_msr_repository_analysis_minimal_version.ipynb
```

### Mode B: Rebuild the dataset from mining

Notebook 1 can be used to reproduce the mining stage. This mode requires GitHub API access and may be affected by API limits, search result limits, network conditions, and repository changes over time.

Recommended order:

```text
1_mine_hpa_usage.ipynb
2_classify_hpa_minimal_version.ipynb
3_msr_repository_analysis_minimal_version.ipynb
```

## GitHub Authentication for Notebook 1

Notebook 1 accesses the GitHub API to search and retrieve candidate HPA files from public repositories. Running it without authentication may quickly hit API rate limits. For this reason, create a local `.env` file in the repository root and define a GitHub personal access token in the `GITHUB_TOKEN` variable.

The `.env` file must not be committed to the repository. It contains a private credential and should remain only in the local or Colab runtime environment.

### 1. Create a GitHub personal access token

GitHub supports fine-grained personal access tokens and classic personal access tokens. GitHub recommends fine-grained tokens whenever possible because they provide more controlled permissions and repository access.

Recommended option for this artifact:

1. Open GitHub.
2. Go to **Settings**.
3. Open **Developer settings**.
4. Select **Personal access tokens**.
5. Prefer **Fine-grained tokens** when available for your use case.
6. Create a token with the shortest practical expiration date.
7. Use read-only permissions. This artifact only needs to read public repository metadata and file contents.
8. Copy the generated token immediately. GitHub only shows it once.

If a fine-grained token does not work for the API endpoints used by your environment, use a **Personal access token (classic)** with the minimum permissions required. For public repository mining, avoid broad write permissions. Do not grant unnecessary scopes.

### 2. Create the `.env` file

In the repository root, create a file named `.env`:

```text
GITHUB_TOKEN=ghp_replace_this_with_your_token
```

The resulting local structure should be:

```text
.
├── .env
├── 1_mine_hpa_usage.ipynb
├── 2_classify_hpa_minimal_version.ipynb
├── 3_msr_repository_analysis_minimal_version.ipynb
├── data/
├── requirements.txt
└── utils.py
```

### 3. Check whether `.env` is ignored

Before committing or packaging the artifact, confirm that `.env` is ignored by Git:

```bash
git status --ignored
```

The repository should include a `.gitignore` rule such as:

```text
.env
*.env
```

Never paste the token directly into a notebook cell, source file, markdown file, or committed configuration file.

### 4. Optional `.env.example`

For documentation, the repository may include a safe example file named `.env.example`:

```text
GITHUB_TOKEN=replace_with_your_github_token
```

This example file does not contain a real credential and can be committed.

### 5. Using the token in Google Colab

When running Notebook 1 in Google Colab, there are two safe options.

Option A, create `.env` inside the runtime:

```python
from pathlib import Path

Path(".env").write_text("GITHUB_TOKEN=ghp_replace_this_with_your_token\n")
```

Option B, define the environment variable directly for the current session:

```python
import os

os.environ["GITHUB_TOKEN"] = "ghp_replace_this_with_your_token"
```

Option B is useful for quick experiments, but the variable disappears when the Colab runtime is reset. Option A works better when the notebook expects `python-dotenv` to load credentials from `.env`.

### 6. Security notes

Treat the GitHub token like a password. If a token is accidentally committed, pasted into a shared notebook, or exposed in a public artifact, revoke it immediately in GitHub and create a new one.

For anonymous review, do not include a real `.env` file, token value, personal GitHub username, or original repository URL in the artifact package.

## Environment Setup

The notebooks were designed to run both locally and in Google Colab.

### Local execution

Create a Python virtual environment and install the dependencies:

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

Then open the notebooks with Jupyter or VSCode.

### Google Colab execution

Each minimal notebook includes a bootstrap cell at the beginning. The bootstrap prepares the execution environment by:

1. detecting whether the notebook is running in Google Colab or locally;
2. defining the project directory;
3. installing dependencies from `requirements.txt`;
4. validating the required files in `data/`;
5. creating output directories for generated figures and intermediate outputs.

By default, the Colab bootstrap uses:

```text
/content/hpa
```

This path is temporary and is recreated when the Colab runtime is reset. To persist generated outputs in Google Drive, define `HPA_PROJECT_DIR` before running the bootstrap cell:

```python
import os
os.environ["HPA_PROJECT_DIR"] = "/content/drive/MyDrive/Colab Notebooks/hpa"
```

For anonymous review, avoid using paths, URLs, or repository names that reveal author identity.

## Required Data Files

The minimal artifact expects the following files in `data/`.

### Required by Notebook 2

```text
phase4_latest.csv
phase3_file_commits_latest.csv
hpa_hfrs_full.csv
```

### Required by Notebook 3

```text
hpa_repo_aggregated.csv
phase4_latest.csv
phase3_file_commits_latest.csv
hpa_hfrs_full.csv
phase6_commit_timeline_latest.csv
phase6_coevolution.csv
```

Additional CSV files may be present in `data/` because they support intermediate analyses, validation checks, or reproducibility.

## Notebook 1: Mining HPA Usage

`1_mine_hpa_usage.ipynb` performs the mining stage. It searches public GitHub repositories for YAML files that contain Kubernetes HPA manifests, retrieves candidate files, and prepares raw/intermediate data for subsequent parsing and classification.

Main responsibilities:

1. configure GitHub API access;
2. search for candidate HPA YAML files;
3. retrieve repository and file metadata;
4. store intermediate mining outputs;
5. prepare data consumed by the classification stage.

Expected inputs:

```text
.env file with GITHUB_TOKEN
```

Expected outputs include intermediate mining files used to build later CSVs in `data/`.

## Notebook 2: HPA Classification and Configuration Analysis

`2_classify_hpa_minimal_version.ipynb` parses HPA manifests and extracts configuration-level information.

Main responsibilities:

1. load curated HPA file data;
2. normalize `apiVersion` and HPA fields;
3. classify HPA configuration choices;
4. compute and inspect HFRS-related fields;
5. analyze API-version adoption;
6. analyze selected configuration indicators, such as CPU thresholds and `minReplicas = maxReplicas`;
7. generate PDF figures used by the paper and artifact inspection.

Key outputs:

```text
figures and PDFs generated by the notebook
updated or validated classification tables
printed summary tables for the analyzed indicators
```

## Notebook 3: Repository-Level MSR Analysis

`3_msr_repository_analysis_minimal_version.ipynb` performs repository-level analyses over the classified HPA files.

Main responsibilities:

1. load repository-level and HPA-level curated datasets;
2. analyze repository activity and HPA stability;
3. compare active and outdated repositories;
4. analyze HFRS across repository activity groups;
5. evaluate sensitivity scenarios for API-version distributions and HFRS;
6. inspect risk indicators;
7. analyze HPA-touch frequency and temporal co-evolution.

Key outputs:

```text
PDF figures under the configured plots directory
summary tables printed by the notebook cells
risk-indicator and sensitivity-analysis summaries
```

## Generated Figures

The notebooks save plots as PDF files to avoid quality loss in paper figures. The exact output directory depends on the notebook bootstrap configuration. In the Colab-ready versions, the plots directory is defined in the initial setup cell.

## Reproducibility Notes

The artifact supports reproducibility at two levels.

1. **Analysis reproducibility:** run Notebooks 2 and 3 over the curated CSV files in `data/`.
2. **Pipeline reproducibility:** run Notebook 1 to rebuild the mining outputs, then run Notebooks 2 and 3.

The first level is recommended for review because GitHub mining is time-dependent and may be affected by repository deletions, updated files, API limits, search limits, and network behavior.

## License

This artifact is released under the MIT License. During double-anonymous review, the copyright holder is written as `Anonymous Authors` to avoid revealing author identity. If the paper is accepted, the copyright holder can be restored in the camera-ready artifact.
