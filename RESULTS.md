# Results Comparison

This document compares the two notebook variants in this repository:

- `phishing_granite_mlp.ipynb`
- `phishing_granite_mlp_structured_features.ipynb`

Both notebooks use the same Granite embedding backbone and the same general MLP classifier. The difference is the input representation.

## Summary

The structured notebook performs better on the held-out test set.

| Notebook | Input style | Input dim | Best epoch | Accuracy | Macro F1 | ROC AUC |
| --- | --- | ---: | ---: | ---: | ---: | ---: |
| `phishing_granite_mlp.ipynb` | `text_combined` only | 384 | 5 | 0.9757215620 | 0.9756825218 | 0.9968163884 |
| `phishing_granite_mlp_structured_features.ipynb` | Granite + numeric features | 423 | 17 | 0.9872773537 | 0.9872586268 | 0.9989886283 |

## Metric Delta

Structured input vs baseline:

- Accuracy: `+0.0115557917`
- Macro F1: `+0.0115761050`
- ROC AUC: `+0.0021722399`

## What Changed Between the Two Notebooks

### Baseline notebook

- Uses the unified `dataset/phishing_email.csv`
- Embeds only `text_combined`
- Sends a 384-dimensional Granite vector to the MLP
- Saves artifacts in `artifacts/`

### Structured notebook

- Uses the original CSV source files
- Keeps `sender`, `subject`, `body`, `receiver`, `date`, and `urls`
- Builds a 384-dimensional Granite embedding from `Subject + Body`
- Adds 39 numeric features from sender, subject, body, and URL-related signals
- Concatenates everything into a 423-dimensional vector
- Saves artifacts in `artifacts_structured/`

## Interpretation

The structured notebook is the stronger version for this dataset because it gives the MLP more than just language semantics.

The extra gain comes from signals that are often important in phishing:

- suspicious sender patterns
- URL presence
- urgency keywords
- capitalization and punctuation patterns
- text length and token-count behavior

That makes sense for this task: phishing is not only about what the email says, but also about how the sender and the structure of the message look.

## Training Notes

- Baseline model size: `0.45 MB`
- Structured model size: `0.49 MB`
- Baseline early-stop epoch: `5`
- Structured early-stop epoch: `17`

The structured model trained for longer because the richer input gave it more room to improve before validation stopped increasing.

## Caveat

The comparison is fair at the task level, but not identical at the raw-input level:

- baseline uses the already unified CSV
- structured notebook rebuilds the dataset from the source CSVs

That said, both solve the same binary classification problem on the same underlying corpus, and the structured version clearly improves the held-out metrics.
