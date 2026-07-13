# WAY-EEG-GAL preprocessing

Tools for converting EEG recordings from the
[WAY-EEG-GAL dataset](https://figshare.com/collections/WAY_EEG_GAL_Multi_channel_EEG_Recordings_During_3_936_Grasp_and_Lift_Trials_with_Varying_Weight_and_Friction/988376)
(Luciw et al., 2014) into tensors aligned to events and ready for machine learning.

The original dataset is MATLAB centric and multimodal. This pipeline standardises the EEG files into
neural component matrices cleaned of artefacts, and into windows locked to events, matching the input
conventions of common deep learning libraries in Python.

Built at [Brainhack Rome 2025](https://brainhackrome.github.io/). The resulting tensors are used to
forecast motor behaviour from EEG recorded before movement onset in
[brainhack-rome-forecasting](https://github.com/matteo-d-m/brainhack-rome-forecasting).

## Pipeline

| Stage | Directory | Input | Output |
|---|---|---|---|
| 0. MATLAB to JSON | `mat_to_json/` | `HS_P*_S*.mat`, `P*_AllLifts.mat` | Session and marker JSON |
| 1. Bandpass and channel selection | `ica/bandpass_filter.py` | `HS_P*_S*.json` | `HS_P*_S*_processed.json` |
| 2. ICA and ICLabel | `ica/ica.py` | processed JSON | `HS_P1_S<run>_eeg.npy` (components × time) |
| 3. Windows aligned to events | `windows/sequences.py` | components and `P1_AllLifts.json` | `train_sequences.pkl` |

**Stage 0.** The `.mat` files are converted into two classes of JSON: *session files* (EEG, EMG, KIN,
ENV and MISC sections, each with signals, channel names and sampling rates) and *marker files*
(event tables with trial information). Original sampling rates are preserved.

**Stage 1.** Retains 14 frontocentral channels and applies a Butterworth bandpass with zero phase
(0.5 to 40 Hz), removing slow drifts and noise at high frequencies. Cleaned signals are written to
the `EEG.filtered_data` field.

**Stage 2.** Builds an MNE `Raw` object, assigns the standard 10/20 montage and an average
reference, then fits ICA to decompose the EEG into independent components. Components are classified
as brain, ocular, muscular or other by
[ICLabel](https://www.sciencedirect.com/science/article/pii/S1053811919304185)
(Pion-Tonachini et al., 2019). Components not classified as brain are discarded, and those retained
are saved as NumPy arrays.

**Stage 3.** Combines the component matrices with the event marker file. For each trial, two windows
are extracted around the `LEDOn` event: a **past** window of 1000 samples (roughly 2 s before) and a
**future** window of 1500 samples (roughly 3 s after), at the sampling rate of 500 Hz. The pairs are
collected across trials and pickled as the training set for a forecasting model.

## Dataset

EEG from 32 channels, 12 participants, 3,936 grasp and lift trials, recorded under unpredictable
changes of object weight and surface friction, sampled at 500 Hz. Open access, CC BY 4.0.

Data descriptor: Luciw, Jarocka & Edin (2014), *Multi-channel EEG recordings during 3,936 grasp and
lift trials with varying weight and friction*, **Scientific Data** 1:140047.
[Paper](https://www.nature.com/articles/sdata201447). Please cite it if you use the data.

## Installation

```bash
git clone https://github.com/annanotaro/way-eeg-gal-preprocessing
cd way-eeg-gal-preprocessing
pip install -r requirements.txt
```

Requires MNE-Python, `mne-icalabel`, NumPy and SciPy.

## Usage

```bash
python ica/bandpass_filter.py     # stage 1
python ica/ica.py                 # stage 2
python windows/sequences.py       # stage 3, writes train_sequences.pkl
```

## Assumptions and scope

- The sampling rate is assumed to be 500 Hz throughout. Adjust in `sequences.py` if this changes.
- Windows are aligned to `LEDOn`, as defined by the `StartTime` and `LEDOn` columns of the original
  marker files.
- EEG only. The dataset also contains EMG, kinematics and force channels, which are not processed
  here.

## Contributors

Developed at Brainhack Rome 2025 by [ADD NAMES].

**Anna Notaro**: [state your stages precisely, for example ICA and ICLabel artefact rejection
(`ica.py`) and extraction of sequences aligned to events (`sequences.py`)].

## Acknowledgements

This work grew out of the Brainhack project
[I Know What You Will Do: Forecasting Motor Behaviour from EEG Time Series](https://github.com/matteo-d-m/brainhack-rome-forecasting).
Thanks to the WAY-EEG-GAL authors and the WAY project for releasing the data and utilities publicly.
