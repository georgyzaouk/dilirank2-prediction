# Structure-Based Prediction of Drug-Induced Liver Injury Risk

## Abstract

Drug-induced liver injury (DILI) is an important safety concern in both clinical medicine and pharmaceutical development. Because DILI can be rare, mechanistically diverse, and difficult to reproduce in preclinical models, identifying hepatotoxic liability before widespread human exposure remains challenging. These limitations have motivated the development of *in silico* approaches that attempt to infer DILI risk directly from molecular structure.

This project investigates how well the ordered DILI concern categories defined by **DILIrank 2.0** can be predicted from two-dimensional molecular structure. Rather than reducing the problem to the binary hepatotoxic/non-hepatotoxic formulation commonly used in previous work, the primary benchmark preserves three verified levels of concern:

**No-DILI Concern < Less-DILI Concern < Most-DILI Concern**

Two molecular representations, RDKit descriptors and Morgan/ECFP4 fingerprints, are evaluated across several machine-learning algorithms under both conventional nominal classification and explicit ordinal classification. To assess chemical generalization rather than only interpolation between related molecules, models are evaluated using both parent-grouped random cross-validation and the more restrictive Bemis–Murcko scaffold-grouped cross-validation.

The results show that molecular structure contains a reproducible but moderate signal for DILI concern. Morgan fingerprints generally outperform the descriptor representation, while Random Forest provides the strongest overall configuration. Explicit ordinal modeling does not consistently improve exact classification accuracy, but it can reduce severe errors between the two extremes of the DILI scale. Overall, the results suggest that structure alone is informative but insufficient to fully explain clinical DILI risk, motivating the inclusion of exposure, metabolism, biological activity, and patient-level information in future models.

---

## 1. Introduction

Drug-induced liver injury encompasses liver damage associated with exposure to therapeutic compounds and remains a major problem in drug safety assessment. Its manifestations range from relatively mild changes in liver enzymes to severe liver failure, and its occurrence can depend on multiple interacting factors including chemical structure, dose, metabolism, pharmacokinetics, biological targets, and patient susceptibility.

Prediction is particularly difficult for **idiosyncratic DILI**, which may occur only in a small subset of exposed patients and can escape detection during conventional preclinical testing. Animal and *in vitro* systems therefore provide incomplete representations of human DILI risk, creating an important role for complementary computational approaches [2–6].

A central difficulty for machine learning is the definition of reliable labels. DILI does not have a simple experimental ground truth, and classifications assembled from different data sources can vary substantially. The FDA-developed **DILIrank** resource addressed this problem by combining information from FDA-approved drug labeling with evidence of clinical causality [2].

The recently published **DILIrank 2.0** expands and updates this resource to **1,336 drugs**, including:

| DILIrank 2.0 category | Drugs |
|---|---:|
| No-DILI Concern | 414 |
| Ambiguous-DILI Concern | 354 |
| Less-DILI Concern | 351 |
| Most-DILI Concern | 217 |
| **Total** | **1,336** |

DILIrank 2.0 therefore provides an updated benchmark for investigating the relationship between molecular structure and clinically observed hepatotoxicity [1].

Previous computational studies have demonstrated that molecular fingerprints, descriptors, machine-learning models, deep neural networks, and ensemble approaches can recover meaningful DILI-related signals [3–6]. However, much of this work converts DILI into a **binary classification problem**.

The present project instead focuses on the more difficult question:

> **Can molecular structure distinguish between multiple ordered levels of clinically verified DILI concern?**

This formulation preserves more of the information encoded by DILIrank and allows us to investigate not only whether a prediction is wrong, but also **how severe the error is**.

---

## 2. Research Questions

The analysis is designed around five main questions:

1. **Can DILI concern be predicted from 2D molecular structure at all?**
2. **Are Morgan fingerprints more informative than conventional molecular descriptors?**
3. **How much predictive performance is lost when models are evaluated on previously unseen molecular scaffolds?**
4. **Does explicitly modeling the order `No < Less < Most` improve prediction?**
5. **What do persistent prediction errors reveal about the limitations of structure-only DILI modeling?**

---

## 3. Data

### 3.1 DILIrank 2.0

The starting dataset contains the **1,336 compounds** reported in DILIrank 2.0.

Molecular structures were retrieved through PubChem and validated with RDKit. Exact structure retrieval produced:

- **1,215 structure-resolved compounds**
- 113 compounds without an exact structure match
- 8 compounds with multiple possible PubChem candidates
- **1,215 / 1,215 retrieved structures successfully parsed by RDKit**

Molecular structures were subsequently standardized before feature generation.

### 3.2 Primary prediction endpoint

The main benchmark uses the three verified DILI concern categories:

```text
No < Less < Most
```

Compounds labelled **Ambiguous-DILI Concern** are excluded from the primary supervised benchmark because their DILI concern is not supported by the same level of verified causality as the three primary classes.

After requiring a valid molecular structure and excluding the ambiguous category, the final modeling population contains **886 compounds**:

| Class | Compounds |
|---|---:|
| No | 350 |
| Less | 330 |
| Most | 206 |
| **Total** | **886** |

The original DILIrank `SeverityClass` variable from 0–8 is retained for future analysis. It is not used as the primary endpoint because several individual severity levels are too sparse for a reliable grouped nine-class cross-validation benchmark.

---

## 4. Molecular Representations

Two complementary representations of molecular structure are compared.

### 4.1 RDKit molecular descriptors

RDKit is used to calculate a collection of 2D physicochemical and topological descriptors describing properties such as:

- molecular size;
- lipophilicity;
- polarity;
- hydrogen-bonding capacity;
- ring systems;
- topology;
- functional groups.

The raw representation contains **217 descriptors**.

Missing-value handling, scaling, constant-feature removal, correlation filtering, and other preprocessing operations are performed **inside the training folds** to avoid information leakage.

### 4.2 Morgan fingerprints

The second representation is a circular Morgan fingerprint:

```text
Radius: 2
Diameter: 4 (ECFP4-style)
Bits: 2048
Type: binary fingerprint
```

Unlike broad molecular descriptors, Morgan fingerprints encode the presence of local molecular environments and substructures.

This representation is particularly relevant to DILI prediction because previous studies have repeatedly demonstrated the utility of structural fingerprints for hepatotoxicity modeling [3–6].

---

## 5. Experimental Design

### 5.1 Nominal classification

The first formulation treats the three DILI classes as independent categories.

Four model families are evaluated:

- Logistic Regression
- Support Vector Machine
- Random Forest
- Histogram Gradient Boosting

### 5.2 Ordinal classification

The second formulation explicitly incorporates the known order:

```text
No = 0
Less = 1
Most = 2
```

Ordinal prediction is implemented through a cumulative reduction in which models learn two binary thresholds:

```text
y > 0
y > 1
```

The predicted class corresponds to the number of thresholds crossed.

This allows the same underlying learner families to be compared under matched nominal and ordinal formulations.

---

## 6. Leakage-Aware Validation

A major objective of the project is to distinguish ordinary predictive performance from genuine **chemical generalization**.

Two fixed five-fold outer cross-validation schemes are therefore used.

### Parent-grouped random CV

The first regime approximates conventional stratified random cross-validation while ensuring that compounds representing the same parent structure cannot appear in both training and test data.

This reduces obvious chemical leakage while providing a benchmark comparable to standard machine-learning evaluation.

### Scaffold-grouped CV

The second regime groups compounds according to their **Bemis–Murcko scaffold**.

All compounds containing the same scaffold are kept within the same fold, forcing the model to make predictions for chemical frameworks that were not present during training.

This provides a more demanding estimate of generalization to structurally novel compounds.

### Nested model selection

Hyperparameter tuning is performed only on the training portion of each outer fold using grouped inner cross-validation.

The complete modeling process therefore follows the structure:

```text
Outer fold
│
├── Training compounds
│   └── grouped inner CV
│       └── preprocessing + hyperparameter selection
│
└── Held-out test compounds
    └── final evaluation
```

This ensures that preprocessing and model selection do not use information from the outer test folds.

---

## 7. Evaluation Metrics

Because the three classes are not equally represented, **balanced accuracy** and **macro-F1** are used as the primary classification metrics.

The evaluation additionally reports:

- Accuracy
- Macro precision
- Macro recall
- Weighted F1
- Class-specific precision, recall, and F1
- Confusion matrices

Because the endpoint is ordered, two additional metrics are used:

- **Mean Absolute Error (MAE)** on the encoded classes `0, 1, 2`
- **Quadratic Weighted Kappa (QWK)**

Errors are also separated into:

```text
Correct:
No → No
Less → Less
Most → Most

Adjacent error:
No ↔ Less
Less ↔ Most

Extreme error:
No ↔ Most
```

This distinction is important because an ordinal model may be scientifically useful even when it does not increase exact classification accuracy, if it reduces clinically more severe two-level errors.

ROC-AUC is not reported in the current benchmark because the saved out-of-fold predictions contain hard class predictions rather than continuous class probabilities or decision scores. Computing ROC-AUC from hard labels would not be valid.

---

## 8. Results

### 8.1 Best-performing models

The strongest configuration under both evaluation regimes is a **nominal Random Forest trained on Morgan/ECFP4 fingerprints**.

| Evaluation | Formulation | Representation | Model | Balanced Accuracy | Macro-F1 | Ordinal MAE | QWK |
|---|---|---|---|---:|---:|---:|---:|
| Parent-grouped random CV | Nominal | Morgan ECFP4 | Random Forest | **0.537** | **0.529** | **0.554** | **0.421** |
| Scaffold-grouped CV | Nominal | Morgan ECFP4 | Random Forest | **0.512** | **0.505** | **0.609** | **0.320** |

For comparison, chance-level balanced accuracy for a three-class problem is approximately:

```text
1 / 3 = 0.333
```

The models therefore recover a meaningful structure–DILI relationship, but the predictive signal remains moderate.

### 8.2 Morgan fingerprints outperform broad molecular descriptors

Morgan fingerprints achieve higher balanced accuracy than the RDKit descriptor representation in **13 of 16 matched comparisons**, with an average balanced-accuracy advantage of approximately **0.020**.

This suggests that local molecular environments and substructures contain more useful information for this endpoint than broad physicochemical properties alone.

The finding is consistent with previous fingerprint-based DILI studies and with the exploratory analysis of this dataset, where the three classes occupy strongly overlapping chemical space.

### 8.3 Generalization to unseen scaffolds is more difficult

Moving from parent-grouped random evaluation to scaffold-grouped evaluation generally decreases performance.

For the best model:

```text
Random CV BA   = 0.537
Scaffold CV BA = 0.512
```

The reduction is relatively modest, indicating that some learned DILI-related structural information transfers beyond exact scaffold families.

However, this should not be interpreted as strong out-of-domain prediction: performance under both regimes remains moderate.

### 8.4 The intermediate class is the most difficult

For the best Random Forest model, **Less-DILI Concern** has the lowest recall under both evaluation regimes.

This is scientifically plausible. `Less` occupies an intermediate position between compounds with no verified concern and compounds associated with the strongest DILI concern, producing substantial structural overlap with both neighboring classes.

The task is therefore considerably more difficult than simply separating clearly hepatotoxic from clearly non-hepatotoxic compounds.

### 8.5 Ordinal modeling changes the nature of errors

Explicit ordinal learning does **not consistently improve exact three-class balanced accuracy**. Across matched model comparisons, ordinal models improve balanced accuracy in 7 of 16 cases.

Their benefit becomes clearer when the magnitude of the mistakes is considered.

Ordinal models reduce extreme `No ↔ Most` errors in **11 of 16 matched configurations**, with the extreme-error rate decreasing by approximately **1.4 percentage points on average**.

This suggests that ordinal learning is useful not necessarily because it predicts the exact class more frequently, but because it can produce **more reasonable mistakes**.

### 8.6 Persistent failures indicate a structure-only information ceiling

Several compounds remain incorrectly predicted across essentially every combination of:

- algorithm;
- representation;
- nominal vs ordinal formulation;
- random vs scaffold evaluation.

Examples among the most consistently difficult compounds include:

- Acarbose
- Busulfan
- Ethambutol hydrochloride
- Methotrexate sodium
- Obeticholic acid

The persistence of these failures suggests that the performance ceiling is not explained solely by the choice of machine-learning algorithm.

Clinical DILI liability depends on factors that are only partially encoded by 2D molecular structure, including:

- administered dose and systemic exposure;
- drug metabolism and reactive metabolites;
- transporter activity;
- pharmacokinetics;
- molecular targets and off-target activity;
- mitochondrial and cellular stress responses;
- patient susceptibility and genetic factors.

This agrees with previous DILI modeling literature, where chemical structure provides useful predictive information but does not fully capture this multifactorial endpoint [3–6].

---

## 9. Discussion

Three main conclusions emerge from the benchmark.

First, **DILI concern is partially predictable from molecular structure**. Performance consistently exceeds the three-class chance baseline, demonstrating that structural information is related to clinical DILI liability.

Second, **the representation of molecular structure matters**. Morgan fingerprints consistently outperform the broader RDKit descriptor space, supporting the importance of local chemical environments and substructures for hepatotoxicity prediction.

Third, the results reveal an important limitation. Changing classifiers does not fundamentally solve the prediction problem. Logistic models, SVMs, Random Forests, and gradient-boosted trees all reach a similar moderate performance range.

The resulting performance plateau suggests an **information ceiling for structure-only prediction** of the three-level DILIrank endpoint.

This distinction is important when comparing the present work with earlier studies. Previous DILI models often report stronger performance after combining `Less` and `Most` into a single DILI-positive category. The present benchmark deliberately retains the more difficult distinction between:

```text
No
Less
Most
```

Performance numbers should therefore not be compared directly with binary DILI benchmarks without accounting for the different prediction task and validation strategy.

The objective of this project is consequently not to claim that a structure-only classifier can replace experimental or clinical safety assessment. Instead, it establishes a reproducible baseline for determining **how much of DILI concern can be explained by molecular structure alone**.

---

## 10. Conclusion

This study evaluates machine-learning models for predicting the ordered DILIrank 2.0 categories `No < Less < Most` directly from molecular structure.

The main findings are:

1. **2D molecular structure contains meaningful predictive information about DILI concern, but the signal is moderate rather than sufficient for highly accurate three-class prediction.**
2. **Morgan/ECFP4 fingerprints are generally more effective than the tested RDKit descriptor representation.**
3. **Random Forest provides the strongest overall benchmark in both random and scaffold-grouped evaluation.**
4. **Performance decreases when predicting unseen scaffolds, confirming that chemical generalization is more difficult than conventional random-fold prediction.**
5. **Ordinal modeling does not systematically improve exact classification, but it can reduce severe `No ↔ Most` mistakes.**
6. **Persistent errors across multiple model families suggest that the main limitation is increasingly the information available to the model rather than the classifier itself.**

The natural next step is therefore to move beyond the question:

> *Which machine-learning algorithm performs best?*

toward the more biologically relevant question:

> **Which additional sources of information are required to explain the DILI liability that molecular structure alone cannot capture?**

Potential extensions include exposure and dose information, predicted or experimental protein targets, metabolic features, bioactivity measurements, transcriptomic signatures, and other mechanistically relevant data.

---

## 11. Project Workflow

The project is organized as a sequential six-notebook pipeline:

```text
01_data_preparation.ipynb
        │
        ▼
02_exploratory_analysis.ipynb
        │
        ▼
03_features_and_splits.ipynb
        │
        ├───────────────────────┐
        ▼                       ▼
04_nominal_models.ipynb   05_ordinal_models.ipynb
        │                       │
        └───────────┬───────────┘
                    ▼
06_evaluation_and_error_analysis.ipynb
```

### `01_data_preparation.ipynb`
Cleans the DILIrank 2.0 dataset, retrieves molecular structures from PubChem, validates them with RDKit, standardizes structures, and audits duplicate and parent-equivalent compounds.

### `02_exploratory_analysis.ipynb`
Explores DILI-class distributions, severity classes, molecular properties, scaffold diversity, structural similarity, chemical space, and the position of Ambiguous compounds.

### `03_features_and_splits.ipynb`
Creates the fixed RDKit descriptor and Morgan fingerprint representations and defines the parent-grouped random and scaffold-grouped outer folds used throughout the benchmark.

### `04_nominal_models.ipynb`
Benchmarks Logistic Regression, SVM, Random Forest, and Histogram Gradient Boosting while treating `No`, `Less`, and `Most` as nominal classes.

### `05_ordinal_models.ipynb`
Repeats the benchmark using cumulative ordinal classification while preserving the order `No < Less < Most`.

### `06_evaluation_and_error_analysis.ipynb`
Performs the final comparison of model performance, representations, validation regimes, ordinal error severity, difficult compounds, and difficult scaffolds.

---

## 12. References

1. **Olubamiwa, A. O., Qu, Y., Connor, S., Tong, W., Li, D., & Chen, M.**  
   *DILIrank 2.0: An updated and expanded database for drug-induced liver injury risk based on FDA labeling and a literature review.*  
   Drug Discovery Today, 2025, 30(11), 104485.  
   https://doi.org/10.1016/j.drudis.2025.104485

2. **Chen, M., Suzuki, A., Thakkar, S., Yu, K., Hu, C., & Tong, W.**  
   *DILIrank: the largest reference drug list ranked by the risk for developing drug-induced liver injury in humans.*  
   Drug Discovery Today, 2016, 21(4), 648–653.  
   https://doi.org/10.1016/j.drudis.2016.02.015

3. **Ancuceanu, R., Hovanet, M. V., Anghel, A. I., et al.**  
   *Computational Models Using Multiple Machine Learning Algorithms for Predicting Drug Hepatotoxicity with the DILIrank Dataset.*  
   International Journal of Molecular Sciences, 2020, 21, 2114.  
   https://doi.org/10.3390/ijms21062114

4. **Liu, A., Walter, M., Wright, P., et al.**  
   *Prediction and mechanistic analysis of drug-induced liver injury (DILI) based on chemical structure.*  
   Biology Direct, 2021, 16, 6.  
   https://doi.org/10.1186/s13062-020-00285-0

5. **Yang, Q., Zhang, S., & Li, Y.**  
   *Deep Learning Algorithm Based on Molecular Fingerprint for Prediction of Drug-Induced Liver Injury.*  
   Toxicology, 2024, 502, 153736.  
   https://doi.org/10.1016/j.tox.2024.153736

6. **Khan, M. Z. I., Ren, J.-N., Cao, C., et al.**  
   *Comprehensive hepatotoxicity prediction: ensemble model integrating machine learning and deep learning.*  
   Frontiers in Pharmacology, 2024, 15, 1441587.  
   https://doi.org/10.3389/fphar.2024.1441587

---

> **Project note:** This project is a reworked and expanded version of my capstone project completed during my bachelor's degree. The current version revisits the original problem using the updated DILIrank 2.0 dataset, a redesigned reproducible pipeline, leakage-aware validation, and a more systematic comparison of nominal and ordinal machine-learning approaches.
