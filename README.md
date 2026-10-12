# ECG R-Peak Detection and Heart Rate Estimation (MATLAB)

**Short description:** A MATLAB demo that detects R-peaks in raw ECG signals using FFT-based baseline removal and a two-pass adaptive window-maximum filter, then estimates the average heart rate.

## Overview

This project takes a raw ECG recording, removes low-frequency drift, locates the R-peaks (the tall spikes of the QRS complex), and computes the average heart rate in beats per minute. The window size of the peak filter is tuned automatically from the data in a second pass, so no manual per-signal tuning is needed.

## Files

| File | Purpose |
|---|---|
| `ecgdemo.m` | Main script. Loads two sample ECGs, runs the full pipeline, prints heart rate, and plots results. |
| `ecgdemowinmax.m` | Helper function. A sliding window-maximum filter that keeps a sample only if it is the maximum of the window centered on it. |
| `ecgdemodata1.mat` | Sample 1 input data: 44,604 samples at 1000 Hz (about 44.6 s). |
| `ecgdemodata2.mat` | Sample 2 input data: 16,999 samples at 1000 Hz (about 17.0 s). |

## Requirements

- MATLAB (no extra toolboxes needed; uses only `fft`, `ifft`, `find`, `plot`, `stem`)
- The included data files `ecgdemodata1.mat` and `ecgdemodata2.mat` on the MATLAB path. Each contains:
  - `ecg`: 1 x N row vector of raw ADC ECG samples (stored as `uint16`)
  - `samplingrate`: sampling frequency in Hz (1000 for both files)

To use your own recording, save a `.mat` file with the same two variables and edit the `load` lines in `ecgdemo.m`.

## Usage

1. Place `ecgdemo.m`, `ecgdemowinmax.m` and the two `.mat` files in the same folder.
2. In MATLAB, run:
   ```matlab
   ecgdemo
   ```
3. The script processes Sample 1 and Sample 2, printing `Average Heart Rate =` followed by the value in BPM for each, and opens two figures per sample.

## How it works

1. **Baseline removal (FFT high-pass).** The FFT of the signal is taken and the lowest-frequency bins (roughly below 5 Hz, plus their mirrored counterparts) are zeroed. The inverse FFT gives a drift-corrected signal.
2. **First pass peak filter.** `ecgdemowinmax` runs with a window of about 571 ms (forced to an odd number of samples). It outputs a value only at samples that are the maximum within the window, and 0 elsewhere.
3. **Thresholding.** The first-pass output is scaled so its maximum is 7, then thresholded at 4 to give a binary peak map.
4. **Window size optimization.** The minimum distance between detected peaks is measured. The new window size is `2*distance - QRdistance`, where `QRdistance` is about 40 ms (made odd) and approximates QRS width.
5. **Second pass.** The filter runs again with the optimized window, and the output is thresholded to give the final R-peak positions.
6. **Heart rate.** The span between the first and last peak is divided by the number of detected peaks to get an average interval, then converted: `HR = 60 * samplingrate / averageInterval`.

### About `ecgdemowinmax`

An efficient sliding-maximum implementation. Instead of rescanning every window, it tracks the current maximum and its position, and rescans only when the maximum slides out of the window. It handles the signal edges separately (first half-window and last half-window). The output has the same length as the input.

## Output

For each sample:

- **Console:** the average heart rate in BPM.
- **Figure (Processing Stages):** six subplots showing the original ECG, FFT-filtered ECG, first-pass filter output, first-pass peaks, second-pass filter output, and final peaks.
- **Figure (Result):** the normalized ECG (green) with detected R-peaks overlaid as dashed black stems.

## Notes and known limitations

- The `ecg` variables are stored as `uint16`. MATLAB's `fft` does not accept integer types, and normalizing integers with `(ecg-min(ecg))/(max(ecg)-min(ecg))` would truncate the result. If you get an error at the `fft` line, add `ecg = double(ecg);` right after each `load` statement.
- The second-pass threshold (`< 4`) is applied to `filtered2` without the rescaling used in the first pass, so it depends on the signal's amplitude units. Scaling `filtered2` the same way as `filtered1` would make detection amplitude-independent.
- The heart rate divides by `length(positions2)` (number of peaks) instead of `length(positions2) - 1` (number of intervals), which slightly underestimates the interval and so slightly overestimates BPM, mainly on short recordings.
- The minimum inter-peak distance in pass one is sensitive to false detections. A single spurious close pair will shrink the second-pass window.
- The script assumes at least two peaks are found in the first pass (`positions(2)` is accessed directly).
- Results are best on clean, single-lead ECG with clear R-peaks.

## Possible improvements

- Use `peaks - 1` intervals for the BPM calculation, or compute beat-to-beat HR from `diff(positions2)`.
- Add a refractory-period check to reject false peaks.
- Parameterize the constants (5 Hz cutoff, 571 ms window, 40 ms QRS width, threshold 4 of 7).
- Compare detections against annotated data (e.g., MIT-BIH) to report sensitivity and precision.
