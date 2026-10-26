# Data-Efficient-Training-and-Adaptive-Reasoning-in-Small-Transformers
# How Little Data Does a Small Model Need to Reason?

### Data-Efficient Training and Adaptive Reasoning in Small Transformers

A research project on how small Transformer models learn to reason from limited data, using Sudoku as the reasoning task. Progress, decisions and results are recorded in the [research log](LOG.md).

## Problem statement

Small models can learn to reason well in a narrow domain, but it is not known how few original training examples they actually need. Large language models reason well across many domains because they use very large models trained on very large datasets. In many real domains, such as medicine, law or science, original training examples are rare or expensive to collect.

Recent work shows that tiny recursive models can solve hard Sudoku and maze puzzles from about 1,000 original examples. However, these models also rely on heavy data augmentation: for Sudoku, each original puzzle is turned into 1,000 transformed copies. It is therefore unclear how much of their performance comes from the original data, from augmentation, or from the model design.

Augmented copies keep the same underlying logic as the original puzzle, while new original examples add new reasoning patterns. It is not known how many augmented copies are worth one new original example, whether harder or more diverse examples reduce the data needed, or below which dataset size a model stops reasoning and starts memorizing.

This research studies how the size, source and selection of training data affect reasoning in a small Transformer. The model is trained from scratch in one domain, Sudoku, so all of its reasoning ability comes from the training data.

The model also uses adaptive reasoning. Instead of a fixed number of layers, it improves its answer over several reasoning steps and learns when to stop, so that easy puzzles use few steps and hard puzzles use more. This makes it possible to study whether extra reasoning at test time can make up for less training data.

The aim is to find the minimum number of original examples needed for good-enough reasoning, and to measure how much data strategies and adaptive reasoning lower that minimum. These findings will be used to build a small reasoning model that performs well with as little data as possible.

## Research questions

**Core question:** What is the minimum number of original training examples a small Transformer needs to reason well in one domain, and how much can data strategies and adaptive reasoning reduce that minimum?

| # | Focus | Question |
|---|---|---|
| RQ1 | Size | How does reasoning performance change as the number of original examples decreases, and at what size does the model start memorizing instead of reasoning? |
| RQ2 | Source | At the same total amount of data, how do new original examples compare with augmented copies? How many copies are worth one original example? |
| RQ3 | Selection | Does choosing examples by difficulty, or ordering them from easy to hard (curriculum), reduce the number of original examples needed? |
| RQ4 | Model | Does a looping model that reasons in steps need less data than a one-pass Transformer? |
| RQ5 | Adaptive reasoning | Does a model that decides how many steps to use spend more steps on harder puzzles, and can extra reasoning steps make up for fewer training examples? |

## Why Sudoku

- Answers can be checked automatically
- Difficulty can be measured (each puzzle has a rating)
- Large public datasets exist, so new original examples are easy to get
- Augmentation that keeps the same logic is possible (swap digits, swap rows, rotate)
- Models are trained from scratch, so all reasoning ability comes from the training data

## Approach

The work runs in stages. Each stage changes one thing while everything else stays fixed, so every result has a clear cause.

| Stage | What | Kept fixed | Status |
|---|---|---|---|
| 1 | Data pipeline and augmentation | | Done |
| 2 | Baseline: one-pass Transformer | Data | Done |
| 3 | Build the reasoning model (looping), choose the best design | One dataset (10,000 puzzles) | Next |
| 4 | Data study: size, augmentation, selection, curriculum (RQ1–RQ3) | The reasoning model | Planned |
| 5 | Adaptive reasoning: learned stopping, steps vs. difficulty (RQ5) | Best data recipe | Planned |
| 6 | Compare with the baseline and an LLM (RQ4) | Same test set | Planned |

All decisions are made on the validation set. The test set is used only for final results.

## Notebooks

| Notebook | What it does |
|---|---|
| `01_sudoku_data.ipynb` | Loads 3M puzzles, samples 111,000, converts them to numbers, checks every solution, groups puzzles into easy / medium / hard, and saves a balanced test set (3,000), a validation set (10,000) and a training pool (98,000) |
| `02_augmentation.ipynb` | Augmentation that keeps puzzles valid (swap digits, swap rows within a band, swap bands, rotate), checked on 5,000 copies, and builds 7 training sets for the first experiments |
| `03_baseline.ipynb` | A small one-pass Transformer (about 806k parameters): a memorization sanity check and the first training runs |

## Data

Source: [3 Million Sudoku Puzzles with Ratings](https://www.kaggle.com/datasets/radcliffe/3-million-sudoku-puzzles-with-ratings) (Kaggle).

The data files (`.npz`) are not stored in this repository. Running notebooks 01 and 02 recreates them exactly, because all random choices use a fixed seed (42).

| Split | Size | Notes |
|---|---|---|
| Test | 3,000 | Balanced: 1,000 easy, 1,000 medium, 1,000 hard. Never used for training |
| Validation | 10,000 | Random mix, used for all decisions during the work |
| Pool | 98,000 | All training sets are drawn from here |

Difficulty groups: easy = rating 0, medium = rating above 0 and up to 2, hard = rating above 2.

## Results so far

**Sanity check:** trained on 10 puzzles, the baseline memorizes them (100% solved) but is at chance level on new puzzles (11.4% of empty cells, where random guessing gives 11.1%).

**Baseline, same model and 10,000 training steps:**

| Training data | Train cells | Validation cells | Gap | Puzzles solved |
|---|---|---|---|---|
| 10,000 puzzles | 68.9% | 53.4% | 15.5 | 0% |
| 98,000 puzzles | 57.6% | 57.5% | 0.1 | 0% |

*Cells = share of empty cells predicted correctly. Solved = all 81 cells correct.*

**What this shows:**
- With too little data, the model memorizes instead of reasoning.
- More data removes memorization and helps a little (53.4% to 57.5% of cells), but a one-pass model still solves 0% of puzzles.
- Solving a full puzzle needs a long chain of deductions. A model with 4 layers that answers every cell at once cannot follow long chains. This is why the reasoning model is built next, before the data study.

## How to run

1. Open the notebooks in Kaggle or Google Colab.
2. Run `01_sudoku_data.ipynb`, then `02_augmentation.ipynb`, to create the data files.
3. Run `03_baseline.ipynb` with a GPU turned on.

## Key references

- Jolicoeur-Martineau (2025). [Less is More: Recursive Reasoning with Tiny Networks](https://arxiv.org/abs/2510.04871)
- Wang et al. (2025). [Hierarchical Reasoning Model](https://arxiv.org/abs/2506.21734)
- Palm et al. (2018). [Recurrent Relational Networks](https://arxiv.org/abs/1711.08028)
- Dehghani et al. (2018). [Universal Transformers](https://arxiv.org/abs/1807.03819)
- Banino et al. (2021). [PonderNet: Learning to Ponder](https://arxiv.org/abs/2107.05407)
