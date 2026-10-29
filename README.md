# GRU-Based Chute-Level Forecasting & Carrier-Speed Advisory

Short-term chute-level forecasting and operator decision support for sugarcane feeding systems, combining electronic mill records, recurrent learning, and a separate bounded carrier-speed advisory.

## Overview

Variable cane feeding can cause the material level in a mill chute to rise or fall outside its operating band. Operators need advance information about these changes and practical guidance on feeding adjustments.

This project is designed to forecast chute level approximately **30 seconds ahead** using a compact **Gated Recurrent Unit (GRU)** network. Inputs include recent electronic chute-level readings, carrier drive signals, and shredder steam-flow information. A separate supervisory policy converts the forecast into warnings and coordinated **Hz or RPM recommendations** for operator review.

The design combines measured-data forecasting with a laptop application for recorded replay, additional operating scenarios, and a clear dashboard. The target operating band for the studied mill is **60–75%, inclusive**.

## System Flow

![GRU forecasting workflow from electronic mill records to operator guidance](images/system-flow.png)

*Workflow and screen concept. Display values are illustrative.*

1. **Mill records:** electronic level measurements, carrier drive records, steam flow, and status/quality information.
2. **Preprocessing:** timestamp alignment, quality checks, chronological splits, training-only scaling, and history-window construction.
3. **Model development:** GRU training and validation, with baseline comparisons and feature ablations.
4. **Testing application:** held-out recorded replay and a separate generated-scenario mode.
5. **Inference and advisory:** history buffer, future-level prediction, and bounded carrier-setting suggestions.
6. **Operator display:** current and predicted level, target band, warning, recommended setting, change, reason, and input freshness.

## Input Data

| Signal group | Information |
| --- | --- |
| Chute-level sensor | Electronically recorded level (%), timestamp, and quality flags |
| Auxiliary carrier | Recorded RPM and/or VFD frequency (Hz), motor current, voltage, and command setting where available |
| Main carrier | Corresponding drive signals with carrier identifier and timestamp |
| Cane carrier | Corresponding drive signals, with carrier order and transport delays documented |
| Shredder turbine | Recorded steam flow with confirmed units |
| Operating context | Run/stop status, faults, production-run identifier, and data-source information |
| Equipment metadata | Nameplate ratings, drive limits, gearing, and carrier coordination rules |

The primary data strategy uses authorized **PLC/SCADA exports, drive logs, and electronic level records**. Manual chute-level entry is not part of the forecasting input workflow. Motor current is a mechanical-loading proxy, not a direct measurement of cane weight or throughput. Hz and RPM require a verified mapping before conversion.

## Model Design

The initial sequence-model configuration is:

```text
12-step multivariate history at 5-second intervals
                       |
        Unidirectional GRU: 32 hidden units
                       |
            Dense: 16 units + ReLU
                       |
            Dense: 1 unit + Sigmoid
                       |
       Chute level at +30 seconds (0–100%)
```

The training design uses **Huber loss**, **AdamW**, gradient clipping, weight decay, and validation-based early stopping. History length, hidden size, and layer count are refined through a limited validation search. The initial timing is adjusted to actual logging intervals and process delays.

Comparison methods are **persistence**, **linear regression**, a **lagged-input MLP**, and a **compact LSTM**. Feature ablations assess carrier-current and steam-flow inputs; a snapshot comparison examines the value of history.

## Data Preparation and Evaluation

- Split actual records chronologically, initially **70% training / 15% validation / 15% test**, before constructing windows.
- Keep histories and prediction targets within their assigned partitions; use temporal gaps where needed.
- Fit scaling on training data only and exclude invalid targets, long gaps, and windows crossing shutdowns.
- Select model settings using validation data and retain the final test partition for evaluation.
- Assess independent production periods and distinct low/high-level events, rather than relying on row count alone.

Evaluation measures include:

| Area | Measures |
| --- | --- |
| Level forecasting | MAE and RMSE in level percentage points, including errors by operating band |
| Warnings | Precision, recall, false alarms per recorded operating hour, and warning lead time |
| Model efficiency | Parameter count and CPU inference latency |
| Robustness | Sensitivity to missing readings, delays, and noise |
| Advisory behavior | Bounds, carrier coordination, adjustment cooldown, and suppression on invalid conditions |

## Two Testing Modes

### Recorded Replay

Held-out mill records are streamed in timestamp order to assess forecasts against measured future levels. Recorded future readings are revealed only when their process timestamps are reached. Replay evaluates forecasting and advisory logic; it cannot establish the physical outcome of a newly suggested carrier setting.

### Generated Operating Scenarios

Coherent operating sequences provide supplemental tests for feed changes, accumulation, restricted discharge, and imperfect inputs. Scenario responses are reported separately from measured-data forecast accuracy. Generated scenarios do not replace the primary training or test records.

The laptop demonstration supports normal or accelerated playback without requiring a live factory connection.

## Carrier-Speed Advisory

The **GRU predicts level**. A separate policy produces carrier-setting recommendations; the network does not learn invented optimal-speed labels.

| Forecast or validity condition | Operator-facing advice |
| --- | --- |
| 60–75%, inclusive | Hold current feed setting and continue monitoring |
| Above 75% | High-level warning and bounded feed reduction, following the carrier coordination rule |
| Below 60% | Bounded increase only when the mill is running normally and no restriction or fault is flagged |
| Fault, suspected blockage, stale data, or unverified limits | Suppress numerical suggestions and request operator review |

Carrier settings use one declared command unit: preferably VFD frequency in **Hz**, otherwise verified **RPM**. Recommendations respect minimum/maximum settings, per-update change limits, carrier sequencing, and a transport-delay-aware cooldown. Current-level alarms remain separate from forecast warnings.

Demonstration settings are distinct from engineer-approved plant limits. Operators retain decision authority, and the application does not send commands to factory drives. A high-level forecast is not a blockage diagnosis.

## Dashboard

The dashboard design presents:

- Current sensor level and predicted level at the forecast horizon.
- The 60–75% target band and current/forecast warning status.
- Current and recommended Hz/RPM settings, setting change, and action reason.
- Data-source mode, timestamps, and input freshness.
- Messages requiring inspection or data-quality review.

## Project Poster

[![Light-theme project poster](poster/chute-level-poster-light.png)](poster/chute-level-poster-light.pdf)

[Download poster PDF](poster/chute-level-poster-light.pdf) · [Open system-flow image](images/system-flow.png) · [Read project proposal](docs/Chute_Level_DL_Proposal.pdf)

The visuals are presentation concepts with illustrative screen values, not measured performance results. The workflow image uses built-in image generation; its [generation prompt](images/system-flow-generation-prompt.md) is included for reference.

## Technology Stack

**Python · NumPy · pandas · PyTorch · scikit-learn · Local Python dashboard**

## Author

**Syed Affan Ali**  
Deep Learning coursework — MS Artificial Intelligence  
NED University of Engineering and Technology, Karachi

[Portfolio](https://syedaffan-saffu.github.io/) · [GitHub](https://github.com/syedaffan-saffu)
