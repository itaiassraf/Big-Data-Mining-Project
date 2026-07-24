# Behind the Byline: A Large-Scale Study of Scientific Author Contributions 

## Overview

This repository contains the complete computational workflow used to extract, normalize, validate, analyze, and model author contribution statements from **Nature-family journals** and **PLOS Journals**.

The workflow converts heterogeneous contribution disclosures into structured author–task records and standardized **CRediT** roles. It then uses the resulting author-level dataset to study contribution distributions, author-position patterns, disciplinary differences, recurring role combinations, and the predictability of self-declared contribution categories.

The repository is organized as a sequence of five notebooks:

```text
Publisher data
    ↓
Contribution and metadata extraction
    ↓
Author-initials resolution and author–task assignment
    ↓
Rule-based and few-shot CRediT classification
    ↓
Harmonized author-level dataset
    ↓
Descriptive analysis, figures, frequent-role bundles, and prediction
```

> **Naming note.** The manuscript and current figures use the label **PLOS Journals**. Some notebook names, variables, and intermediate filenames retain the earlier labels `PLOS ONE` or `OnePlos` for backward compatibility with the original processing pipeline.
>
> The misspelled source column `contribtuion type` is also retained in several notebooks because it is part of the original data schema. Corrected downstream columns use names such as `contribution_type` or `contribution type correction`.

---

## Repository contents

| Execution order | Current notebook | Suggested repository name | Main responsibility |
|---:|---|---|---|
| 1 | `Nature_OnePlos_Extract_Author_Contribution (1).ipynb` | `01_extract_and_enrich_data.ipynb` | Extract contribution statements, authors, affiliations, and article metadata; enrich PLOS records with OpenAlex fields. |
| 2 | `author_initials_mapping_rules_Plos_Nature.ipynb` | `02_map_authors_and_tasks.ipynb` | Resolve initials and abbreviated author forms to full names; extract Nature author–task relations; report ambiguous mappings. |
| 3 | `credit_assignment_rules_and_fewshot (1).ipynb` | `03_assign_credit_roles.ipynb` | Map free-text contribution tasks to the 14 CRediT roles using deterministic rules and a controlled few-shot embedding layer. |
| 4 | `Analysis_and_Results_notebook.ipynb` | `04_analysis_and_results.ipynb` | Build the harmonized author-level dataset and reproduce the descriptive analyses, figures, Pareto fits, heatmaps, and frequent role bundles. |
| 5 | `predict_contribution_categories (1).ipynb` | `05_predict_contribution_categories.ipynb` | Evaluate leakage-safe multi-label prediction of self-declared CRediT roles. |

---

## Data sources

### Nature-family journals

Nature-family contribution data are represented as JSON files containing article-level arrays such as:

```python
data[source_key]["title"][article_index]
data[source_key]["year"][article_index]
data[source_key]["url"][article_index]
data[source_key]["contribution"][article_index]
data[source_key]["authors"][article_index]
```

The extraction workflow can also collect contribution pages and full author lists from Nature journal pages. Author names are later searched inside the complete contribution paragraph so that local author–task relations can be reconstructed.

### PLOS Journals

PLOS data are extracted from an XML corpus. Two disclosure formats are handled:

1. **Structured author roles**, where contribution roles are attached directly to each author.
2. **Free-text contribution statements**, where task descriptions and abbreviated author names appear together in a contribution paragraph.

The workflow creates separate article-level tables for these two formats before mapping them into a shared author-level representation.

### OpenAlex metadata

PLOS article records are enriched with OpenAlex metadata, including:

- OpenAlex work identifier;
- DOI;
- topic;
- subfield;
- field;
- domain;
- exact or fuzzy match type;
- title-match score.

Matching is performed by normalized title and publication year. Exact matches are attempted first; unmatched records are optionally processed with fuzzy title matching using a high cutoff.

---

## Units of analysis

The notebooks use several related data levels. Distinguishing them is important when interpreting counts and output files.

| Data level | One row represents | Typical use |
|---|---|---|
| Article level | One article containing lists of authors and tasks | Initial extraction, duplicate selection, article metadata. |
| Author level | One author in one article | Author position, team size, total author task count. |
| Author–task level | One original contribution task assigned to one author | Rule-based CRediT classification and validation. |
| Atomic-action level | One action internally extracted from an original task | Multi-action classification and few-shot processing. |
| Author–category level | One author–CRediT occurrence after exploding role lists | Positional distributions, heatmaps, frequency analysis. |

The original task text is preserved even when an internal atomic-action representation is created.

---

# Notebook documentation

## 1. `Nature_OnePlos_Extract_Author_Contribution (1).ipynb`

### Purpose

This notebook creates the initial article-level datasets used by the rest of the project. It supports both Nature-family web/JSON data and PLOS XML files, and it enriches the extracted PLOS records with OpenAlex subject metadata.

### Main operations

#### Nature-family extraction

- Retrieves journal pages with `requests` and `BeautifulSoup`.
- Collects article URLs and contribution sections by topic.
- Extracts complete author lists from article pages.
- Uses retries, request delays, progress summaries, and error files.
- Produces updated JSON files containing the extracted author metadata.

#### PLOS XML extraction

- Opens the PLOS XML archive.
- Extracts article title and publication year.
- Extracts contribution statements from contribution footnotes.
- Extracts full author names and affiliation/address information.
- Detects whether contribution information is represented as structured roles or as free text.
- Creates separate DataFrames for the two PLOS contribution formats.

#### OpenAlex enrichment

- Downloads or reuses a local OpenAlex cache.
- Normalizes article titles.
- Matches by exact normalized title and year.
- Applies high-cutoff fuzzy matching only to remaining unmatched records.
- Adds topic, subfield, field, domain, DOI, and match diagnostics.
- Saves both CSV and Parquet versions.

This notebook establishes data provenance and creates a common metadata foundation. It separates the two PLOS disclosure formats rather than forcing them into one representation prematurely, and it attaches disciplinary information needed for domain-level analyses.

---

## 2. `author_initials_mapping_rules_Plos_Nature.ipynb`

### Purpose

This notebook connects initials or abbreviated author mentions to the correct full author names and then reconstructs author–task relations.

The same alias-generation and collision-resolution framework is used for both PLOS and Nature data. The difference is where the observed alias is found:

- In PLOS contribution data, abbreviated forms are already stored in the `authors` field.
- In Nature data, aliases are detected inside the full contribution paragraph.

### Author-alias rules

The mapping system includes:

- accent, punctuation, spacing, and hyphen normalization;
- initials from all name components;
- first-and-last initials;
- omission of middle names;
- alternative name orders;
- East Asian name-order variants;
- hyphenated and compound given names;
- first initial plus full surname;
- full given name plus surname initial;
- initials plus full surname;
- surname particles such as `van`, `de`, and `von`;
- full-name matching;
- permutations of two to four name components;
- splitting of accidentally concatenated aliases;
- common-word filtering so ordinary words are accepted only when written as uppercase initials;
- removal of declarations such as “All authors…”;
- collision resolution based on specificity and longer uniquely observed aliases.

Ambiguous initials are not forced to a single author. They are written to an explicit failure/conflict output.

### PLOS mapping stage

For every article:

1. Parse the `authors` and `full name` list columns safely with `ast.literal_eval`.
2. Flatten the observed author forms into one unique initials set.
3. Resolve each unique normalized initial once.
4. preserve all original zero-based full-name indices;
5. collapse duplicate occurrences of the same full name into one readable row;
6. write mapping success, mapping rule, ambiguity, and failure diagnostics.

### Nature author–task extraction stage

For each contribution paragraph:

1. Flatten the article's full author list.
2. Generate aliases for every author.
3. detect aliases and retain their character spans;
4. resolve collisions and overlapping matches;
5. group adjacent author mentions into author blocks;
6. inspect text before and after each author block;
7. detect `authors → task` and `task → authors` constructions;
8. assign the extracted task to every resolved author in the block;
9. preserve clauses marked with `respectively` or `needs_review`;
10. aggregate raw assignments into one row per article–author.

Commas are not treated as automatic task boundaries. Multi-action and multi-role interpretation is handled downstream in the CRediT-classification notebook.


This notebook is the link between article-level disclosure text and individual author records. Its diagnostic files make author resolution auditable and prevent uncertain initials from silently contaminating the downstream CRediT analysis.

---

## 3. `credit_assignment_rules_and_fewshot.ipynb`

### Purpose

This notebook converts free-text contribution tasks into the 14 standardized CRediT roles through a staged hybrid framework.

The design follows a conservative principle:

> A semantic embedding prediction is considered only when the deterministic system has not already produced a confident assignment.

### Deterministic classification stages

The decision order is:

1. Explicit CRediT labels.
2. High-precision text rules.
3. Special statements and non-task noise detection.
4. Dependency-based action–object rules.
5. Nominal phrase rules.
6. Contextual corrections.
7. Unresolved status and controlled transfer to few-shot classification.

Examples of deterministic relations include:

- `analyze + data/results` → **Formal Analysis**
- `collect + data/samples` → **Investigation**
- `provide + samples/equipment` → **Resources**
- `write + manuscript/paper` → **Writing – Original Draft**
- `provide + funding/grant` → **Funding Acquisition**

The contextual correction layer prevents common false positives, including:

- passive phrases such as “under the supervision”;
- training a model versus supervising people;
- comments or feedback without manuscript context;
- analysis-plan preparation versus completed analysis;
- literature review versus manuscript editing;
- generic or malformed statements that do not identify a specific contribution.

### Action-aware representation

An original task is preserved as written. When the grammar clearly contains several actions, the notebook creates an internal atomic-action representation.

For example:

```text
collected and analyzed the data
```

is represented internally as:

```text
collected the data
analyzed the data
```

This allows the two actions to receive different roles while retaining the original statement in the task-level output. Repeated roles are also preserved at the action level.

### Few-shot embedding stage

Only unresolved atomic actions are processed semantically.

- Embedding model: `sentence-transformers/all-MiniLM-L6-v2`
- Embeddings and category centroids are normalized.
- Examples may be frequency weighted.
- Each role receives a category-specific threshold derived from leave-one-out example behavior.
- Acceptance uses similarity and score-margin requirements.
- The output records the top category, top score, second category, score margin, threshold, and nearest example.

The few-shot layer does not overwrite deterministic assignments.

This notebook provides the core H-Contrib normalization layer. It combines transparent deterministic evidence with a restricted semantic fallback and retains enough provenance to audit every final assignment.

---

## 4. `Analysis_and_Results_notebook.ipynb`

### Purpose

This notebook creates the harmonized author-level analysis dataset and reproduces the descriptive results and figures.

It can be run in two ways:

1. Run the complete preparation workflow from the source-specific intermediate files.
2. Upload a prepared `all_author_tasks_df.csv` and begin directly at the **Analysis** section.

### Data preparation

The notebook:

- expands article-level structured-role records into one row per author;
- expands PLOS initials–task groups;
- joins initials with full author names;
- joins task text with final CRediT assignments;
- aggregates Nature task-level output to one author row;
- calculates article-level task quantities;
- calculates author position and team size;
- normalizes author position as

```text
author_index / (number_of_authors - 1)
```

for multi-author articles;

- maps Nature source fields into four broad research domains;
- selects the most complete article version when duplicate title–year records exist;
- standardizes source-specific column names;
- combines Nature and PLOS data into one author-level DataFrame.

### Main intermediate and final tables

| Output | Description |
|---|---|
| `authors_with_tasks_roles_df.csv` | One row per author from the structured PLOS Roles source. |
| `plos_one_authors_tasks_final1.csv` | One row per mapped PLOS free-text contribution author. |
| `nature_author_tasks_final.csv` | One row per Nature author with final CRediT assignments. |
| `all_author_tasks_df.csv` | Harmonized author-level dataset used by the analysis and prediction notebooks. |

### Analyses

The notebook includes:

- article counts by year;
- article counts by journal and year;
- normalized task share by author position;
- first-, middle-, and last-author comparisons;
- domain-specific author-position profiles;
- journal-specific author-position profiles;
- article-level maximum-to-minimum task ratios;
- linear and logarithmic fits by team size;
- contribution profiles by exact team size;
- team-size comparisons by journal and research domain;
- CRediT-category distributions along the author list;
- journal-level CRediT frequency comparisons;
- journal-by-domain heatmaps;
- Pareto-style cumulative contribution curves and fit diagnostics;
- frequent author-level role bundles using Apriori association mining;
- corpus totals and average article-level task quantities.

### Main figure outputs

Depending on the executed sections, the notebook writes files such as:

- `{journal}_normalized_tasks_by_domain.png`;
- `{journal}_pareto_by_domain.png`;
- `pareto_fit_results_by_domain.csv`;
- `max_min_task_ratio_by_author_count.png`;
- `normalized_tasks_by_exact_team_size.png`;
- `{journal}_team_sizes_by_domain.png`;
- journal/domain CRediT position figures as PDF and 1200-DPI PNG;
- `credit_categories_by_journal.png`;
- journal-specific heatmaps as PDF and 1200-DPI PNG.

This notebook transforms the pipeline outputs into the statistical summaries and visual evidence reported in the article. It also centralizes the duplicate-selection logic and author-position normalization used across analyses.

---

## 5. `predict_contribution_categories.ipynb`

### Purpose

This notebook evaluates whether self-declared CRediT roles can be predicted from article and authorship metadata.

It is designed specifically to prevent article-level leakage and to address reviewer concerns about model evaluation, class imbalance, and target-derived predictors.

### Input structure

Each row must represent one author in one article.

| Type | Columns used |
|---|---|
| Target | `contribution_type` |
| Categorical predictors | `field`, `domain` |
| Numeric predictors | `number_of_authors`, `normalized_author_position` |
| Group identifier | DOI/article ID when available; otherwise title, year, and journal |
| Descriptive only | `number_of_tasks` |

`number_of_tasks` is excluded from the predictors because it is derived from the target labels.

### Experimental design

- Fixed random seed: `42`.
- Held-out test proportion: `20%`.
- Five-fold grouped cross-validation inside the training set.
- Article-level multi-label stratification.
- All authors from one article remain in the same partition.
- Imputation, scaling, and one-hot encoding are fitted inside each training fold.
- Per-role thresholds are selected from out-of-fold training predictions only.
- Hyperparameters are selected using average precision rather than accuracy.
- The held-out test labels are used only for final evaluation.

### Models

- One-vs-rest logistic regression with balanced class weights.
- One-vs-rest XGBoost with fold-specific positive-class weighting.

### Reported metrics

The notebook reports:

- precision;
- recall;
- F1;
- average precision;
- ROC AUC where defined;
- Category threshold comparisons;
- micro and macro multi-label metrics;
- per-role support and prevalence.


---

# Key columns in the harmonized dataset

The exact set of columns may vary slightly by source, but the following fields are central to the analysis.

| Column | Meaning |
|---|---|
| `title` | Article title. |
| `year` | Publication year. |
| `journal` | Harmonized source label. |
| `field` | OpenAlex or source field. |
| `domain` | Broad research domain. |
| `full_name` | Resolved full author name. |
| `initials` / `matched_aliases` | Observed abbreviated form or resolved aliases. |
| `author_index` | Zero-based position in the article's author list. |
| `number_of_authors` | Article team size. |
| `normalized_author_position` | Author position scaled from 0 for first author to 1 for last author. |
| `tasks` | Original extracted free-text task or task list. |
| `contribution_type` | Final CRediT role list attached to the author. |
| `number_of_tasks` | Number of task/role assignments attached to the author row. |
| `article_tasks` / `article_number_of_tasks` | Article-level task quantity used by the relevant analysis section. |
| `article_instance_key` | Stable source-specific identifier for one Nature article instance. |
| `credit_assignment_method` | Rule, explicit label, nominal phrase, dependency rule, few-shot, or unresolved. |
| `credit_send_to_few_shot` | Whether deterministic processing left the action unresolved. |
| `needs_review` | Quality-control indicator for uncertain extraction or mapping. |

Multi-value text fields in the mapping outputs commonly use a readable ` | ` separator. List-valued columns saved by pandas may appear as Python-list strings and are parsed with `ast.literal_eval`.

---

# Installation

The notebooks were developed for Python and Google Colab. A local environment can be created with:

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip

pip install \
  pandas numpy scipy matplotlib scikit-learn mlxtend \
  spacy sentence-transformers openpyxl \
  iterative-stratification xgboost joblib \
  beautifulsoup4 requests rapidfuzz tqdm nltk pyarrow
```

Install the English spaCy model:

```bash
python -m spacy download en_core_web_sm
```

The rule system can fall back to a blank English tokenizer, but dependency-based action–object rules are less informative without the full spaCy model.

---

# Reproduction workflow

## Full pipeline

1. Configure data paths and the OpenAlex API key in the extraction notebook.
2. Run `Nature_OnePlos_Extract_Author_Contribution (1).ipynb`.
3. Run `author_initials_mapping_rules_Plos_Nature.ipynb`.
4. Inspect `initials_mapping_failures.csv`, Nature alias conflicts, and `needs_review` records.
5. Run `credit_assignment_rules_and_fewshot (1).ipynb`.
6. Inspect unresolved tasks and `few_shot_predictions.csv`.
7. Run `Analysis_and_Results_notebook.ipynb` to create `all_author_tasks_df.csv` and reproduce figures.
8. Run `predict_contribution_categories (1).ipynb` using the harmonized author-level file.

## Analysis-only reproduction

To reproduce only the final descriptive analyses:

1. Upload `all_author_tasks_df.csv`.
2. Open `Analysis_and_Results_notebook.ipynb`.
3. Begin at the **Analysis** section.

## Prediction-only reproduction

1. Upload `all_author_tasks_df.csv`.
2. Confirm the target column is `contribution_type`.
3. Review the configuration cell.
4. Run all cells in `predict_contribution_categories (1).ipynb`.

---

# Reproducibility and audit files

The workflow intentionally retains intermediate evidence rather than exporting only the final labels.

| Audit artifact | What it allows a reviewer to inspect |
|---|---|
| `initials_mapping_failures.csv` | Unmapped and ambiguous author initials. |
| `nature_alias_conflicts_*.csv` | Competing full-name candidates for an observed alias. |
| `nature_article_summary_*.csv` | Extraction completeness and article-level review status. |
| `actions_credit_final.csv` | The exact atomic action used for every CRediT decision. |
| `few_shot_predictions.csv` | Similarity, threshold, score margin, and nearest example. |
| `mapping_summary.json` / `nature_run_summary.json` | Corpus-level quality-control counts. |
| `split_manifest.csv` | Proof that articles do not cross train/test partitions. |
| `test_predictions_<model>.csv` | Independent inspection of final model predictions. |
| `best_parameters_<model>.json` | Reproduction of selected model settings. |

---

# Methodological safeguards

The code includes several safeguards that are important for interpreting the study:

- uncertain initials are not silently assigned;
- general declarations and standalone noise are not treated as scientific tasks;
- commas do not automatically create separate tasks;
- the original contribution text is retained;
- multi-action tasks can receive several roles;
- deterministic assignments are not overwritten by the embedding stage;
- only unresolved actions reach the few-shot classifier;
- category-specific semantic thresholds are saved;
- duplicate article versions are selected using explicit completeness criteria;
- first and last author positions are derived from the original ordered author list;
- predictive splits are performed at the article level;
- preprocessing and threshold selection are restricted to training folds;
- target-derived task counts are excluded from the predictive features;
- the models predict **reported** contribution labels, not verified labor or causal effects.

---

# Known limitations

- Contribution statements are self-reported and may reflect journal templates, disciplinary conventions, or selective disclosure.
- Web pages and publisher interfaces can change after the data-collection date.
- Title-based OpenAlex matching may remain uncertain for duplicated or substantially altered titles.
- Alias generation is intentionally broad, but some uncommon abbreviations, typographical errors, transliterations, consortium authors, or incomplete names may remain unresolved.
- A `respectively` construction can require manual review when the text does not provide a safe one-to-one alignment.
- Few-shot results depend on the quality and diversity of the verified examples.
- The semantic model identifies similarity to documented CRediT examples; it does not establish ground truth independently of the disclosure text.
- Several notebooks use Google Colab paths under `/content/`. Users running locally must change the configuration cells.
- Raw publisher data should be redistributed only in accordance with the applicable source terms. The repository can still provide code, derived tables, validation files, and exact instructions for rebuilding the processed data.

---

# Recommended repository layout

The notebooks currently reference several Colab paths. For a public release, the following structure makes the workflow easier to navigate:

```text
.
├── README.md
├── notebooks/
│   ├── 01_extract_and_enrich_data.ipynb
│   ├── 02_map_authors_and_tasks.ipynb
│   ├── 03_assign_credit_roles.ipynb
│   ├── 04_analysis_and_results.ipynb
│   └── 05_predict_contribution_categories.ipynb
├── data/
│   ├── raw/
│   ├── interim/
│   ├── processed/
│   └── validation/
├── models/
│   └── credit_centroid_model/
├── outputs/
│   ├── figures/
│   ├── tables/
│   ├── diagnostics/
│   └── predictive_model/
└── requirements.txt
```

Large raw archives and model files may be stored in a versioned external repository, while this Git repository retains the notebooks, small audit tables, documentation, and stable download instructions.

---

## Scope of the repository

This repository is intended to support:

- transparent reconstruction of the data-processing pipeline;
- inspection of author-name and task-mapping decisions;
- reproduction of the article's descriptive figures and tables;
- evaluation of the hybrid CRediT classification framework;
- reuse of the processed author-level data for further scientometric research.
