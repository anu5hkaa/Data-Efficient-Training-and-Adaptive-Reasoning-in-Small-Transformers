
# Research Log

What was done, every decision and why, and the results. Newest entries at the top.

---

## 2026-10-27 — Baseline model

### Done
- Built a small one-pass Transformer: digit embedding + position embedding, 4 Transformer layers, 9 output scores per cell. About 806k parameters.
- Training: AdamW, learning rate 0.0005, batch size 64, cross-entropy loss over all 81 cells.
- Evaluation: share of **empty** cells correct, and share of puzzles **fully** solved (all 81 cells), also per difficulty group.

### Results

**Sanity check (memorize 10 puzzles, 1,000 steps):**

| | Empty cells correct | Puzzles solved |
|---|---|---|
| The 10 training puzzles | 100% | 100% |
| New puzzles (validation) | 11.4% | 0% |

The code works (it can memorize), but with 10 puzzles the model learns nothing general: 11.4% is chance level (random guessing = 1 in 9 = 11.1%).

**Same model, 10,000 steps, two data sizes:**

| Training data | Train cells | Val cells | Gap | Solved |
|---|---|---|---|---|
| 10,000 puzzles | 68.9% | 53.4% | 15.5 | 0% |
| 98,000 puzzles | 57.6% | 57.5% | 0.1 | 0% |

- With 10,000 puzzles, validation stopped improving around step 7,000 while training kept rising: the model started memorizing.
- With 98,000 puzzles, there is no gap (no memorizing), and validation was still rising at step 10,000.
- Neither solves any full puzzle. To solve a puzzle, all ~55 empty cells must be right, which needs about 99% cell accuracy.

### Decisions
| Decision | Why |
|---|---|
| Train for a fixed number of **steps**, not epochs | An epoch depends on dataset size, so epochs would give big datasets more training. Fixed steps give every experiment the same training effort. |
| Count only **empty** cells for cell accuracy | Given cells can be copied from the input, which would make accuracy look better than it is. |
| A puzzle counts as solved only if **all 81** cells are right | That is what solving a sudoku means. |
| **Build the reasoning model before the data study** (plan changed) | The baseline solves 0% even with 98,000 puzzles, so every data experiment would score 0%. Data is not the main bottleneck; the model design is. A one-pass model with 4 layers cannot follow long chains of deductions. |
| Choose the reasoning model design on **one dataset** (10,000 puzzles), then freeze it | Keeps the number of experiments small: designs and datasets are tested in separate stages instead of every combination. |

### Limitation noted
The model design will be chosen with 10,000 puzzles. A different design might be slightly better with very small data (e.g. 100 puzzles). This will be mentioned in the report.

### Next
- Stage 3: a looping model. One small block (2 layers) reused 8 times (16 rounds of thinking), with the puzzle added again before every loop. Compare with the baseline on the same 10,000 puzzles and 10,000 steps.
- Then: an answer after every loop (learn from each step), then a learned stop decision (adaptive reasoning).

---

## 2026-10-26 — Data pipeline and augmentation

### Done
**Notebook 01 (data):**
- Loaded the Kaggle dataset: 3,000,000 puzzles with columns `id, puzzle, solution, clues, difficulty`. Empty cells are written as `.`.
- Took a random sample of 111,000 puzzles (seed 42).
- Converted each puzzle from text to 81 numbers (0 = empty).
- Checked all 111,000 solutions: every one is a valid sudoku, and every one keeps the puzzle's given numbers.
- Grouped puzzles by difficulty rating.
- Split into test, validation and pool, checked there are no duplicates between them, and saved them.

**Notebook 02 (augmentation):**
- Four changes that keep a puzzle valid with the same logic: swap digits, swap rows inside a band, swap whole bands, rotate.
- Checked 5,000 augmented copies: all valid, all matching their puzzle, all with the same number of clues. 10 copies of one puzzle were all different.
- Built and saved 7 training sets.

### Results

**Difficulty in the sample (111,000 puzzles):**

| Group | Rating | Count | Share |
|---|---|---|---|
| Easy | 0 | 48,004 | 43% |
| Medium | above 0, up to 2 | 32,955 | 30% |
| Hard | above 2 | 30,041 | 27% |

Rating summary: mean 1.23, median 1.2, 75th percentile 2.1, max 8.2.

**Data splits:**

| Split | Size |
|---|---|
| Test | 3,000 (1,000 per group) |
| Validation | 10,000 |
| Pool | 98,000 |

**Training sets built (name = originals × versions per puzzle):**

| File | Originals | Versions each | Total |
|---|---|---|---|
| train_n100_k1 | 100 | 1 | 100 |
| train_n100_k100 | 100 | 100 | 10,000 |
| train_n1000_k1 | 1,000 | 1 | 1,000 |
| train_n1000_k100 | 1,000 | 100 | 100,000 |
| train_n10_k1000 | 10 | 1,000 | 10,000 |
| train_n1000_k10 | 1,000 | 10 | 10,000 |
| train_n10000_k1 | 10,000 | 1 | 10,000 |

### Decisions
| Decision | Why |
|---|---|
| **Sudoku** as the domain | Answers checkable automatically, difficulty measurable, large datasets available, rich augmentation possible, and the model learns everything from our data. |
| Read the CSV **as text** | Otherwise pandas turns `"0043..."` into a number and loses the leading zeros (empty cells). |
| **Random** sample of 111,000, not the first rows | The file might be sorted (e.g. easy puzzles first). A random sample is a fair mix. |
| **Fixed seed (42)** everywhere | Every result can be repeated exactly. This also allowed the notebooks to be re-run in Colab with identical output. |
| Three **difficulty groups** | 43% of puzzles are very easy, which could hide weak reasoning in an overall score. Results are reported per group. Groups also make selection and curriculum experiments possible. |
| **Balanced test set** (1,000 per group) | With a random 1,000-puzzle test set, only ~270 would be hard, making the hard score unreliable. Balanced groups give equally reliable scores and stop easy puzzles from inflating the result. |
| Never augment the **test** set | The test must stay real, original puzzles so every experiment is scored the same honest way. |
| Decide with **validation**, use **test** only at the end | Checking the test set while making choices slowly "tunes to the test" and makes final results look better than they are. |
| 98,000 in the pool | The top of the data-size range, not a guess of what is needed. If results still improve at 98,000, more puzzles can be taken from the 3M file. |
| Augmentation **pre-built and saved**, not made during training | Every experiment uses exactly the same data, so comparisons are fair. |
| `k` = versions per puzzle (1 original + k−1 copies) | `k = 1` means no augmentation. A clear, simple definition of "how much augmentation." |
| Originals = **first N** of the pool | Smaller sets are inside bigger ones, so differences between sizes come from the amount of data, not from a luckier set of puzzles. |
| Build only **7 sets** now | Which sizes matter is not known yet. More sets will be built around the interesting sizes once the first results are in. |
| No column swaps in augmentation | Keeps the code simple. Rotation already mixes rows and columns, and one puzzle still gives about 1.9 billion different versions. |

### Practical notes
- Kaggle GPUs need phone verification, which did not work. Training moved to Google Colab (free T4 GPU). Data files are stored in Google Drive.
- Files downloaded from Kaggle came with a `.zip` ending. A `.npz` file is a zip of `.npy` arrays, so renaming `.zip` to `.npz` fixes it.
