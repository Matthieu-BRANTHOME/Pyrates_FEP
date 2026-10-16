# Pyrates_FEP

Analysis pipeline for the interaction traces collected with [Pyrates](https://py-rates.fr),
as part of the experiment on LLM-generated feedback (CONTROL, FREE and CONSTRAINED conditions).

```
campaign/
  1_pilot/          pilot annotation campaign: coding form and results
  2_full/           full campaign: game assignment, coding form and results
  3_adjudication/   adjudication of the experts' disagreements: choices handed to the platform, gold standard
  4_llm/            LLM annotation: context files, rendered prompt, raw responses and results
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
  06_adjudication.ipynb             adjudication campaign and gold standard
  07_llm_annotation.ipynb           annotation of the B/C feedback by frontier LLMs, evaluation against the gold standard
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

**API keys (notebook 07 only)**: create a `.env` file at the project root (it is git-ignored)
with `ANTHROPIC_API_KEY`, `OPENAI_API_KEY` and `TYPESAFE_API_KEY`. Never write a key in a notebook.

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
| `campaign/2_full/full_form.pdf` | readable bilingual (EN/FR) rendering of `full_form.json`, for review outside the platform |
| `campaign/2_full/full_result.csv` | full campaign annotation results, one row per (game, trace, expert), input of notebooks 05, 06 and 07 |
| `campaign/3_adjudication/adjudication_result.csv` | adjudication campaign results (standard FEP export), one row per (game, trace, arbiter), input of notebook 06 |
| `campaign/4_llm/platform_context.json` | export of the FEP webtool: trace labels, details panel rules, FEP level sheets and memo, feedback rendering rules |
| `campaign/4_llm/game_rules.json` | Pyrates game rules (execution outcomes, control functions, line limits), written for the LLM prompt |
| `campaign/4_llm/level_maps.json` | ASCII maps of the 8 levels with randomness and dynamics, checked against the game's Tiled maps |
| `campaign/4_llm/game_interface.json` | student-side texts of the game (level panel, programming memo, interface, assistant panel), extracted from the Pyrates repository |

`data/input/imported/` is versioned so that the notebooks can be replayed from a fresh clone.
`data/input/cleaned/` and `data/output/` are not: both are rebuilt by running the notebooks.

| Output | Produced by | Role |
| --- | --- | --- |
| `data/input/cleaned/cleaned_interaction_traces.csv` | notebook 01 | shared input of notebooks 02 and 03 |
| `data/output/expert_assignment.csv` | notebook 02 | working output, not reused downstream |
| `data/output/plateau_feedback_sample.csv` | notebook 03 | working output, not reused downstream |
| `campaign/2_full/full_campaign_assignment.csv` | notebook 04 | assignment handed to the annotation platform |
| `campaign/3_adjudication/adjudication_assignment.csv` | notebook 06 | games to adjudicate, one semicolon-separated list per arbiter |
| `campaign/3_adjudication/adjudication_input.csv` | notebook 06 | the two experts' codings of the games to adjudicate (extract of `full_result.csv`), handed to the platform |
| `campaign/3_adjudication/gold.csv` | notebook 06 | gold standard, one row per feedback, input of notebook 07 |
| `campaign/4_llm/prompt_<version>.md` | notebook 07 | rendered prompt and sample contexts, for review |
| `campaign/4_llm/llm_result.csv` | notebook 07 | LLM annotations in the FEP export format |
| `campaign/4_llm/few_shot_examples.csv` | notebook 07 | trace ids of the few-shot examples of the prompt |
| `campaign/4_llm/dev_scores.csv` | notebook 07 | dev-set agreement of each prompt version |

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

## 3.6/ Notebook 06: adjudication and gold standard

Inputs `campaign/2_full/full_form.json`, `campaign/2_full/full_campaign_assignment.csv`,
`campaign/2_full/full_result.csv` and, once the adjudication campaign is coded,
`campaign/3_adjudication/adjudication_result.csv`. Each question on which the two experts of a
feedback disagree becomes a **choice** between their two answers. The two arbiters (`Penny` and
`Shaquille`, the pilot experts) adjudicate every choice on the FEP platform: seeing both original
codings (anonymized as expert A and expert B), the arbiter keeps, for each question in disagreement,
the complete answer of one of the two experts, without mixing them. The arbiter never writes free text: a kept answer comes with the "other"
text of its expert; for a question in agreement, the two experts' "other" texts are joined by ` / `.

A feedback enters the gold standard when every question has a label (expert agreement, or the same
answer reached by both arbiters); if the arbiters reach different answers on any question, or an
answer is neither expert's answer, the feedback is left out.

Disagreement rule, shared with the platform: questions are compared one by one; for the
multiple-choice questions, the comparison is on the set of ticked codes (order does not matter);
the free texts of the "other" options are not compared.

FEP assigns whole games. Two files go to the platform:

- `adjudication_assignment.csv`: one row per arbiter, with every game holding at least one choice as a
  semicolon-separated list (`expert_id`, `n_games`, `game_ids`), as for a regular campaign;
- `adjudication_input.csv`: the two experts' codings of every feedback of these games, an extract of
  `full_result.csv` in the same format.

The adjudication campaign is exported in the standard FEP format (one row per (game, trace, arbiter),
one column per question, complete coding) as `adjudication_result.csv`. The notebook checks that every
adjudicated answer is one of the two experts' answers, reports the agreement between arbiters per
question (same answer kept), and writes `campaign/3_adjudication/gold.csv` (one row per feedback,
one column per question, `source` = `experts` or `adjudication`).

## 3.7/ Notebook 07: LLM annotation

Inputs `data/input/cleaned/cleaned_interaction_traces.csv`, `campaign/2_full/full_form.json`, the
four context files of `campaign/4_llm/`, `campaign/2_full/full_result.csv` and
`campaign/3_adjudication/gold.csv` (notebook 06).
Annotates the B/C feedback with frontier LLMs (Claude Opus, GPT Sol, Jev), each treated as one
more annotator. For every feedback, the prompt rebuilds what an expert sees on the FEP platform
(same trace window, labels and details, feedback cleaned like the platform), plus the game
knowledge the experts have (rules, maps, level dynamics, student interface and memo). The games coded
by the experts are split by game into an example pool (few-shot examples, with their gold labels),
a dev set and a test set; on the test set, the agreement of each model with the experts is compared
with the agreement between experts (Krippendorff's alpha, as in notebook 03), and per-category F1
is computed against the gold standard.

`DRY_RUN = True` (default) makes no API call: the notebook renders the contexts, writes
`campaign/4_llm/prompt_<version>.md` for review and estimates the cost. Raw responses are cached in
`campaign/4_llm/raw/<model>.jsonl` (key: model, prompt version, trace), so a rerun makes no new call;
results go to `campaign/4_llm/llm_result.csv` in the FEP export format (`expert_id` = model id).

Workflow once the gold standard exists: tune the prompt on the dev set (`RUN_SPLITS = ["dev"]`,
`EVAL_SPLIT = "dev"`, bump `PROMPT_VERSION` at each change; the scores of every version are kept in
`campaign/4_llm/dev_scores.csv`), freeze it, annotate and evaluate the test set once
(`RUN_SPLITS = ["test"]`, `EVAL_SPLIT = "test"`), then annotate the production set with the batch
functions.

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
