# Pyrates_FEP

Analysis pipeline for the interaction traces collected with [Pyrates](https://py-rates.fr),
as part of the experiment on LLM-generated feedback (CONTROL, FREE and CONSTRAINED conditions).

```
data/
  imported/   raw exports, versioned
  cleaned/    datasets produced by the notebooks, not versioned
notebooks/
  01_data_cleaning.ipynb       trace cleaning and export in the webtool import format
  02_expert_assignation.ipynb  assignment of the games to the experts
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
| `data/imported/imported_interaction_traces.xlsx` | interaction traces, input of notebook 01 |
| `data/imported/imported_pre_test.xlsx` | pre-test |
| `data/imported/imported_post_test.xlsx` | post-test |

The raw exports are versioned so that the notebooks can be replayed from a fresh clone. The
datasets in `data/cleaned/` are not: they are rebuilt by running the notebooks.

# 3/ Execution

Execute the notebooks sequentially in numerical order (i.e., `01_*.ipynb`, `02_*.ipynb`, etc.),
either from VSCode with "Run All", or headless:

```bash
jupyter execute --inplace notebooks/01_data_cleaning.ipynb
```

## 3.1/ Notebook 01: trace cleaning

Input `data/imported/imported_interaction_traces.xlsx`, output
`data/cleaned/cleaned_interaction_traces.csv`, in the format expected by the webtool importer. The
notebook is self-contained and can be replayed at any time, with no access to the production
database.

What it does:

- **Repairs the `code` column**, and nothing else: text that Excel stored as a formula (a copy
  starting with `=`), mojibake from CP1252 punctuation mis-decoded into C1 control characters, and
  empty `COPIED` or `PASTED > CONTENT` captures, refilled from their counterpart in the same game
  when the attribution is unambiguous.
- **Adds session traces.** A game runs over two or three class periods, which the raw data does not
  mark. The notebook cuts wherever two consecutive traces are more than one hour apart, and frames
  each session with a `STARTED` and an `ENDED` trace whose `object_name` is `SESSION_1`,
  `SESSION_2` or `SESSION_3`.
- **Drops the games whose LLM assistant failed too often**, more than a third of the requests left
  unanswered, since replaying those measures the infrastructure rather than the feedback.
- **Validates the export** against the importer rules, then reads the written file back.

Two requirements on the importer side: the `SESSION_n` object names must exist in the database, and
timestamps carry milliseconds (`YYYY-MM-DD HH:MM:SS.SSS`), which the source keeps for every trace
and which the ordering depends on.

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
