# Resonant Plasma Inverter Control & DSP Simulation Pipeline

A comprehensive Python-based simulation package for **Sub-Sample Interpolated Zero-Crossing Detection (ZCD)** and **Phase-Shift PWM Control** tailored for high-frequency Resonant Plasma Power Supplies.

This repository provides a complete, self-contained, and reproducible execution framework that models real-world power electronics phenomena—including harmonic distortion, high-frequency switching ripple, Additive White Gaussian Noise (AWGN), Digital Signal Processing (DSP) frequency filtering, microsecond-accurate ZCD, and MOSFET Zero-Voltage Switching (ZVS) power loss analysis.

---

## 👥 Team Members

* **2520040061** - Aditya (ECE, KL University)
* **2520040108** - Charvik (ECE, KL University)
* **2520040076** - Gnaneshwar (ECE, KL University)

---

## 📋 Table of Contents

- [Project Overview](#project-overview)
- [System Operational Parameters](#system-operational-parameters)
- [Directory Structure & File Inventory](#directory-structure--file-inventory)
- [Execution Guide & How Files Work Together](#execution-guide--how-files-work-together)
- [Module & Function Breakdown](#module--function-breakdown)
- [Installation & Requirements](#installation--requirements)
- [How to Run the Pipeline](#how-to-run-the-pipeline)
- [Expected Outputs & Verification](#expected-outputs--verification)

---

## ⚙️ Project Overview

In high-power resonant plasma power supplies, switching losses in power MOSFETs/IGBTs dictate overall system efficiency. Achieving **Zero-Voltage Switching (ZVS)** requires microsecond-accurate detection of zero-crossing points in the resonant current/voltage waveforms despite heavy switching noise and dynamic plasma load harmonics.

This project simulates a complete end-to-end DSP control pipeline:
1. **Signal Generation:** Synthesizes a 50 Hz AC reference wave distorted by 3rd and 5th harmonics, 25 kHz resonant switching ripple, and AWGN noise.
2. **DSP Filtering:** Uses Fast Fourier Transform (FFT) analysis and an ideal Low-Pass Filter (LPF) to isolate the fundamental signal.
3. **Sub-Sample ZCD:** Implements a hysteresis-gated state machine with sub-sample linear root interpolation to eliminate zero-crossing jitter down to microsecond scales.
4. **Phase-Shift PWM Generation:** Derives synchronized 3.3V logic gate pulses with configurable phase offsets.
5. **ZVS & Efficiency Simulation:** Models MOSFET Drain-Source voltage ($V_{DS}$) discharge during dead time and quantifies power loss reduction (Hard vs. ZVS switching).
6. **Diagnostic Dashboard:** Renders and saves a detailed 6-panel analytical plot (`zcd_25khz_system_dashboard.png`).

---

## 🎛️ System Operational Parameters

| Parameter | Symbol | Value | Description |
| :--- | :--- | :--- | :--- |
| **Sampling Frequency** | $F_s$ | $500\text{ kHz}$ | 20× oversampling relative to 25 kHz |
| **AC Fundamental Frequency** | $f_{\text{line}}$ | $50\text{ Hz}$ | $20\text{ ms}$ fundamental period |
| **Resonant Switching Frequency** | $f_{\text{sw}}$ | $25\text{ kHz}$ | $40\text{ }\mu\text{s}$ switching period |
| **Hysteresis Band** | $V_{\text{hyst}}$ | $\pm 0.15\text{ V}$ | Dual-threshold noise-rejection band |
| **Phase-Shift Angle** | $\phi$ | $30.0^\circ$ | Target Phase-Shift PWM delay angle |
| **DC Bus Voltage** | $V_{\text{DC}}$ | $100.0\text{ V}$ | Inverter bridge input voltage |
| **MOSFET Parasitic Capacitance** | $C_{\text{oss}}$ | $1.0\text{ nF}$ | Output capacitance for ZVS power loss |

---

## 📁 Directory Structure & File Inventory

```text
Resonant_Plasma_Inverter_Pipeline/
├── main.py                        # Primary execution script coordinating the pipeline
├── signal_generator.py            # Module: AC waveform synthesis engine
├── dsp_filter.py                  # Module: FFT frequency analysis & LPF filter
├── zcd_detector.py                # Module: Sub-sample interpolated ZCD state machine
├── pwm_zvs_driver.py              # Module: Phase-Shift PWM & MOSFET ZVS power loss model
├── config.json                    # Configuration file for system parameters
├── README.md                     # Markdown project guide and documentation
└── zcd_25khz_system_dashboard.png # Output graphic generated upon execution

```

---

## 🔄 Execution Guide & How Files Work Together

The execution pipeline follows a modular data-flow architecture where each stage consumes the output of the preceding layer:

```text
[config.json] ──> [main.py]
                   ├──> [signal_generator.py] ──(Noisy Waveform)──> [dsp_filter.py]
                   │                                                     │
                   │                                              (Filtered Signal)
                   │                                                     │
                   │                                                     ▼
                   ├──> [pwm_zvs_driver.py] <──(Timestamps)────── [zcd_detector.py]
                   │            │
                   │      (Gate Signals & V_DS)
                   │            │
                   └────────────┴──> Console Terminal Logs + Graphic Dashboard

```

1. **`main.py`** loads operational settings from **`config.json`**.
2. **`signal_generator.py`** synthesizes raw time-series arrays containing $50\text{ Hz}$ fundamental, harmonics, $25\text{ kHz}$ ripple, and AWGN noise.
3. **`dsp_filter.py`** transforms the raw signal to the frequency domain via FFT, truncates components above $1\text{ kHz}$ via LPF mask, and reconstructs the cleaned wave using IFFT.
4. **`zcd_detector.py`** evaluates the filtered wave through a hysteresis state machine, identifying exact zero-crossing timestamps using sub-sample linear root interpolation.
5. **`pwm_zvs_driver.py`** uses extracted timestamps to create phase-delayed $3.3\text{V}$ PWM gate signals and simulates $V_{DS}$ discharge across a $1.2\text{ }\mu\text{s}$ dead-time window.
6. **`main.py`** outputs timing jitter, computes power losses ($W$), and renders the 6-panel diagnostic visualization.

---

## 🧩 Module & Function Breakdown

### `signal_generator.py`

Synthesizes the physical AC input waveform.

* Injects a pre-roll buffer ($-5\text{ ms}$ to $0\text{ ms}$) to allow filter transient response stabilization.
* Adds 3rd harmonic ($8\%$), 5th harmonic ($5\%$), and high-frequency $25\text{ kHz}$ switching ripple ($6\%$).
* Injects additive white Gaussian noise to hit target Signal-to-Noise Ratio ($12\text{ dB}$).

### `dsp_filter.py`

Performs frequency-domain signal extraction.

* Computes Real Fast Fourier Transform (`np.fft.rfft`).
* Zeroes out spectral components above $1\text{ kHz}$ (eliminating ripple and noise).
* Reconstructs cleaned time-domain wave via Inverse FFT (`np.fft.irfft`).

### `zcd_detector.py`

High-precision root extraction engine.

* Signal must drop below $-V_{\text{hyst}}$ ($-0.15\text{V}$) before arming to avoid false triggers around zero.
* Applies linear interpolation between adjacent samples $(t_0, y_0)$ and $(t_1, y_1)$ upon rising crossing $+V_{\text{hyst}}$:

$$t_{\text{exact}} = t_0 + \frac{-y_0}{y_1 - y_0} (t_1 - t_0)$$



### `pwm_zvs_driver.py`

Simulates gate drive logic and power stage ZVS transitions.

* Delays pulse turn-on by $\Delta t = \frac{\phi}{360^\circ} \times T_{\text{line}}$.
* Models resonant capacitor discharge ($C_{\text{oss}}$) bringing $V_{DS}$ from $100\text{V}$ down to $0\text{V}$ using a cosine decay profile over $1.2\text{ }\mu\text{s}$ dead time.

---

## 📦 Installation & Requirements

### Prerequisites

* Python 3.8+
* `numpy`
* `matplotlib`

### Installation

Open terminal or command prompt in your working folder and install required dependencies:

```bash
pip install numpy matplotlib

```

---

## 🚀 How to Run the Pipeline

Run the master execution script:

```bash
python main.py

```

---

## 📊 Expected Outputs & Verification

### 1. Terminal Output Verification

```text
================================================================================
EXECUTING RESO-ZCD SYSTEM SIMULATION AT f_sw = 25.0 kHz
================================================================================
True Ground-Truth Crossings (ms):  [ 0. 20. 40.]
Detected ZCD Timestamps (ms):     [ 0.1885898  20.17395879 40.22905896]
Sub-Sample Timing Jitter:         23.304 microseconds
Hard Switching Power Loss:        0.1250 W per MOSFET
ZVS Operating Power Loss:         0.03075900 W per MOSFET
Efficiency Factor:                Loss reduced by 4.1x
================================================================================

```

### 2. Analytical Diagnostic Dashboard

The image file **`zcd_25khz_system_dashboard.png`** will be generated with 6 plots:

1. **Corrupted Input AC Waveform:** Input wave showing noise and ripple.
2. **FFT Spectrum Analysis:** Spectral breakdown showing $\le 1\text{ kHz}$ LPF isolation.
3. **Hysteresis Gating & ZCD:** Filtered wave with $\pm 0.15\text{V}$ thresholds and root markers.
4. **Synchronized PWM Output:** Phase-shifted 3.3V gate drive logic.
5. **MOSFET $V_{DS}$ Waveform:** Soft-switching voltage profile across 100V DC bus.
6. **Microsecond Zoom-In:** Microsecond view confirming $V_{DS}$ decay prior to gate turn-on.

```

```
