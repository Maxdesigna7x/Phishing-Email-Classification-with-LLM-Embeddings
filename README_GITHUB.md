# Phishing Email Classification with Granite Embeddings

![Lit mode](lit_mode.jpg)

This repository explores phishing detection with a hybrid approach:

1. Extract semantic embeddings with `ibm-granite/granite-embedding-97m-multilingual-r2`.
2. Add structured numeric features from the email metadata.
3. Train a lightweight PyTorch `MLP` on the combined vector.

There are two notebook variants:

- `phishing_granite_mlp.ipynb`
- `phishing_granite_mlp_structured_features.ipynb`

The baseline notebook uses the unified `text_combined` field. The structured notebook preserves original email fields such as `sender`, `subject`, `body`, and `urls`, and it performs better on the held-out test set.

## Results

| Variant | Accuracy | Macro F1 | ROC AUC |
| --- | ---: | ---: | ---: |
| Baseline embedding-only | 0.9757 | 0.9757 | 0.9968 |
| Structured features + embedding | 0.9873 | 0.9873 | 0.9990 |

See [`RESULTS.md`](./RESULTS.md) for the full comparison.

## Project Layout

- `dataset/` source and unified phishing CSVs
- `artifacts/` baseline model outputs
- `artifacts_structured/` structured model outputs
- `DOC.md` step-by-step explanation
- `RESULTS.md` experimental comparison
- `README.md` internal project overview

## Key Idea

Phishing is not only about language semantics. It also leaves structural clues in the sender, subject line, body shape, and URL patterns. The structured notebook captures those signals and improves the model.
