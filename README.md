# HEDGE: Enhancing Adversarial Resilience in Heterogeneous Intrusion Detection Ensembles via Directional Transfer-Aware Training

Research code, processed datasets, pretrained models, and evaluation artifacts accompanying the [HEDGE manuscript](HEDGE_paper.pdf).

**Authors:** Tuyen T. Nguyen, Minh Q. Tran, Hanh P. Du, Hai H. Le, and Hoa N. Nguyen.  
**Affiliation:** VNU University of Engineering and Technology, Vietnam.

HEDGE studies adversarial resilience in an intrusion-detection ensemble containing one deep neural network and four tree-based learners. Its core method, **Directional Transfer-Aware Training (DTAT)**, assigns adversarial examples to each protected model according to their generation source and attack mechanism. The experiments cover NSL-KDD and CSE-CIC-IDS2018 under seven attacks.

The results below are reported in the supplied manuscript. The repository contains an experimental code snapshot; consult [implementation and reproduction notes](#implementation-and-reproduction-notes) when rerunning it.

## Method

HEDGE has three stages, described in Section 4 and Algorithm 1 of the paper:

1. **Prepare the clean training reference.** Clean and split the data, then balance the training partition using KMeans representative selection for majority classes and Augmented WGAN generation for minority classes.
2. **Construct target-specific adversarial curricula.** Train clean seed models, generate source-specific adversarial pools from training records, and route the pools according to the DTAT policy below. Retain ground-truth labels, remove duplicates, and sample for class and attack coverage. The adversarial budget is approximately 50% of the balanced clean training set per target.
3. **Retrain and aggregate.** Retrain each component on clean data plus its curated adversarial set. The paper's inference design applies component-specific feature quantization to the same incoming record and combines normalized class scores using fixed weighted soft voting.

### Directional training policy

| Protected target | Direct adversarial training examples | Additional transferred training examples |
| --- | --- | --- |
| Each tree model | ZOO and HSJA generated against that same tree | FGSM, JSMA, PGD, DeepFool, and C&W generated against the DNN |
| DNN | All seven attacks generated against the DNN | None in the evaluated policy |

Transferred examples are supplied to the target without target-side attack optimization. Tree-to-DNN transfer is evaluated separately, but is outside the default DTAT training policy.

**Isolated adversarial training (IAT)** trains each tree on its own query-based examples. DTAT adds the DNN-originated gradient-based pool to the tree curriculum under the same intended adversarial budget. The DNN curriculum is shared by IAT and DTAT. This directional assignment is the contribution examined by the paper's IAT–DTAT comparison; balancing, feature squeezing, and weighted voting are supporting mechanisms.

### Models and inference settings

The following settings are reported in Table 3. CLI identifiers are the names used by the scripts and model files.

| Model | Paper name | CLI identifier | Voting weight | NSL-KDD FS bits | CSE-CIC-IDS2018 FS bits |
| --- | --- | --- | ---: | ---: | ---: |
| XGBoost | XGB | `xgb` | 0.3 | 6 | 6 |
| Gradient Boosting Machine | GBM | `gbm` | 0.2 | 8 | 2 |
| CatBoost | CBT | `catb` | 0.2 | 8 | 2 |
| Bagging meta-estimator | BME | `bagging` | 0.2 | 8 | 2 |
| Deep neural network | DNN | `dnn` | 0.1 | 2 | 2 |

The code also supports `histgbm` as an optional replacement for `gbm`. The five-model configuration above is the one described in the paper. Sequential and parallel ensemble scripts use the same aggregation rule; they differ in execution scheduling.

### Attacks

| Access to the generation source | Attacks | Supported generation sources |
| --- | --- | --- |
| Gradient-based / white-box | FGSM, JSMA, PGD, DeepFool, C&W L2 | DNN |
| Score-based / black-box | ZOO | DNN and tree models |
| Decision-based / black-box | HopSkipJumpAttack (HSJA) | DNN and tree models |

Attack parameters used by the CLI are defined in [the CIC2018 configuration](cli/hedge/configs/cic2018.py) and [the NSL-KDD configuration](cli/hedge/configs/nslkdd.py). The files in `params/atttacks/` are empty placeholders and are not the active configuration source.

## Evaluation contract

Sections 3.2–3.4 distinguish three conditions:

| Condition | Meaning |
| --- | --- |
| Direct component attack | The generation source and evaluated target are the same model. |
| Common-input directed transfer | An example generated against one source is passed unchanged to another target, without target-side re-optimization. |
| Component-conditioned ensemble stress diagnostic | Different ensemble branches may receive separately generated adversarial versions of the same original record before aggregation. |

The third condition is an oracle-style diagnostic. Its branch-specific inputs differ from deployment, where all components process deterministic transformations of one shared input. **Ensemble attack results in the paper use this diagnostic unless explicitly labeled common-input.**

Attack success rate (ASR) is conditional on the evaluated target being correct on the clean record:

```text
ASR = count(clean prediction correct AND adversarial prediction wrong)
      / count(clean prediction correct)
```

This measures multiclass exact-label disruption. A malicious record assigned to another malicious class still counts as a classification error; strict detection bypass additionally requires a benign prediction. ASR is generally different from `1 - adversarial accuracy`, and transferred target-side ASR does not require the source attack to succeed first.

## Datasets and bundled artifacts

The paper uses a stratified 70:30 record-level split after cleaning. Training data are balanced to 14,000 records per class. Test majority classes are capped at 6,000 records each; minority test classes retain their available records. These capped test distributions are experimental distributions rather than estimates of operational traffic prevalence.

| Dataset | CLI resource | Model features | Classes | Balanced training records | Test records |
| --- | --- | ---: | ---: | ---: | ---: |
| NSL-KDD | `nslkdd` | 41 | 5 | 70,000 | 17,336 |
| CSE-CIC-IDS2018 | `cic2018` | 62 | 12 | 168,000 | 51,129 |

The feature counts refer to model inputs. CSV files additionally contain labels and, for training data, provenance columns. The bundled CIC2018 CSVs also retain `Protocol`, which the current model preprocessor removes.

### Repository layout

```text
HEDGE_paper.pdf                  Manuscript
README.md                       Method, artifacts, and usage
cli/hedge/
  1_preprocessing_cic2018/       CIC2018 cleaning and splitting
  1_preprocessing_nslkdd/        NSL-KDD preparation
  2_major/                      Majority-class compression
  3_minor/                      Minority-class augmentation
  4_merge/                      Balanced train and sampled test assembly
  5_training/                   Native model training and evaluation
  6_classifiers/                ART wrapper evaluation
  7_generating_samples/         Attack generation and curriculum curation
  8_attack_evaluation/           Adversarial component and ensemble evaluation
  configs/                      Dataset paths, labels, and attack parameters
  preprocessing/                Shared feature and label preparation
  helpers/                      Model loading and ensemble utilities
src/
  training/                     Model implementations
  art_classifier/               ART adapters and feature squeezing
  art_generator/                Seven attack implementations
  resampling/                   KMeans and Augmented WGAN
  argparsers/                    Shared argument parser
  utils/                        Logging
encoders/                       Serialized dataset encoders
params/training/                Training hyperparameters
notebooks/evaluate.ipynb         Evaluation commands and stored outputs
balanced_data.7z                Four processed train/test CSVs
samples_and_models.7z.001        First of six archive volumes (.001–.006)
```

### Extract data and models

Run from the repository root with 7-Zip installed. Keep all six `samples_and_models.7z.*` volumes in the same directory and open the first volume:

```bash
7z x balanced_data.7z
7z x samples_and_models.7z.001
```

The archives contain approximately 1.23 GB of uncompressed files and create:

```text
data/
  nslkdd_merged_train_raw_processed.csv
  nslkdd_test_random_sample_clean_merged.csv
  cic2018_merged_train_raw_processed.csv
  cic2018_test_random_sample_clean_merged.csv
resources/
  NSLKDD/
    adv_samples/
      isolated/                 Curated isolated training samples
      robust/                   Curated DTAT training samples
      test/                     Source-model/attack-specific evaluation samples
    models/                     Baseline, isolated, and robust checkpoints
  CIC-2018/                     Same directory organization
```

Tree models use `.pkl`; DNN checkpoints use `.pth`. Checkpoint stems are `<resource>_<model>`, `<resource>_<model>_isolated_robust`, and `<resource>_<model>_robust`. Curated DTAT CSVs use `<resource>_<model>_robust_train_post.csv`.

The archives include isolated checkpoints for the four tree models. The IAT evaluation reuses the robust DNN checkpoint, consistent with the shared neural curriculum. Original raw benchmark files and intermediate preprocessing outputs are not bundled.

## Environment setup

Use **Python 3.10 or newer**. The dependency snapshot in `requirements.txt` targets Linux with CUDA packages. The paper reports experiments on an NVIDIA T4 GPU, 256 GB RAM, and a 48-core CPU; these describe the experimental machine rather than minimum requirements for every script.

For the recorded Linux environment:

```bash
python3.10 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
python -m pip install xgboost adversarial-robustness-toolbox prettytable
python -m pip install --no-deps -e .
```

The additional installation supplies packages imported by the code but omitted from the dependency files. Their versions are not pinned by this repository, so this setup is not a complete lock of the original experimental environment. For other platforms or CPU-only installations, select a compatible PyTorch build and adapt the CUDA-specific dependency snapshot.

The editable installation makes modules under `src/` importable. Run the following commands from the repository root so relative data, encoder, and configuration paths resolve correctly. The examples select `--device cpu`; compatible GPU environments can use the device options exposed by each script.

## Evaluate the bundled models

### Clean baseline

Evaluate the five individual baseline models and the native weighted ensemble:

```bash
python cli/hedge/5_training/model_evaluation.py \
  -r nslkdd --mode model --models xgb catb gbm bagging dnn \
  --device cpu --save-mt --save-cm

python cli/hedge/5_training/parallel_ensemble.py \
  -r nslkdd --device cpu --save-mt --save-cm
```

Replace `nslkdd` with `cic2018` for the other dataset. These commands write metrics and confusion matrices under `reports/<resource>/`.

### Common-input component transfer

Pass an explicit source CSV with `--adv-in` to apply the same adversarial records to all requested targets. For example, evaluate DNN-originated DeepFool examples on the DNN and four tree models:

```bash
python cli/hedge/8_attack_evaluation/classifier_evaluate_adv.py \
  -r nslkdd -m dnn xgb catb gbm bagging --device cpu \
  --adv-in resources/NSLKDD/adv_samples/test/nslkdd_dnn_deepfool_adv.csv
```

The DNN result is direct; the tree results are DNN-to-tree transfer. For the reverse direction, evaluate the DNN on a tree-originated query attack:

```bash
python cli/hedge/8_attack_evaluation/classifier_evaluate_adv.py \
  -r nslkdd -m dnn --device cpu \
  --adv-in resources/NSLKDD/adv_samples/test/nslkdd_xgb_hsja_adv.csv
```

ASR requires the clean and adversarial files to contain corresponding records in the same order. Use `--plain-in` when evaluating against a clean file other than the default test set. With `--attack` instead of `--adv-in`, the evaluator selects a model-specific file and may fall back to the DNN file, so retain the source identity when interpreting results.

### Component-conditioned ensemble stress diagnostic

Evaluate the archived complete HEDGE configuration with DTAT checkpoints and the paper's dataset-specific feature-squeezing settings:

```bash
python cli/hedge/8_attack_evaluation/classifier_parallel_ens_eval.py \
  -r nslkdd -a zoo hsja fgsm jsma pgd deepfool cw --device cpu \
  --using-robust --fs-enable \
  --fs-config xgb:6 catb:8 gbm:8 bagging:8 dnn:2

python cli/hedge/8_attack_evaluation/classifier_parallel_ens_eval.py \
  -r cic2018 -a zoo hsja fgsm jsma pgd deepfool cw --device cpu \
  --using-robust --fs-enable \
  --fs-config xgb:6 catb:2 gbm:2 bagging:2 dnn:2
```

This script selects a separate adversarial input file for each component, with DNN-source fallback where applicable. It prints accuracy, weighted F1, precision, recall, and ASR tables to the console.

The [evaluation notebook](notebooks/evaluate.ipynb) retains earlier experiment labels. Their mapping to the paper and existing command flags is:

| Paper configuration | Notebook label | Flags added to the stress diagnostic |
| --- | --- | --- |
| Base | No defense | None |
| HEDGE-FS | HEDGE-FS | `--fs-enable` |
| HEDGE-IAT | HEDGE-IR | `--using-isolated-robust --fs-enable` |
| HEDGE-DTAT | HEDGE-HT | `--using-robust` |
| HEDGE | HEDGE-ATH | `--using-robust --fs-enable` |

For each FS-enabled command, also supply the dataset-specific `--fs-config` shown above. The archived IAT command enables FS; preserve that detail when describing the executed ablation. The isolated selector prefers an isolated checkpoint, then falls back to a robust checkpoint and finally a baseline checkpoint. The robust selector also permits baseline fallback, so check which artifacts are available before attributing results to a training variant.

## Training and curriculum construction

The offline workflow is:

```text
Balanced clean training data
  -> train seed components
  -> generate source-specific adversarial training CSVs
  -> route and merge the appropriate source pools for each target
  -> deduplicate and sample across attacks and labels
  -> retrain each component on clean + curated adversarial data
```

The relevant entry points are:

| Script under `cli/hedge/` | Role |
| --- | --- |
| `5_training/model_training.py` | Train a component; `--adv-train` adds its curated adversarial CSV. |
| `7_generating_samples/generate_adv.py` | Generate samples for one source model and attack; use `--subset train` for curricula. |
| `7_generating_samples/merge_training_samples.py` | Assemble a target's adversarial pool and remove duplicates with clean training data. |
| `7_generating_samples/postprocess_merge.py` | Select examples across attacks and labels using `--sample-threshold`. |
| `7_generating_samples/label_distribution.py` | Inspect label-by-attack counts in the curated datasets. |

For example, retrain an XGB component using the bundled NSL-KDD DTAT curriculum and save it separately from the supplied checkpoint:

```bash
python cli/hedge/5_training/model_training.py \
  -r nslkdd -m xgb --adv-train --device cpu \
  --model-out resources/NSLKDD/retrained_models/nslkdd_xgb_robust.pkl
```

By default, `--adv-train` reads `resources/NSLKDD/adv_samples/robust/nslkdd_xgb_robust_train_post.csv`. Use `--adv-in` to select another curated training file. Rebuilding all curricula requires generating the corresponding training pools first; the bundled archives contain curated training sets and test attacks rather than every intermediate source pool.

## Paper-reported results

The central comparison in Table 8 measures the effect of DTAT relative to IAT. These are **fixed-split results under the component-conditioned ensemble stress diagnostic**, with lower ASR indicating fewer initially correct predictions disrupted.

| Dataset | Base DeepFool ASR | IAT DeepFool ASR | DTAT DeepFool ASR, without FS | Complete HEDGE clean accuracy |
| --- | ---: | ---: | ---: | ---: |
| NSL-KDD | 98.60% | 17.52% | 1.43% | 99.46% |
| CSE-CIC-IDS2018 | 88.05% | 6.43% | 2.03% | 95.95% |

Complete HEDGE, including FS, also has DeepFool ASRs of 1.43% and 2.03%. Its ASRs across all seven attacks are:

| Attack | NSL-KDD | CSE-CIC-IDS2018 |
| --- | ---: | ---: |
| ZOO | 0.10% | 0.03% |
| HSJA | 0.11% | 0.59% |
| FGSM | 0.24% | 0.02% |
| JSMA | 0.03% | 0.07% |
| PGD | 0.09% | 0.07% |
| DeepFool | 1.43% | 2.03% |
| C&W | 2.23% | 0.45% |

DTAT's gains are attack dependent: HSJA ASR increases slightly relative to IAT in the DTAT-only configuration. Feature squeezing provides a small, non-uniform additional effect after DTAT.

Separately, common-input component results in Table 5 show that DNN-originated DeepFool examples yield tree-target ASRs of 69.25–80.17% on NSL-KDD and 49.35–88.40% on CSE-CIC-IDS2018. These support the selected incoming exposure pathway under the evaluated configuration, without establishing a universal architecture-only transfer ordering.

## Implementation and reproduction notes

The manuscript specifies the experimental method, while the supplied code and saved notebook outputs are a snapshot of the implementation. Exact numerical reproduction requires resolving the following points:

- **Attack objectives.** The current PGD configuration sets `targeted=True`, while the generator passes ground-truth labels as targets. JSMA also receives ground-truth labels. Audit objective and target-label handling before interpreting the near-zero results as robustness; the paper discusses the need for such attack audits in Section 5.5.
- **Score aggregation.** Native DNN inference applies softmax. The ART DNN wrapper passes through network outputs without an explicit softmax, and ART ensemble helpers combine these with tree probabilities. Verify score normalization against the paper's soft-voting definition before comparing these paths.
- **Feature squeezing.** The ART defence currently uses fixed `clip_values=(0, 1)` on the input passed to the wrapper; shared preparation retains unscaled continuous features and ordinal categories. The paper describes quantization of mutable normalized coordinates. In addition, the stage-8 ensemble script enables FS for adversarial predictions but does not pass FS parameters to its clean baseline. These details need reconciliation for a matched reproduction.
- **Record correspondence.** Adversarial evaluators compute ASR by row position and trim unequal lengths. They do not join on persistent sample IDs. Random subsampling, dropped generation batches, or differently ordered component streams require explicit alignment before evaluation.
- **Configuration snapshots.** Table 3 reports 100 GBM iterations; `params/training/gbm.json` currently specifies 300 estimators. Dependency files also omit some required packages and do not fully capture the environment used to create serialized artifacts.
- **Older entry points.** Some stage-1 scripts still access constants at module level after the configuration classes were introduced. Both stage-6 ART ensemble scripts omit the required `clip_values` argument. The examples above use bundled processed data and stage-5/stage-8 evaluation entry points.

The `--using-robust` naming convention selects existing robust artifacts; it does not itself construct or verify a DTAT curriculum. Keep the generation source, attack, protected target, preprocessing, and selected checkpoint associated with every reported experiment.

## Scope of the evidence

As discussed in Sections 4.5 and 5.5 of the paper:

- Attacks operate on post-featurization vectors. Feature-domain constraints do not establish packet-level realizability or preservation of malicious network behavior.
- Training and evaluation use the same seven attack families on disjoint records. The results establish attack-aware empirical evidence under the recorded configurations.
- The study does not provide certified robustness or a fully adaptive end-to-end attack evaluation through repair, quantization, and ensemble aggregation.
- Results use one fixed split, without repeated random seeds or confidence intervals. The capped benchmarks do not establish temporal, cross-dataset, or deployment-prevalence generalization.
- Signature detection, traffic sensing, and mitigation in Figure 1 illustrate deployment context outside the evaluated implementation. No latency or throughput guarantee is established by the supplied experiments.

## Citation

The following entry identifies the supplied manuscript; update publication metadata when a final bibliographic record is available.

```bibtex
@misc{nguyen2026hedge,
  title = {{HEDGE}: Enhancing Adversarial Resilience in Heterogeneous Intrusion Detection Ensembles via Directional Transfer-Aware Training},
  author = {Nguyen, Tuyen T. and Tran, Minh Q. and Du, Hanh P. and Le, Hai H. and Nguyen, Hoa N.},
  year = {2026},
  note = {Manuscript accompanying the HEDGE research code and experimental artifacts}
}
```
