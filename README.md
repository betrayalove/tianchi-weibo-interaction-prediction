# Weibo Interaction Prediction Baseline

A reproducible baseline for the [Alibaba Tianchi Weibo interaction prediction competition](https://tianchi.aliyun.com/competition/entrance/231574/team).
Given a user's posting history, the task is to predict how many forwards, comments and likes each new Weibo post will receive within one week.
The solution builds user-history and text features, validates on a held-out time period, and exports a submission in the Tianchi format.

## What the task is

For each post, predict three targets:

- `forward_count`
- `comment_count`
- `like_count`

The local files in this project do not match the dates written in `instruction.md`.

- `instruction.md` says training data is from `2014-07-01` to `2014-12-31` and prediction data is from `2015-01-01` to `2015-01-31`
- the actual local training file spans `2015-02-01` to `2015-07-31`
- the actual local prediction file spans `2015-08-01` to `2015-08-31`

The baseline uses the real local files as the source of truth.

## Data layout in this repository

The dataset is not committed to this repository. Download it from the competition page and put the files into `Weibo Data/`.

GitHub does not accept files larger than `100 MB`, so the loader also supports the training file split into parts:

- `Weibo Data/weibo_train_data.partaa`
- `Weibo Data/weibo_train_data.partab`
- `Weibo Data/weibo_train_data.partac`
- `Weibo Data/weibo_train_data.partad`
- `Weibo Data/weibo_train_data.partae`

The loader reads either:

- the original monolithic `Weibo Data/weibo_train_data.txt` if it exists locally
- or the split `part*` files automatically when the large file is absent

## What is implemented

- time-based validation split
- user history mean baseline
- user history features
- simple text and metadata features
- character n-gram hashing for Chinese text
- optional three separate `SGDRegressor` models blended with history baseline
- submission export in Tianchi format
- two local score variants for the ambiguous `sign()` definition

## Environment

Create the environment:

```bash
python3 -m venv .venv --system-site-packages
.venv/bin/pip install -r requirements.txt
```

## Run

Full run:

```bash
.venv/bin/python train_baseline.py
```

Run with the optional text model:

```bash
.venv/bin/python train_baseline.py --enable-text-model
```

Faster smoke run:

```bash
.venv/bin/python train_baseline.py --max-train-rows 200000 --max-predict-rows 50000
```

Outputs written by `train_baseline.py`:

- `artifacts/result.txt` — submission: `uid`, `mid` and a comma-separated `forward,comment,like` triplet, tab-separated
- `artifacts/validation_report.json` — row counts, selected model and validation scores

The committed `artifacts/submission_baseline.txt` is an older submission from the first version of the baseline (five tab-separated columns: `uid`, `mid`, `forward`, `comment`, `like`).

## Notes on the metric

The archived formula image was incomplete, so the local evaluator uses the most likely interpretation from public writeups:

- `deviation = abs(pred - real) / (real + 5)`
- `precision_i = 1 - 0.5 * deviation_f - 0.25 * deviation_c - 0.25 * deviation_l`

Two variants are reported:

- `indicator`: counts a post only when `precision_i > 0.8`
- `math_sign`: uses the mathematical sign `-1/0/1`

## Baseline choice

The models are intentionally conservative:

- `user_history_mean` predicts each user's average historical interactions; it is fast, stable, and aligns with public writeups of the task
- `recent30_q58_result11` predicts the 0.58 quantile of each user's interactions over the last 30 days, scaled per target (forward ×1.1, comment ×0.8, like ×1.0); it is selected only if it beats `user_history_mean` on validation
- the text model is optional because it is slower and may or may not help on a given split; it is blended in only if it improves the validation score

## Results

Time-based validation from `artifacts/validation_report.json` (posts from `2015-07-01` onward are held out):

| Split | Rows |
| --- | ---: |
| Train (before 2015-07-01) | 1,040,874 |
| Validation (from 2015-07-01) | 184,214 |
| Predict | 177,923 |

| Model | Indicator precision | Math-sign precision |
| --- | ---: | ---: |
| `user_history_mean` (baseline) | 0.315 | -0.368 |
| `recent30_q58_result11` (selected) | **0.353** | **-0.291** |

The selected model improves the indicator precision by about 12% relative to the baseline.

## What I learned

- Validation has to follow time: holding out the last month of posts gives a realistic estimate for predicting future posts.
- When the metric definition is ambiguous, it is safer to implement both readings (`indicator` and `math_sign`) and compare models on both.
- GitHub rejects files larger than 100 MB, so large training data needs to be split into parts and the loader has to read them transparently.
- A simple, honest baseline is a strong reference: a heavier model (the text model) is used only if it actually beats the baseline on validation.
