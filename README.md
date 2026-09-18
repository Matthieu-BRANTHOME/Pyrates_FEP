# Pyrates_FEP

Analysis pipeline for the interaction traces collected with [Pyrates](https://py-rates.fr),
as part of the experiment on LLM-generated feedback (CONTROL, FREE and CONSTRAINED conditions).

```
campaign/
  1_pilot/          pilot annotation campaign: coding form and results
  2_full/           full campaign: game assignment, coding form and results
data/
  input/
    imported/        raw exports, versioned
    cleaned/          cleaned_interaction_traces.csv, produced by notebook 01, not versioned
  output/            working files produced by notebooks 02 and 03, not versioned
notebooks/
  01_data_cleaning.ipynb            trace cleaning and export in the webtool import format
  02_expert_assignation.ipynb       assignment of the games to the experts
  03_coding_grid_pretest.ipynb      pilot game selection, coding grid pretest, inter-rater analysis
  04_full_campaign.ipynb            full campaign: game assignment and coding handoff
  05_full_campaign_analysis.ipynb   full campaign: inter-rater reliability analysis
```

# 1/ Environment Setup

## 1.1/ Prerequisites
- VSCode with the "Jupyter" extension (Microsoft)
- Python 3.12
- Git

## 1.2/ Installation Steps

1/ **Clone the repository**
```bash
git clone https://github.com/Matthieu-BRANTHOME/Pyrates_FEP
cd Pyrates_FEP
```

2/ **Create a Python virtual environment**
```bash
python3.12 -m venv .venv
```

3/ **Activate the virtual environment**
```bash
# On macOS/Linux
source .venv/bin/activate

# On Windows
.venv\Scripts\activate
```

4/ **Install dependencies**
```bash
pip install -r requirements.txt
```

5/ **Configure VSCode**
- Open the project in VSCode
- Open any notebook (.ipynb file)
- Click on "Select Kernel" in the top right corner
- Choose the `.venv` interpreter

# 2/ Data

| File | Content |
| --- | --- |
| `data/input/imported/imported_interaction_traces.xlsx` | interaction traces, input of notebook 01 |
| `data/input/imported/imported_pre_test.xlsx` | pre-test |
| `data/input/imported/imported_post_test.xlsx` | post-test |
| `campaign/1_pilot/pilot_form.json` | coding form used for the pilot annotation, input of notebook 03 |
| `campaign/1_pilot/pilot_form.pdf` | readable bilingual (EN/FR) rendering of `pilot_form.json`, for review outside the platform |
| `campaign/1_pilot/pilot_result.csv` | pilot annotation results, one row per (game, trace, expert), input of notebook 03 |
| `campaign/2_full/full_form.json` | coding form used for the full campaign, referenced by notebook 04 |
| `campaign/2_full/full_result.csv` | full campaign annotation results, one row per (game, trace, expert), input of notebook 05 |

`data/input/imported/` is versioned so that the notebooks can be replayed from a fresh clone.
`data/input/cleaned/` and `data/output/` are not: both are rebuilt by running the notebooks.

| Output | Produced by | Role |
| --- | --- | --- |
| `data/input/cleaned/cleaned_interaction_traces.csv` | notebook 01 | shared input of notebooks 02 and 03 |
| `data/output/expert_assignment.csv` | notebook 02 | working output, not reused downstream |
| `data/output/plateau_feedback_sample.csv` | notebook 03 | working output, not reused downstream |
| `campaign/2_full/full_campaign_assignment.csv` | notebook 04 | assignment handed to the annotation platform |

# 3/ Execution

Notebooks `01`, `02` and `04` chain automatically, each consuming the previous one's output;
run them in numerical order, either from VSCode with "Run All", or headless:

```bash
jupyter execute --inplace notebooks/01_data_cleaning.ipynb notebooks/02_expert_assignation.ipynb notebooks/04_full_campaign.ipynb
```

## 3.1/ Notebook 01: trace cleaning

Input `data/input/imported/imported_interaction_traces.xlsx`, output
`data/input/cleaned/cleaned_interaction_traces.csv`, in the format expected by the webtool
importer. Repairs the `code` column (Excel formula artifacts, CP1252 mojibake, empty copy/paste
captures), adds synthetic session traces (a game spans several class periods, cut wherever two
consecutive traces are more than an hour apart), drops games whose LLM assistant failed too often,
and validates the export against the importer's rules. Self-contained, replayable at any time.

## 3.2/ Notebook 02: expert assignation

Input `data/input/cleaned/cleaned_interaction_traces.csv`, output
`data/output/expert_assignment.csv`. Assigns eligible `B`/`C` games (whole `game_id`, never a
subset) to pairs of experts, so that every game is annotated by exactly 2 experts, each expert
stays under their own workload cap, the `B`/`C` split stays balanced per expert and overall, and
the same duo is not reused beyond a set limit. Tunable parameters (expert list and caps,
eligibility thresholds, pairing limits, balance weights) sit together near the top of the
notebook, with the effect of any change reported in its checks section.

## 3.3/ Notebook 03: coding grid pretest

Input `data/input/cleaned/cleaned_interaction_traces.csv`. Three parts: (1) selects a pilot
sample of games around a learning plateau (reached level 5, never level 7), exported to
`data/output/plateau_feedback_sample.csv`; (2) two experts (`Penny`, `Shaquille`) code those games on
the FEP platform using `campaign/1_pilot/pilot_form.json`, producing
`campaign/1_pilot/pilot_result.csv`; (3) analyzes inter-rater reliability on that pilot (raw
agreement, Cohen's kappa, Krippendorff's alpha) per coding axis, closing with a synthesis of what
needs clarifying before the full annotation campaign.

## 3.4/ Notebook 04: full campaign, game assignment

Input `data/output/expert_assignment.csv` (notebook 02's output). The expert names it carries
(`Michael`, `Magic`, ...) are pseudonyms, same principle as the pilot's `Penny`/`Shaquille`, and
are kept as-is. Runs the same per-pair and per-game checks as notebook 02, and exports
`campaign/2_full/full_campaign_assignment.csv`, handed to the annotation platform alongside the
coding form `campaign/2_full/full_form.json`. Output of the coding step:
`campaign/2_full/full_result.csv`, analyzed in notebook 05.

## 3.5/ Notebook 05: full campaign, results analysis

Input `campaign/2_full/full_result.csv`. Same inter-rater reliability analysis as the pilot
(notebook 03, part 3), generalized to 8 experts: each trace is annotated by one of several
possible expert pairs rather than always the same two, so the pairing is derived per trace
instead of assumed fixed. Not runnable until the full campaign's coding is complete.

# 4/ Development Guidelines

## 4.1/ Adding Dependencies
When installing new packages (in venv):
```bash
pip install <package-name>
pip freeze > requirements.txt
```

## 4.2/ Before Committing

**Always perform these checks:**

1/ **Verify all notebooks run successfully**
   - Restart kernel and run all cells in each notebook
   - Ensure no errors occur

2/ **Clear all outputs and metadata**
   - In venv terminal type:
```bash
nbstripout notebooks/*.ipynb
```
