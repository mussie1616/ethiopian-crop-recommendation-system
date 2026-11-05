c# Crop Recommendation Based on Soil and Climate Conditions

## Verification status (2026-11-04)

The saved-model verification script was rerun against the included 600-row dataset and saved model. It reproduced the 420/90/90 split and the recorded held-out metrics (83.33% accuracy, 0.8305 Macro-F1, and 96.67% top-three accuracy). See `evidence/verification_log_current_2026-11-04.txt`.

The five tests in `test_input_checks.py` passed. The full pytest run could not complete in the verification environment because Streamlit was not installed, so the Streamlit app tests are not claimed as freshly passed. See `evidence/test_execution_status_2026-11-04.txt`. The model was serialized with scikit-learn 1.9.1; the verification environment had 1.8.0 and emitted a compatibility warning. Rerun in the intended scikit-learn 1.9.1 environment before final release.

## Project overview
A multiclass machine-learning prototype that ranks three crop labels from seven input features:
`N`, `P`, `K`, `temperature`, `humidity`, `ph`, and `rainfall`.

## Dataset

**Short version:** the 600-row file is treated as **synthetic** data of **undocumented origin**. It is **not** verified
Ethiopian field data, and the results must not be used to make real farm recommendations.

The supplied dataset has 600 rows, 7 numeric input features, and a `label` target with 10 crop classes (60 rows per
class). The included CSV is `data/Crop_recommendation_Ethio.csv`.

**Where it came from.** The file was supplied to the group as a CSV with no accompanying documentation: no author,
collection site, dates, measurement units, or labelling protocol. Who produced it, and with what code or method, could
not be verified. The file name and crop names suggest an Ethiopian focus, but nothing shows that any row is an
Ethiopian field observation.

**Were the records synthetically generated?** Almost certainly yes, but this is an inference from the data, not a
documented fact. The notebook (Task 3) computes the evidence and saves it to `outputs/data_provenance_checks.csv`:
- exactly 60 rows for every crop;
- every one of the 70 crop-by-feature distributions is consistent with a normal distribution (field data, especially
  rainfall, is usually skewed);
- features are uncorrelated within each crop (mean absolute correlation 0.100, against about 0.104 expected from
  independent random draws);
- rainfall has a near-constant spread across crops (SD 131 to 153) even though crop means range from 441 to 1,415
  (coefficient of variation 10% to 31%), whereas real variability usually grows with the mean;
- humidity is capped at exactly 100 and rainfall floored at exactly 100 in one row each, and all values carry about 15
  decimal places, which suggests generated values clipped at a bound rather than rounded instrument readings.

**How were the crop labels assigned?** The structure indicates each row was generated *for* a crop, around a
crop-specific profile of nutrient, climate and pH values. The label is therefore the category the row was generated for.
It is **not** a recorded outcome such as the crop a farmer grew, or the crop that gave the best yield on that plot.
The profiles themselves are undocumented.

**What "no synthetic data was added" means.** Our preparation step only validates the file (numeric conversion, dropping
missing or non-finite rows, trimming and lower-casing labels). We did not generate, augment, oversample or edit rows. This
is a statement about our pipeline only; it says nothing about how the supplied rows were produced.

Because the model learns to separate ten generated, overlapping profiles, the reported scores are
**software-validation results, not agricultural validation**. Units (for example whether temperature is in °C and
rainfall in mm) are also undocumented.

## Files
- `README.md` — basic instructions for installing, running, and using the project.
- `Group_Contributions.md` — List of group 2 menbers Role area and Specific contribution.
- `Ethiopian_Crop_Recommendation_Implementation.ipynb` — notebook organized into the eight project tasks; runs locally or in Colab, and is saved with its outputs.
- `data/Crop_recommendation_Ethio.csv` — input dataset.
- `crop_recommendation_model.joblib` — fitted model (Random Forest), selected by validation Macro-F1 and fitted on train + validation rows.
- `model_metadata.json` — feature list, labels, model name, provenance note, test metrics, and the fitted-range summary used by the app's input checks.
- `input_checks.py` — warns when an input is outside, or unlike, what the model was fitted on (used by the app and notebook).
- `outputs/` — every table and chart the report cites: validation comparison, test metrics, classification report, per-class metrics, confusion matrix, top-k accuracy, provenance checks, EDA charts, feature importance, and `model_training_features.csv` (the rows the model was fitted on).
- `requirements.txt` — Python package dependencies.
- `Project Implementation Report.docx` — report of method, measured results, limitations, and conclusion. It embeds the EDA charts, model-comparison chart, confusion matrix, class-level results, a test-accuracy vs top-3 explanation, and the app's input-validation screenshots.
- `build_report.py` — rebuilds the report from `outputs/`, `evidence/` and `model_metadata.json` (`python build_report.py`), so figures and numbers always match the saved model.
- `app.py` — Streamlit interface for entering values and viewing top-three results with reliability warnings.
- `test_model.py` — minimal inference example.
- `verify_saved_model.py` — checks that the saved model reproduces the reported metrics on the held-out rows.
- `test_app_validation.py` — automated tests of the app's input warnings (needs Streamlit).
- `test_input_checks.py` — Streamlit-free tests of the range, edge and combination checks.
- `evidence/` — screenshots of the running app and the log of the verification run.

## Run locally
Use Python 3.10+ in a virtual environment:
```bash
python -m venv .venv
# Windows: .venv\Scripts\activate
# macOS/Linux: source .venv/bin/activate
pip install -r requirements.txt
```
Then, from the project folder:
```bash
pip install notebook                  # only if Jupyter is not already installed
jupyter notebook Ethiopian_Crop_Recommendation_Implementation.ipynb   # run all cells; regenerates outputs/, the model and metadata
python verify_saved_model.py          # confirms the saved model reproduces the reported test metrics
python test_input_checks.py           # tests the input-check logic (no Streamlit needed)
python test_app_validation.py         # tests the app's input warnings
python build_report.py                # rebuilds Project Implementation Report.docx
streamlit run app.py                  # starts the web interface (http://localhost:8501)
```
The notebook reads `data/Crop_recommendation_Ethio.csv` automatically. The app expects
`crop_recommendation_model.joblib`, `model_metadata.json`, `input_checks.py` and `outputs/model_training_features.csv`
in the project folder.

## Run in Google Colab
1. Upload `Ethiopian_Crop_Recommendation_Implementation.ipynb` to Google Colab.
2. Run cells from top to bottom.
3. When prompted, upload `Crop_recommendation_Ethio.csv`.
4. Review the data checks, EDA, validation comparison, test metrics, confusion matrix, and top-k results.
5. The last task tests the recommendation function and downloads a ZIP of the model, metadata and evaluation outputs.

## Input validation in the app
A valid number is not necessarily a reliable input for this model. Besides the physical limits (pH 0 to 14, humidity
0 to 100%, no negative nutrients or rainfall), the app compares every entry with the rows the model was fitted on:
- **Outside the fitted range** (below the minimum or above the maximum seen in training): a red "Low reliability"
  banner and a per-feature warning that names the range.
- **Edge of the fitted range** (more extreme than 99% of fitted rows): a caution message.
- **Unfamiliar combination**: even if each value is individually in range, the combination is flagged when it is farther
  from every fitted row than the most unusual 1% of fitted rows are from their nearest neighbour.

The reason: the model gives a confident-looking score even for impossible inputs (temperature −5 with rainfall 20 still
returns barley at 63%). On the 90 held-out test rows the check flags 4 as outside range and none as an unfamiliar
combination, but it flags 79% of random combinations whose values all lie inside their individual ranges
(`outputs/input_check_validation.csv`). Units are undocumented, so enter values in the same units as the training data;
the app shows the fitted range beside every input.

## Run the model without the app
```python
import joblib
import pandas as pd

model = joblib.load("crop_recommendation_model.joblib")

features = [
    "N", "P", "K", "temperature",
    "humidity", "ph", "rainfall"
]

sample = pd.DataFrame(
    [[60, 30, 35, 22, 60, 6.4, 700]],
    columns=features
)

probabilities = model.predict_proba(sample)[0]
order = probabilities.argsort()[::-1][:3]

for rank, i in enumerate(order, start=1):
    print(
        f"{rank}. {model.classes_[i]} "
        f"— model score: {probabilities[i]:.2%}"
    )
```

## Results at a glance (held-out test set, 90 rows)
- **Test accuracy 0.822**: the single highest-scored crop is the true label for 74 of 90 rows.
- **Top-3 accuracy 0.967**: the true label is among the three highest-scored crops for 87 of 90 rows. This is an easier
  test (random guessing gives 10% for top-1 and 30% for top-3) and it does not say which of the three is right.
- Macro-F1 0.821. Five-fold cross-validated Macro-F1 0.808 ± 0.026.

## Interpretation and responsible use
- The model recommends labels learned from this dataset; it does not calculate yield, profitability, fertilizer prescriptions, or guaranteed crop suitability.
- Inputs should be measured in units consistent with the training dataset.
- Field validation with trusted Ethiopian soil, climate, and crop outcome data is needed before operational agricultural use.
- Do not enter values the app warns about, and do not treat high scores as probabilities calibrated for real-world outcomes; calibration has not been established.
