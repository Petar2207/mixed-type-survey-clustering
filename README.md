# survey-segmentation-kmodes-lca

Survey segmentation pipeline comparing K-Modes/K-Prototypes with latent class analysis on mixed-type questionnaire data.

The notebook takes a raw survey export and a question-metadata file, prepares the data automatically based on the declared question types, then segments respondents across several thematic dimensions. For each dimension it fits both a distance-based model (K-Modes, K-Prototypes or K-Means, depending on the variable mix) and a latent class model, compares them, and picks a winner using an explicit rule rather than by eye.

## Why both methods

Survey data is rarely one type. A dimension may combine Likert scales, single-choice questions and multi-select items in the same block. K-Prototypes handles that mix directly but needs a `gamma` weight and an elbow judgement to choose `k`. Latent class analysis treats everything as categorical and selects `k` by BIC, which is more principled, but it can produce classes with poor separation. Running both and comparing them on the same variables makes the trade-off visible instead of hidden in a modelling choice.

## Pipeline

**1. Load and classify**
Reads the survey export and a question file with `QuestionID`, `Type`, `Options` and `Text`. Question types are read from the metadata, not guessed:

| Metadata type | Treated as |
|---|---|
| `Scale` with more than 2 options | ordinal |
| `Scale` with 2 or fewer options | binary |
| `SelectOne` | nominal |
| `SelectMultiple` | multi-select, expanded to dummies |
| `Text` | dropped |

**2. Clean**
Columns above a missingness threshold are dropped, as are free-text items. Multi-select answers are split on the separator into dummy columns, with an explicit no-selection flag so "answered nothing" stays a signal rather than becoming silent zeros.

**3. Impute**
Missingness is split into two cases. Structurally missing values (the question did not apply to that respondent) are filled with `not_applicable` and kept as a real category. Randomly missing values get mode imputation for categorical items and median imputation for ordinal ones. Ordinal columns are then standardised; categorical columns stay raw for the distance-based models.

**4. Cluster per dimension**
Each dimension is fitted twice:

- **Distance-based** — K-Prototypes for mixed blocks, K-Modes for purely categorical ones, K-Means where everything is numeric. `k` comes from elbow detection on the cost curve via `kneed`, with a fallback heuristic if it is unavailable.
- **Latent class** — every column label-encoded and passed to `StepMix` with a categorical measurement model. `k` is the BIC minimum across the scan range.

**5. Compare and decide**
Both solutions are scored on the same raw variables using Cramér's V, plus a normalised-entropy cluster balance measure and each model's native fit statistic. The decision rule: take the LCA solution when its relative entropy clears the floor (default 0.5), otherwise fall back to the distance-based model. The rule and the numbers behind it are printed for every dimension, so a choice can be overridden deliberately.

**6. Report**
`build_dimension_report()` prints cluster sizes, per-variable profiles and the comparison table for each dimension. Naming the clusters is handled separately — see below.

## Setup

Install the dependencies from `requirements.txt`: `pandas`, `numpy`, `scikit-learn`, `scipy`, `matplotlib`, `seaborn`, `kmodes`, `stepmix`, `kneed` and `openpyxl`.

## Usage

Two files are needed in the working directory:

- the survey export, one row per respondent, with columns named by question ID
- `question.xlsx`, the metadata, with `QuestionID`, `Type`, `Options` and `Text` columns, where `Options` is a separator-delimited list of code-and-label pairs

Configuration happens in two dictionaries near the top of the clustering section. `cluster_models` maps each dimension name to the list of question IDs that belong to it. `cluster_method_map` maps the same dimension names to the distance-based method to use — K-Prototypes for blocks that mix ordinal and categorical items, K-Modes for purely categorical ones, K-Means where everything is numeric.

With those set, the run-all cell loops over every dimension, calling `run_dimension()` and `build_dimension_report()` in turn and writing the winning labels back onto `df_proc`. A single dimension can also be run on its own, with the scan range, entropy floor and fallback `k` overridable per call.

## Naming the clusters

Clustering produces integers. Turning cluster 3 into something a stakeholder can act on is normally manual work: read a crosstab, squint at the percentages, invent a name, repeat for every cluster in every dimension. `build_combined_naming_prompt()` automates the tedious half of that.

It walks every dimension and builds one prompt containing, per cluster, the size and share, the mean of each ordinal variable, and the dominant categories of each categorical variable. Crucially, question IDs and option codes are decoded back to their original survey wording through `question_map` and `option_map`, so the prompt reads as real questions and answers rather than bare numeric codes. The requested output format is ready-to-paste Python: a short-name dict, a one-sentence-description dict and the mapping line for each dimension, so the result drops straight back into the notebook with no retyping.

The names it returns are a starting point, not a result. Read them against the profile output before adopting them, since a plausible-sounding label can paper over a cluster that is not actually distinct.

### Making the prompt sharper

Each variable line currently shows the top categories *within* a cluster. When one category dominates the whole sample, every cluster's top three look alike and the model hedges — descriptions like "the majority default profile" are the tell. Ranking by lift against the sample baseline instead makes the contrast explicit: divide each cluster's share for a category by that category's share across all respondents, so 1.0 means the cluster matches the sample average.

Printing both the share and the lift is what makes a label decisive. A 41% branch-visit rate against a 15% baseline reads as clearly branch-oriented; the raw 41% on its own reads as a minority and invites a vague name.

Two smaller adjustments in the same place: collapse multi-select dummy columns to the share of positives only, since printing the complement wastes prompt space, and truncate long question text to roughly 80 characters so six dimensions still fit in one prompt.

## Tuning notes

- **`gamma` (K-Prototypes)** — controls how much categorical dissimilarity counts against numeric distance. The default is derived from the data, but there is a dedicated cell that sweeps candidate values and reports the resulting Cramér's V per variable, which is a more useful check than cost alone.
- **`entropy_floor`** — raise it to be stricter about accepting LCA solutions; lower it to prefer LCA more often.
- **`k_range`** — widen it if the elbow or the BIC minimum lands at the edge of the current range, which usually means the true optimum is outside it.

## Data

No survey data is included in this repository. The `.gitignore` excludes spreadsheet and CSV files so respondent-level data is not committed by accident. Point the loading cell at a local copy of the export to run the pipeline.
