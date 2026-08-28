# Characterizing Horizontal Pod Autoscaler Usage in Open-Source Kubernetes Projects

Replication package for the SBES 2026 paper *Characterizing Horizontal Pod Autoscaler Usage in Open-Source Kubernetes Projects: A Large-Scale Empirical Study*

The package contains the mined dataset and the analysis notebooks used to study how Kubernetes Horizontal Pod Autoscaler (HPA) manifests are configured, maintained, and underutilized in public GitHub repositories. It reproduces the tables, figures, and statistics reported in the paper, including the HPA Feature Richness Score (HFRS), the configuration archetypes, and the six configuration smells.

**Accepted paper:** [paper/sbes2026-hpa.pdf](paper/sbes2026-hpa.pdf), included in the package, and available as an individual file in the version-specific Zenodo record at <https://zenodo.org/records/22134989/files/sbes2026-hpa.pdf>.

**Artifact DOI:** <https://doi.org/10.5281/zenodo.22134989>

## What is included

| Path | Description |
|---|---|
| `1_mine_hpa_usage.ipynb` | Mines candidate HPA files from GitHub and builds the raw mining outputs. Requires a GitHub token and takes several hours. |
| `2_classify_hpa_minimal_version.ipynb` | Parses and classifies HPA manifests, computes HFRS and the archetypes, and generates the classification figures. |
| `3_msr_repository_analysis_minimal_version.ipynb` | Runs the repository-level analyses: maintenance status, HFRS comparisons, sensitivity checks, configuration smells, and co-evolution. |
| `data/` | Curated CSV files that Notebooks 2 and 3 read. |
| `data/figures/notebook-2/` | PDF figures written by Notebook 2. |
| `data/figures/notebook-3/` | PDF figures written by Notebook 3. |
| `utils.py` | Helper functions used by the mining stage. |
| `requirements.txt` | Python dependencies. |
| `LICENSE` | MIT License. |

### Dataset files

| File | Rows | Content |
|---|---|---|
| `hpa_hfrs_full.csv` | 9,764 | One row per HPA manifest, with the parsed fields, the 13 HFRS components, the HFRS score and class, and the archetype label. |
| `phase4_latest.csv` | 16,506 | One row per metric definition, with the metric type, target type, and target value. |
| `hpa_repo_aggregated.csv` | 12,741 | One row per mined repository, with GitHub metadata, activity dates, and the toy-project and confidence indicators. |
| `phase3_latest.csv` | 9,795 | Repository and path metadata for every parsed candidate file, including the Helm templates later excluded. |
| `phase3_file_commits_latest.csv` | 9,770 | Most recent commit touching each candidate file, used to classify maintenance age. |
| `phase6_commit_timeline_latest.csv` | 8,926 | Commit timeline for the 169 repositories of the stratified co-evolution sample. |
| `phase6_coevolution.csv` | 241 | The 241 co-evolution events observed in the stratified sample of 169 repositories, which covers 70 active, 50 outdated, and 49 intermediate ones. |
| `hpa_hfrs.csv` | 9,764 | Reduced HFRS view kept for convenience. |

The numbers above are the corpus numbers reported in the paper: 12,741 repositories, 7,013 of them with at least one retained HPA file, 9,764 HPA manifests, and 16,506 metric definitions.

## Requirements

### Hardware

- Disk: the package occupies approximately 381 MiB after extraction, excluding Git metadata. Reserve at least 1 GB for the package, a virtual environment, and generated figures.
- Memory: the curated CSV files occupy about 42 MB once loaded, so 4 GB of RAM is sufficient.
- Network: needed only to install the dependencies, to download the commit-history archive on first run, and to run Notebook 1.
- No GPU, cluster, or Kubernetes installation is required. The study analyzes manifests as files and never deploys them.

### Software

- Python 3.12.13 is the primary and recommended environment for this artifact revision.
- Python 3.9.25, 3.10.20, and 3.11.15 were also validated as compatibility environments.
- The exact direct-dependency versions pinned in `requirements.txt` are shared by all four validated Python environments.
- Validated operating system: Linux x86_64. The notebooks use `pathlib`, and equivalent environment setup commands are provided below for macOS and Windows, but those systems were not part of this validation run.
- No Docker or virtual machine is required.
- A GitHub personal access token, required only by Notebook 1. Notebooks 2 and 3 reproduce the results of the paper without one. See [Notebook 1, the mining stage](#notebook-1-the-mining-stage) for how to create and configure it.

### Tested versions

Notebooks were executed in full in each environment below:

| Python | Notebook 2 | Notebook 3 | Generated figures | Result |
|---|---:|---:|---:|---|
| 3.9.25 | 25/25 code cells | 18/18 code cells | 11 + 2 PDFs | Passed |
| 3.10.20 | 25/25 code cells | 18/18 code cells | 11 + 2 PDFs | Passed |
| 3.11.15 | 25/25 code cells | 18/18 code cells | 11 + 2 PDFs | Passed |
| 3.12.13 | 25/25 code cells | 18/18 code cells | 11 + 2 PDFs | Passed |

All four environments used the same direct-dependency versions:

```text
jupyter 1.1.1        ipykernel 6.29.5        pandas 2.2.3
numpy 2.0.2           scipy 1.13.1             seaborn 0.13.2
matplotlib 3.9.4      pyyaml 6.0.2             requests 2.32.3
urllib3 2.2.3         python-dateutil 2.9.0.post0
python-dotenv 1.0.1
```

`requirements.txt` pins this validated direct-dependency set. Python or direct-dependency versions not listed above are outside the recorded validation scenarios.

### Validation scope

This artifact revision was validated on August 27, 2026, in the four Linux x86_64 environments above:

- Notebook 2 completed all 25 code cells and generated all 11 Notebook 2 figures in every Python environment.
- Notebook 3 completed all 18 code cells and generated both Notebook 3 figures and the expected result tables in every Python environment.
- The textual contents of all 13 generated PDFs were identical across the four Python environments.
- Notebook 1 was validated in the primary Python 3.12.13 environment up to its credential gate. Without `GITHUB_TOKEN`, it created the `.env` template and stopped with the documented message before making GitHub requests. Its complete mining stage was not rerun because it requires a token, approximately 27,600 API calls, and several hours.

## Installation

The steps below assume a terminal opened in the package root, which is the directory that contains `README.md`.

### 1. Create an isolated environment

On Linux or macOS:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

On Windows, using PowerShell:

```powershell
py -m venv .venv
.venv\Scripts\Activate.ps1
```

If PowerShell blocks the activation script, run `Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass` in the same session and try again.

### 2. Install the dependencies

```bash
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

This is the only installation step. There is nothing to compile and nothing to configure.

### 3. Check the installation

Run the command below. It loads the three datasets and prints the corpus numbers reported in the paper.

```bash
python -c "import pandas as pd; \
print('HPA files      :', len(pd.read_csv('data/hpa_hfrs_full.csv'))); \
print('metric defs    :', len(pd.read_csv('data/phase4_latest.csv'))); \
print('repositories   :', len(pd.read_csv('data/hpa_repo_aggregated.csv')))"
```

Expected output:

```text
HPA files      : 9764
metric defs    : 16506
repositories   : 12741
```

If these three numbers appear, the environment is ready. A `ModuleNotFoundError` means the dependencies were installed outside the active environment; activate it again and repeat step 2.

### 4. Open the notebooks

```bash
python -m jupyter lab
```

Jupyter opens in the browser. Visual Studio Code also works: open the package folder and select `.venv` as the kernel.

## Running the analysis

The recommended path for reviewers is to run Notebooks 2 and 3 over the curated CSV files. Notebook 1 rebuilds the corpus from GitHub and is not needed to reproduce the results.

### Notebooks 2 and 3, the reproducible path

Open and run all cells, in this order:

```text
2_classify_hpa_minimal_version.ipynb
3_msr_repository_analysis_minimal_version.ipynb
```

Both notebooks start with a bootstrap cell that detects whether the runtime is local or Google Colab, resolves the project directory, verifies the files under `data/`, and creates the output directories. In a local run with the dependencies already installed, the bootstrap changes nothing else.

With the data already expanded, both notebooks completed in under one minute in every validated environment. A clean bootstrap that also downloads and extracts the packaged data takes longer, and runtime varies with hardware and storage performance.

Expected results:

- 13 PDF figures, written to `data/figures/notebook-2/` by Notebook 2 and to `data/figures/notebook-3/` by Notebook 3. They include `fig03_temporal_commit_year.pdf`, `fig30_capability_adoption_over_time.pdf`, `fig_47a_configuration_profiles.pdf`, and `fig17a_hfrs_by_repository_status.pdf`, which appear in the paper.
- Summary tables printed by the cells, covering the API-version distribution, the HFRS classes, the archetypes, the maintenance status, and the six configuration smells.
- The statistics reported in the paper, including the median HFRS of 4 out of 13, the Mann-Whitney comparison between HPA-active and HPA-outdated repositories, and the smell counts.

### Notebook 1, the mining stage

Notebook 1 reruns the GitHub mining. It needs a personal access token, issues around 27,600 API calls, and runs for several hours. Its results also change over time, because repositories are created, edited, and deleted, and because GitHub Code Search caps each query at 1,000 results. Use it to inspect the mining procedure rather than to reproduce the exact corpus.

Notebooks 2 and 3 do not need a token. The only exception is the optional cell of Notebook 2 that rebuilds the manifest history from the original repositories, which the packaged `repos_history.zip` already provides.

#### Why a token is needed

Without authentication the GitHub API allows 60 calls per hour, which is not enough for any phase of the mining. A token raises the limit to 5,000 calls per hour. Notebook 1 also uses the GraphQL API, which rejects unauthenticated requests outright.

#### Which token to create

The notebook only reads public data: search results, repository metadata, file contents, and commit dates. It never writes anything.

- **Fine-grained token (recommended).** Go to *GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token*. Set a name and an expiration. Under *Repository access*, `Public Repositories (read-only)` is enough. No extra permission is required, because every fine-grained token already has read-only access to public repositories.
- **Classic token.** Go to *GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token (classic)*. Set a name and an expiration and **leave every scope unchecked**. A token with no scopes can already read public information and gets the 5,000 calls per hour limit.

Do not select the `public_repo` scope. It also grants write access to public repositories, which this artifact never needs.

Copy the token as soon as GitHub shows it, since the value is not displayed again.

#### Where to put it

Create a file named `.env` in the package root, which is the same directory as `README.md` and the notebooks. Notebook 1 reads `.env` from the current working directory, so it must sit next to the notebook you open, and not in a subfolder or in your home directory.

```text
GITHUB_TOKEN="github_pat_xxxxxxxxxxxxxxxxxxxxxxxx"
```

Notes on the format:

- the variable name must be exactly `GITHUB_TOKEN`;
- the quotes are optional, and surrounding spaces are stripped;
- a classic token starts with `ghp_`, and a fine-grained one with `github_pat_`. Both work.

If `.env` does not exist, the token cell creates a skeleton file, prints its full path, and stops with an error asking you to fill in the value. Fill it in and run the cell again. On success the cell prints how many characters the token has, and never prints the token itself.

`.env` is listed in `.gitignore`, so it is not committed. Do not redistribute it, and revoke the token on GitHub if it ever leaks.

#### Checking the token

The command below confirms that the token works and shows the remaining rate limit. Run it from the package root with the environment activated:

```bash
python -c "import os, requests; from dotenv import load_dotenv; load_dotenv('.env'); \
t = os.getenv('GITHUB_TOKEN','').strip(); \
r = requests.get('https://api.github.com/rate_limit', headers={'Authorization': f'token {t}'} if t else {}); \
print('status:', r.status_code); \
print('core limit:', r.json().get('resources',{}).get('core','(no data)'))"
```

A working token prints `status: 200` and a core limit of 5,000:

```text
status: 200
core limit: {'limit': 5000, 'remaining': 5000, ...}
```

A limit of 60 means the token was not read, so check the file name, the variable name, and the directory. A `status: 401` means the value is invalid, expired, or revoked.

## Storage, ethical and legal statements

The dataset contains metadata and configuration fields extracted from public GitHub repositories. It records repository names, file paths, commit dates, and the parsed contents of HPA manifests. It contains no personal data, no author identity, no credentials, and no repository source code beyond the HPA manifests themselves. Commit metadata is limited to dates and no author information is stored.

The mined repositories are public and each remains under its own license. This package redistributes derived data, that is, parsed configuration fields and computed metrics, together with the repository name and file path needed to trace each record back to its origin. Users who intend to redistribute manifest contents should check the license of the corresponding repository. The collection respected the GitHub REST API terms and its rate limits.

The `LICENSE` file applies to this package, which includes the notebooks, the helper code, the curated CSV files, and the figures.

## Reproducibility notes

The package supports two levels of reproducibility.

1. Analysis reproducibility, which reruns Notebooks 2 and 3 over the curated CSV files. The results are deterministic and match the paper.
2. Pipeline reproducibility, which reruns Notebook 1 and rebuilds the corpus. The mining stage depends on the state of GitHub at execution time, so the resulting corpus will differ from the one analyzed in the paper.

Every intermediate artifact of the mining stage is persisted as CSV, so the pipeline can be inspected phase by phase without repeating the API calls.

## Citation

```bibtex
@inproceedings{rodrigues2026hpa,
  author    = {Rodrigues, Thiago Serafim and Ribeiro J{\'u}nior, Humberto F. and Mendon{\c{c}}a, Nabor das Chagas},
  title     = {Characterizing Horizontal Pod Autoscaler Usage in Open-Source Kubernetes Projects: A Large-Scale Empirical Study},
  booktitle = {Proceedings of the 40th Brazilian Symposium on Software Engineering (SBES)},
  year      = {2026}
}
```

## License

This package is released under the MIT License. See `LICENSE`.
