# IBT — MATLAB Implementation on SP Cup 2017 Database

> A real-time tempo induction and beat tracking system implemented in MATLAB.  
> Based on: **"IBT: A Real-Time Tempo and Beat Tracking System"** — ISMIR 2010.

---

## Authors

| Name | Affiliation |
|---|---|
| João Lobato Oliveira | INESC Porto & LIACC/FEUP, Porto, Portugal |
| Fabien Gouyon | INESC Porto, Porto, Portugal |
| Luis Gustavo Martins | CITAR/UCP, Porto, Portugal |
| Luis Paulo Reis | LIACC/FEUP, Porto, Portugal |

Paper Link: https://ismir2010.ismir.net/proceedings/ismir2010-50.pdf

---

## Overview

IBT is an agent-based tempo induction and beat tracking system that processes audio causally in real time. It extends the competing-agents strategy of [BeatRoot](https://code.soundsoftware.ac.uk/projects/beatroot) to support continuous, frame-based spectral flux input — making it suitable for live audio, MIR pipelines, and robotic platforms.

The algorithm is evaluated on the **ISMIR 2004 Tempo Induction Contest** dataset (3199 instances) and a **1360-piece beat tracking** benchmark, achieving results competitive with state-of-the-art systems.

---

## Repository Structure

```
ibt/
├── beatVer1.m       # Main script: full IBT pipeline (pre-tracking + beat tracking)
├── specflux.m       # Spectral flux onset detection function (STFT-based)
├── peakpick.m       # Adaptive peak-picking on onset detection function
├── comp_dgt_fb.m    # Filter bank DGT (Gabor transform) — borrowed from LTFAT
├── winfuns.m        # Window function generator (Hamming, Hann, Blackman, etc.)
└── open_025.wav     # Example audio file for running beatVer1.m
```

---

## Requirements

- MATLAB R2016b or later (uses `designfilt`, `audioread`, `findpeaks`)
- Signal Processing Toolbox
- No additional toolboxes required (LTFAT helper `comp_dgt_fb` is bundled)

---

## Quick Start

1. Place all `.m` files and your `.wav` file in the same directory.
2. Update the filename in `beatVer1.m` if needed:
   ```matlab
   [y, fs] = audioread('open_025.wav');
   ```
3. Run the script:
   ```matlab
   >> beatVer1
   ```
4. The script outputs:
   - A plot of the autocorrelation periodicity function with detected tempo peaks
   - `beat_final` — a vector of estimated beat positions in **milliseconds**

---

## Algorithm Pipeline

```
Audio WAV file
      │
      ▼
┌─────────────────────────────────────┐
│  1. AUDIO FEATURE EXTRACTION        │
│     specflux.m                      │
│     • Hamming window, 1024 samples  │
│     • Hop size: 512 samples         │
│     • Spectral flux (half-wave rect)│
│     • Zero-phase Butterworth LP     │
│       filter (2nd order, IIR)       │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  2. PRE-TRACKING (5s window)        │
│                                     │
│  a) Period Hypotheses Induction     │
│     • Autocorrelation of SF         │
│     • Peak-pick in [50, 250] BPM    │
│       (240–1200 ms at 44100 Hz)     │
│     • Threshold: δ × rms(A(τ))      │
│                                     │
│  b) Phase Hypotheses Selection      │
│     • 30 uniformly spaced phases    │
│     • Beat train template matching  │
│     • Best phase per period: φᵢ     │
│                                     │
│  c) Agent Scoring & Ranking         │
│     • Raw score: beat–onset error   │
│     • Relational score: rewards     │
│       integer metrical ratios       │
│     • Final score Sᵢ = Sᵢrel / max │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  3. BEAT TRACKING (causal)          │
│     beatVer1.m                      │
│                                     │
│  Multi-agent system (max 30 agents) │
│  For each onset event:              │
│  • Best agent emits beat to output  │
│  • Inner window (±46.4 ms):         │
│    period & phase correction        │
│    P ← P + 0.25·err                 │
│    φ ← φ + P + 0.25·err            │
│  • Outer window (−0.2P, +0.4P):     │
│    spawn 3 child agents (C1, C2, C3)│
│    – C1: phase correction only      │
│    – C2: period + phase correction  │
│    – C3: half-correction (both)     │
│  • Agent culling: redundancy,       │
│    score gap > 80%, out-of-range    │
└──────────────┬──────────────────────┘
               │
               ▼
         beat_final (ms)
```

---

## Key Parameters

All parameters are set at the top of `beatVer1.m` and are easy to modify:

| Parameter | Value | Description |
|---|---|---|
| `hop_size` | `512` | STFT hop size in samples |
| `Plow` | `240` ms | Minimum beat period (= 250 BPM) |
| `Phigh` | `1200` ms | Maximum beat period (= 50 BPM) |
| `ind_win` | `5 × Fs / hop_size` frames | Induction window length |
| `Tin` | `46.4` ms | Inner tolerance window (±) |
| `Tl_out` | `0.2 × P` | Outer tolerance — left margin |
| `Tr_out` | `0.4 × P` | Outer tolerance — right margin |
| `phase_hyp` | `30` | Number of phase hypotheses |
| `child_score` factor | `0.9` (non-best), `0.8` (best) | Score inherited by child agents |
| Max agents | `30` | Agent pool size cap |

---

## File Reference

### `beatVer1.m`
Main script. Runs the full IBT pipeline end-to-end:
- Loads audio, computes spectral flux, applies LP filter
- Detects onsets via `peakpick`
- Initialises beat agents from period/phase hypotheses
- Runs the causal multi-agent beat tracking loop
- Outputs `beat_final` (beat positions in ms)

### `specflux.m`
```matlab
[SF, V0] = specflux(f, win_length, tgap)
```
Computes the **spectral flux** onset detection function of signal `f` using a Hamming-windowed Gabor transform. Half-wave rectifies the magnitude difference between consecutive frames.

| Arg | Description |
|---|---|
| `f` | Input audio signal (column vector) |
| `win_length` | STFT window length in samples (e.g. `1024`) |
| `tgap` | Hop size in samples (e.g. `512`) |
| `SF` | Spectral flux vector (1 × N frames) |
| `V0` | Raw STFT coefficients |

### `peakpick.m`
```matlab
peaks = peakpick(SF, thre, range, multi)
```
Picks significant local maxima from the onset detection function. A peak is accepted only if it exceeds the **local mean** over an asymmetric neighbourhood `[−multi×range, +range]` by more than `thre`.

| Arg | Description |
|---|---|
| `SF` | Onset detection function |
| `thre` | Threshold above local mean (e.g. `0.35`) |
| `range` | Neighbourhood radius (e.g. `3`) |
| `multi` | Asymmetric extension factor (e.g. `3`) |

### `comp_dgt_fb.m`
```matlab
coef = comp_dgt_fb(f, g, a, M)
```
Computes the **Discrete Gabor Transform** using a filter bank approach. Borrowed from the [LTFAT toolbox](http://ltfat.sourceforge.net/) by Peter L. Søndergaard (GPL). Handles boundary conditions periodically for both leading and trailing edges.

### `winfuns.m`
```matlab
g = winfuns(name, N)
g = winfuns(name, N, L)
g = winfuns(name, x)
```
Generates standard window functions by name. Used internally by `specflux` to produce the Hamming window for the STFT.

**Supported windows:** `hann`, `hamming`, `blackman`, `blackharr`, `nuttall`, `gauss`, `cos`, `rec`, `tri`, and many Nuttall variants (`nuttall10` through `nuttall03`).

---

## Agent Score Functions

### Raw score (within induction window)
For each beat prediction `bp` vs. local spectral flux maximum `m`:

```
If m ∈ Tᵢₙ:   Δs =  (1 − |err|/Tr_out) · (Pᵢ/Pmax) · SF(m)
If m ∈ Tₒᵤₜ:  Δs = −(|err|/Tr_out)     · (Pᵢ/Pmax) · SF(m)
```

### Relational score (agent ranking)
```
Sᵢrel = 10 · Sᵢraw + Σⱼ≠ᵢ r(nᵢⱼ) · Sⱼraw

       ⎧ 6 − n,   1 ≤ n ≤ 4
r(n) = ⎨ 1,       5 ≤ n ≤ 8
       ⎩ 0,       otherwise
```

where `nᵢⱼ` is the nearest integer ratio between periods `Pᵢ` and `Pⱼ`.

### Child agent generation (outer window)
| Child | Period | Phase |
|---|---|---|
| C1 | `P` | `φ + P + err` (phase-only fix) |
| C2 | `P + err` | `φ + (P+err) + err` (period + phase fix) |
| C3 | `P + 0.5·err` | `φ + (P+0.5·err) + 0.5·err` (half-fix) |

---

## Performance (from ISMIR 2010 paper)

### Global Tempo Estimation (a2 metric, % correct incl. metrical levels)

| Condition | Ballroom | Loops | Songs | Overall |
|---|---|---|---|---|
| IBT causal | 83% | 73% | 73% | 76% |
| IBT non-causal | 90% | 76% | 82% | **83%** |

### Beat Tracking P-score (1:1 metrical level)

| System | P-score | Excerpts |
|---|---|---|
| IBT causal | 74 | 558 |
| IBT non-causal | 81 | 613 |
| BeatRoot | 81 | 613 |

---

## License

- `beatVer1.m`, `specflux.m`, `peakpick.m` — based on work from **INESC Porto**, presented at ISMIR 2010.
- `comp_dgt_fb.m`, `winfuns.m` — © 2013 Nicki Holighaus, part of [NSGToolbox v0.1.0](http://nsg.sourceforge.net/), licensed under [CC BY-NC-SA 3.0](http://creativecommons.org/licenses/by-nc-sa/3.0/). `comp_dgt_fb` additionally licensed under GPL v3+.

---

## Citation

If you use this code, please cite:

```bibtex
@inproceedings{oliveira2010ibt,
  title     = {{IBT}: A Real-Time Tempo and Beat Tracking System},
  author    = {Oliveira, Jo{\~a}o Lobato and Gouyon, Fabien and Martins, Luis Gustavo and Reis, Luis Paulo},
  booktitle = {Proceedings of the 11th International Society for Music Information Retrieval Conference (ISMIR)},
  pages     = {291--296},
  year      = {2010}
}
```

---

## References

1. Dixon, S. — *Automatic extraction of tempo and beat from expressive performances.* Journal of New Music Research, 30(1):39–58, 2001.
2. Dixon, S. — *Onset detection revisited.* Proc. DAFx, 2006.
3. Dixon, S. — *Evaluation of the audio beat tracking system BeatRoot.* JNMR, 36(1):39–50, 2007.
4. Gouyon, F. et al. — *An experimental comparison of audio tempo induction algorithms.* IEEE TASLP, 14(5):1832–1844, 2006.
5. McKinney, M.F. et al. — *Evaluation of audio beat tracking and music tempo.* JNMR, 36(1):1–6, 2007.
