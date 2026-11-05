Movement-Aware ECG Monitoring and Classification
A research project that aims to distinguish normal ECG segments,
movement-corrupted ECG segments, and potentially abnormal cardiac
patterns using ECG and body-movement signals. The intended deployment
target is a Raspberry Pi 3.
Medical disclaimer: This is a research prototype, not a medical
device. Its predictions must not be used to diagnose, treat, or rule
out a heart condition. Clinical validation is required before
real-world medical use.

1. Project overview
ECG recordings collected while a person walks or runs can contain: -
Genuine physiological changes, such as a naturally increased heart rate
during exercise. - Measurement disturbances, such as motion artifact,
muscle activity, electrode movement, and poor electrode contact. -
Potentially abnormal cardiac patterns.

The project aims to distinguish unreliable ECG measurements from
potentially meaningful cardiac patterns. Movement information from an
accelerometer can help assess signal quality.
Intended output classes
  Output                              Meaning
  NORMAL                            A usable ECG segment without a
target abnormal pattern identified
by the model

  MOTION_ARTIFACT                   An ECG segment that is
substantially corrupted or
unreliable due to movement or
recording noise
POTENTIALLY_ABNORMAL              A usable ECG segment containing a
pattern associated with an
annotated abnormality and requiring
further review
These are research model outputs. NORMAL does not prove a person is
healthy, and POTENTIALLY_ABNORMAL is not a clinical diagnosis.
2. Proposed architecture
The recommended design has two stages:
1. Signal-quality model: estimates whether an ECG segment is
   reliable or corrupted. It can use ECG and synchronized accelerometer
   signals.
2. Cardiac-pattern model: analyzes usable ECG segments for selected
   patterns represented in clinically annotated ECG datasets.
ECG sensor -----------------+
                            +--> Synchronize and preprocess
Accelerometer (X, Y, Z) ----+
                                     |
                                     v
                            Signal-quality model
                              /             \
                       Reliable ECG      Corrupted ECG
                            |                  |
                            v                  v
                    Cardiac-pattern       MOTION_ARTIFACT
                         model
                      /        \
                  NORMAL    POTENTIALLY_ABNORMAL
If signal quality is poor, the system should flag the segment as
MOTION_ARTIFACT or UNRELIABLE_SIGNAL rather than make a confident
cardiac classification from corrupted data.
3. Datasets
Check each dataset's current documentation, channel definitions,
annotations, license, and terms before downloading or redistributing
data.
3.1 ScientISST MOVE
- Source: PhysioNet -- ScientISST
  MOVE
- Purpose: study physiological signals and movement during activities
  including walking and running.
- Useful signals: chest ECG and accelerometer channels, plus other
  wearable signals depending on the recording.
- Intended use: investigate movement-related signal quality and
  develop/evaluate the signal-quality model.
Labeling caution: Do not label every walking or running segment as
artifact. Movement does not automatically make an ECG unusable. Inspect
the ECG and use documented annotations or a defensible signal-quality
labeling protocol.
3.2 MIT-BIH Arrhythmia Database
- Source: PhysioNet -- MIT-BIH Arrhythmia
  Database
- Purpose: develop and evaluate detection of selected annotated ECG
  beat patterns.
- Intended use: train/evaluate the cardiac-pattern model using expert
  annotations.
Labeling caution: MIT-BIH is not a synchronized walking/running
dataset. Its abnormal beat labels do not establish how the model
performs during exercise. Define target annotation symbols using the
database documentation and clinical advice. Do not automatically treat
every non-normal annotation as the same abnormality.
3.3 MIT-BIH Noise Stress Test Database
- Source: PhysioNet -- MIT-BIH Noise Stress Test
  Database
- Purpose: evaluate robustness to baseline wander, muscle artifact,
  and electrode-motion artifact.
- Intended use: supplementary artifact robustness testing.
Added or simulated noise is useful for controlled experiments, but it is
not a complete substitute for real movement recordings.
Dataset integration warning
Do not simply merge these datasets and label all ScientISST MOVE
recordings as normal and all MIT-BIH abnormal beats as abnormal. They
differ in subjects, devices, leads, sampling rates, recording
conditions, and annotation schemes. This shortcut can create
dataset/source leakage and misleading performance.

Keep signal-quality and cardiac-pattern tasks separate initially.
Validate the integrated system on appropriately annotated recordings
containing both movement and cardiac conditions.
4. Suggested repository structure
Adapt this proposed structure as the codebase develops.
ecg-movement-classification/
├── README.md
├── requirements.txt
├── .gitignore
├── data/
│   ├── raw/                 # Original downloaded files; do not commit
│   ├── interim/             # Temporary converted/cleaned files
│   └── processed/           # Model-ready windows and labels
├── notebooks/
│   ├── 01_dataset_exploration.ipynb
│   ├── 02_ecg_visualization.ipynb
│   └── 03_model_evaluation.ipynb
├── src/
│   ├── data/
│   │   ├── convert_edf_to_csv.py
│   │   ├── load_records.py
│   │   └── annotations.py
│   ├── preprocessing/
│   │   ├── ecg_filter.py
│   │   ├── synchronization.py
│   │   ├── segmentation.py
│   │   └── feature_extraction.py
│   ├── models/
│   │   ├── signal_quality_model.py
│   │   └── cardiac_pattern_model.py
│   ├── training/
│   │   ├── train_signal_quality.py
│   │   └── train_cardiac_model.py
│   ├── evaluation/
│   │   ├── metrics.py
│   │   ├── confusion_matrix.py
│   │   └── subject_split.py
│   └── deployment/
│       ├── export_tflite.py
│       └── raspberry_pi_app.py
├── models/
│   ├── signal_quality.tflite
│   └── cardiac_pattern.tflite
├── reports/
│   ├── figures/
│   └── evaluation/
└── tests/
The names are suggestions; update this section to match the actual
codebase.
5. Data preparation
1. Download datasets from their official PhysioNet pages.
2. Read the documentation and identify ECG, accelerometer,
   sampling-rate, and annotation channels.
3. Preserve original recordings and store converted files separately.
4. Check missing values, clipping, flat lines, sensor disconnection,
   and signal units.
5. Synchronize ECG and accelerometer using timestamps. Do not assume
   their sampling rates are identical.
6. Apply suitable ECG preprocessing and document filter settings and
   any resampling.
7. Divide recordings into fixed windows. An initial experiment could
   use 5-second ECG windows with 50% overlap.
8. Assign labels using annotations and a documented signal-quality
   review procedure.
9. Split data by participant/recording before generating overlapping
   windows to prevent leakage.
Example window table
Actual columns depend on the source recording and preprocessing choices.
  recording_id   subject_id       start_time ECG samples ACC X/Y/Z      label
  record_001     subject_01            0.0 s ECG window  synchronized   NORMAL
window         
record_001     subject_01            2.5 s ECG window  synchronized   MOTION_ARTIFACT
window
The rows are illustrative, not real dataset labels.
6. Model development
Stage 1: Signal-quality classification
Inputs: ECG window and, where available, synchronized three-axis
accelerometer window.

Initial baseline: lightweight 1D CNN, compared with a feature-based
model such as Random Forest or gradient boosting.

Potential features include ECG signal-quality indices, baseline
variation, high-frequency noise, R-peak detection consistency,
accelerometer magnitude, and movement intensity.
Stage 2: Cardiac-pattern classification
Input: ECG windows that pass the signal-quality threshold.

Initial baseline: lightweight 1D CNN. Compare it with a beat-feature
model and evaluate whether beat-centered windows are more appropriate
for the selected annotations.

Start with clearly defined target categories, such as normal beats and
selected annotated premature beats. Document annotation mapping and
class balance. Consider a multi-class model only if there are enough
examples and reliable labels for each class.
Model selection
Start with simple baselines: - 1D CNN - Random Forest or gradient
boosting on engineered features - CNN-LSTM only if experiments show a
meaningful benefit

Use class weighting or training-only resampling if classes are
imbalanced. Never resample the test set to improve reported results.
7. Training and evaluation
Prevent data leakage
- Split by subject/patient or recording, not randomly by overlapping
  windows.
- Keep test subjects completely unseen during training and model
  selection.
- Fit normalization and feature-scaling parameters on training data
  only.
- Do not allow windows from the same original recording to appear in
  both training and test sets.
- Keep a separate validation set for model selection and threshold
  tuning.
Metrics
Report metrics separately for each stage and for the integrated
three-class system: - Confusion matrix - Per-class precision,
recall/sensitivity, and F1-score - Specificity for defined binary
tasks - Macro-F1 and balanced accuracy - AUROC or AUPRC where
appropriate - False alarms per hour - Percentage of time marked as
unusable - Performance by activity, subject, and recording condition -
Inference latency and memory use on Raspberry Pi 3

For cardiac screening, sensitivity and false-alarm rate are particularly
important. Report confidence intervals where feasible. Do not rely on
accuracy alone.
Validation limitation
Performance on MIT-BIH alone does not demonstrate abnormality detection
during walking or running. The integrated system needs external
validation on recordings with reliable cardiac annotations and
synchronized movement/activity information. Until then, describe
exercise-time abnormality detection as an intended capability, not a
proven result.
8. Raspberry Pi 3 deployment
Train and evaluate models on a development computer. Use the Raspberry
Pi 3 for inference after model validation.
1. Export the trained model to TensorFlow Lite or another supported
   runtime.
2. Test float and quantized models; verify that quantization does not
   unacceptably reduce sensitivity.
3. Install a compatible runtime and signal-processing dependencies on
   the Pi.
4. Stream ECG and accelerometer samples with timestamps.
5. Maintain synchronized rolling windows and run inference at the
   selected interval.
6. Display the class, confidence, signal-quality state, and timestamp.
7. Log predictions for later review, with appropriate privacy and
   security safeguards.
Measure actual inference time, CPU and memory usage, temperature, and
dropped samples on the target Raspberry Pi 3. Do not assume desktop
performance transfers to the Pi.
9. Installation and running
The commands below are examples. Adjust them to the scripts and
dependencies that actually exist in this repository.
python -m venv .venv

# Windows
.venv\Scripts\activate

# Linux / macOS
source .venv/bin/activate

# Install dependencies after creating requirements.txt
pip install -r requirements.txt
Example development workflow:
python -m src.data.load_records
python -m src.preprocessing.segmentation
python -m src.training.train_signal_quality
python -m src.training.train_cardiac_model
python -m src.evaluation.metrics
python -m src.deployment.export_tflite
These are illustrative entry points. They work only after the
corresponding modules and command-line interfaces have been implemented.
10. Reproducibility and responsible use
For every experiment, record: - Dataset name, version, source, and
download date - Subject/recording split - ECG lead and sensor channel
names - Sampling rates and units - Filter settings and resampling
method - Window length and overlap - Annotation-to-class mapping - Model
architecture and training parameters - Random seeds and software
versions - Per-class evaluation results and confusion matrix

Do not commit raw patient or research data to a public repository unless
the dataset license and privacy requirements explicitly allow it. Do not
publish identifiable recordings or sensitive metadata.
11. Project checklist
- [ ] Download and inspect ScientISST MOVE
- [ ] Convert required EDF recordings to a documented tabular format
- [ ] Identify and verify ECG and accelerometer channels
- [ ] Define a reproducible signal-quality labeling protocol
- [ ] Prepare MIT-BIH annotations and target beat classes
- [ ] Create subject-independent train/validation/test splits
- [ ] Train and evaluate signal-quality baseline
- [ ] Train and evaluate cardiac-pattern baseline
- [ ] Evaluate integrated three-class predictions
- [ ] Validate on movement recordings with cardiac reference
  annotations
- [ ] Export and test models on Raspberry Pi 3
- [ ] Document limitations and clinical-validation requirements
12. References
- ScientISST MOVE --
  PhysioNet
- MIT-BIH Arrhythmia Database --
  PhysioNet
- MIT-BIH Noise Stress Test Database --
  PhysioNet
Project type: Research / machine learning / biomedical signal
processing
Target device: Raspberry Pi 3
Primary signals: ECG and accelerometer
Intended outputs: Normal, movement artifact, potentially abnormal