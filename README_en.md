# Melting Point Prediction for Organic Molecules (QSPR)

A learning research project: predict the melting point of an organic molecule from its structure (SMILES), and find out which features and which validation scheme give an honest estimate of model quality.

## Summary

- Data: the open Jean-Claude Bradley Open Melting Point Dataset, **19,824 unique molecules** after cleaning.
- Model: random forest on RDKit descriptors, compared across several feature sets: all 216 descriptors, a small physics-based set, a top-K set selected by model importance (inside the folds), and the same sets extended with 4 engineered features. Everything is compared against a median baseline. Validation: 5-fold cross-validation with a **scaffold-based split** (GroupKFold), so the test set contains new structures.
- Result: **RMSE 47 K and MAE 36 K**, about half the error of a median-only baseline (93 K and 75 K).
- A small set of 10-14 physically motivated features trails the full set (216 descriptors) by roughly 4-5 K RMSE.

## Goal

Build a QSPR (Quantitative Structure-Property Relationship) model for melting point and answer three questions:

1. How much better is the model than a trivial baseline when it is tested on **new** scaffolds rather than on molecules similar to the training set?
2. Is a small descriptor set chosen from physics and chemistry enough, or is the full set needed?
3. Do features engineered from the original descriptors help?

## Data

- Source: Jean-Claude Bradley Open Melting Point Dataset, figshare, DOI `10.6084/m9.figshare.1031637`. The raw set has 28,645 measurements, some flagged as unreliable ("do not use").
- The file is not stored in the repository: download it via the DOI and place it in `data/raw/`.
- Download date and file name: 23 November 2026, `BradleyMeltingPointDataset.xlsx`.

## Data cleaning

| Step | What was done | Rows after step |
|---|---|---|
| Raw dataset | loaded | 28,645 |
| "Do not use" filter | removed rows flagged as unreliable | 28,268 |
| SMILES canonicalization | RDKit `MolToSmiles` | 27,976 |
| Invalid SMILES | 292 rows failed to parse in RDKit; structures for 261 were recovered by name via PubChem, the rest were dropped | 28,237 |
| Temperature range | not applied | 28,237 |
| Duplicates | values for the same molecule merged by median; molecules whose sources disagree by more than 20 K were removed | **19,824** |

Decisions taken:

- Stereochemistry: not removed. The default RDKit canonical SMILES (with stereochemistry) was used, so stereoisomers are treated as separate molecules.
- Salts and mixtures: no fragment selection was applied; multi-component SMILES were left as they are.
- Structures recovered by name were not all checked manually; PubChem takes the first match, so some of them may be wrong.

## Features

- **All descriptors:** 216 RDKit descriptors (after removing columns with many missing values and constant columns). `inf` values were replaced, and remaining missing values are filled with the median inside the pipeline (from the training part only).
- **Physical set (10 features):** `MolWt`, `TPSA`, `NumHAcceptors`, `MolLogP`, `HeavyAtomCount`, `LabuteASA`, `NumRotatableBonds`, `FractionCSP3`, `NumAromaticRings`, `RingCount`. Chosen via Tm = ΔH/ΔS: lattice strength (hydrogen bonding, polarity, size) and melting entropy (flexibility, symmetry).
- **Engineered features (4):**
  - `HBD_per_atom` = H-bond donors / heavy atoms;
  - `TPSA_per_mass` = TPSA / molecular weight;
  - `rot_per_atom` = rotatable bonds / heavy atoms;
  - `force_over_freedom` = (donors + acceptors + TPSA/20) / (1 + rotatable bonds). The weights are arbitrary and serve as a hypothesis to test.

## Validation

- **GroupKFold, 5 folds, groups = Murcko scaffolds.** Molecules sharing a scaffold always land in the same fold, so the model is tested on scaffolds it has not seen. Acyclic molecules get their own group (their SMILES).
- Test fold sizes: 5329, 3624, 3624, 3624, 3623. The first fold is noticeably larger, probably because of one very large group (the aromatic scaffold group).
- All models and feature sets are compared **on the same folds**.
- Baseline: `DummyRegressor(strategy="median")`, i.e. the median of the training part.
- Metrics: MAE and RMSE in kelvin, mean over folds ± standard deviation across folds (a rough guide to spread, not a rigorous statistical error).

## Results

### Model vs baseline (5-fold by scaffold)

| Model | MAE, K | RMSE, K |
|---|---|---|
| median | 74.7 ± 6.8 | 93.3 ± 9.6 |
| random forest, all descriptors | 35.7 ± 1.9 | 47.1 ± 1.4 |

Improvement over the median: about 52% in MAE and 50% in RMSE.

### Effect of the splitting scheme (single 80/20 split, random forest)

| Scheme | MAE, K | RMSE, K | Median: MAE / RMSE |
|---|---|---|---|
| random | 29.1 | 39.3 | 77.5 / 96.6 |
| by scaffold | 38.3 | 48.4 | 66.2 / 82.5 |

The random split gives a more optimistic estimate (59% improvement over the median in RMSE versus 41% for this particular split). A single split is noisy, so the 5-fold result above is treated as the main one.

### Feature set comparison (5-fold by scaffold, random forest, 50 trees)

| Feature set | Features | RMSE, K |
|---|---|---|
| physical (chemistry-based) | 10 | 52.2 ± 1.0 |
| physical + 4 engineered | 14 | 50.7 ± 1.0 |
| top-11 by importance (selected inside folds) | 11 | 51.2 ± 1.3 |
| all RDKit descriptors | 216 | 46.5 ± 1.3 |
| all + 4 engineered | 220 | 46.3 ± 1.1 |

Contribution of each engineered feature (added to the physical set one at a time):

| Feature | Change in RMSE, K | Folds better than base |
|---|---|---|
| `force_over_freedom` | −1.08 | 5 of 5 |
| `HBD_per_atom` | −0.98 | 5 of 5 |
| `rot_per_atom` | −0.23 | 3 of 5 |
| `TPSA_per_mass` | +0.14 | 1 of 5 |

The two working features together: RMSE 50.6 ± 1.0.

## Conclusions

1. The model clearly beats the baseline even on new scaffolds: the error is roughly halved, and the result is stable across folds.
2. Scaffold splitting gives a noticeably higher error than random splitting, so part of the apparent quality comes from structural similarity to the training molecules.
3. Sets of 10-11 features, chosen by physics and automatically by model importance, performed comparably (RMSE 51-52 K). The full set is better by about 5 K.
4. Normalized features related to hydrogen-bond density and the "forces / flexibility" balance improve the small set by 1.5 K (about 3%), but add nothing to the full set: their information is already contained in other descriptors.
5. The remaining gap between the small and full sets (about 4 K) is not yet explained. Next step: add descriptor families (topological, VSA, functional groups, etc.) to the small set one at a time and see which gives the largest gain.

## Limitations

- **Crystal packing.** Melting point depends on crystal packing and polymorphism, which a SMILES string does not capture. Part of the error is fundamentally irreducible.
- **Noisy data.** Sources disagree, and values depend on sample purity and measurement method. Molecules with a spread above 20 K were removed, but the threshold is arbitrary.
- **Recovery by name via PubChem** was not fully verified by hand.
- **Scaffold splitting** protects acyclic molecules less (their scaffold is empty and each forms its own group): homologues may appear in both training and test. A stricter option (splitting by similarity clusters) was not tested.
- **Feature set comparison** used the same folds throughout, and some features were chosen after seeing results on them. For the final number this is mild overfitting; the gain from engineered features should be rechecked on a different split or with a different `random_state`.
- **Applicability domain** was not assessed: it is unknown for which molecules the predictions can be trusted.
- Stereochemistry, tautomers and polymorphs were not considered.

## Project structure

```
melting_point_project/
  data/
    raw/           source data (not in the repository)
    processed/     melting_points_clean.csv (not in the repository)
  notebooks/
    01_clean_data.ipynb
    02_features.ipynb
    03_models_split.ipynb
    03_models_scaffold.ipynb
    03_models_Kfold.ipynb
  README.md
```

Run order: `01_clean_data` → `02_features` → the modeling notebooks `03_models_split`, `03_models_scaffold`, `03_models_Kfold` and `03_models_fe`.

## Environment

Python 3.13.18, RDKit 2026.09.2, scikit-learn 1.6.1, pandas 2.2.3, numpy 2.4.6, pubchempy 1.0.5. Calculations were run in Google Colab. For reproducibility, `random_state=0` is fixed for the model; the `GroupKFold` split is deterministic.

## Next steps

- Break down descriptor families to explain the gap between the small and full sets.
- Assess the applicability domain (distance to the nearest training molecules by fingerprint).
- Split by similarity clusters (Butina) as a stricter test.
- Morgan fingerprints and gradient boosting for comparison.
- Physical consistency checks: ordering of isomers (ortho / meta / para), odd-even effect in homologous series.
- Error analysis by compound class for the molecules with the largest errors.

## References

- Jean-Claude Bradley Open Melting Point Dataset, figshare, DOI `10.6084/m9.figshare.1031637`.
- RDKit: https://www.rdkit.org
- scikit-learn: https://scikit-learn.org
