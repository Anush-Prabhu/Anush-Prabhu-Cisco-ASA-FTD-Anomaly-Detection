# Cisco ASA / FTD Log Anomaly Detection

Machine learning workflows for detecting unusual activity in Cisco Adaptive Security Appliance (ASA) and Firepower Threat Defense (FTD) firewall logs. The project combines rule-based labeling, supervised classification, unsupervised scoring, and time-series spike detection.

## Overview

Firewall devices emit high-volume syslog messages identified by signature IDs (for example `ASA-6-302013`, `FTD-6-430003`). This repository provides two Jupyter notebooks:

| Notebook | Purpose |
|----------|---------|
| `Cisco_bd.ipynb` | Builds a synthetic ASA/FTD log dataset, applies rule-based anomaly labels, and trains a Random Forest classifier |
| `Signature_logs.ipynb` | Scores a labeled log CSV with Isolation Forest, evaluates against ground truth, and detects per-signature rate spikes |

Together they cover both **supervised baseline modeling** (when labels exist) and **unsupervised detection** (when only normal traffic is assumed at training time).

## Features

### Synthetic data generation (`Cisco_bd.ipynb`)

- Generates ~9,600 rows across 16 ASA/FTD message signatures
- Fields include timestamp, host, interfaces, zones, five-tuple, user, action, and traffic volume
- Rule-based labels for:
  - Blocked or VPN-related actions (`deny`, `teardown`, `vpn`)
  - Low-severity alerts (severity ≤ 4)
  - Off-hours activity (configurable business hours and weekend handling)
  - High traffic volume (mean + 3σ on `bytes_transferred`)
- Random Forest pipeline with one-hot encoding for categorical fields
- Feature importance chart and confusion matrix

### Log scoring and spike detection (`Signature_logs.ipynb`)

- Loads `Signature_logs.csv` (25,000 rows, 20 columns)
- Feature engineering: message ID encoding, protocol mapping, time-of-day fields
- **Isolation Forest** trained on known-normal rows only (`is_anomaly == False`)
- Exports scored results and top anomalies
- **Rolling z-score spike detection** per `msg_id` over one-minute buckets (120-minute window)

## Requirements

- Python 3.10+
- Jupyter Notebook or JupyterLab

Install dependencies:

```bash
pip install -r requirements.txt
```

## Usage

### 1. Supervised baseline

Open `Cisco_bd.ipynb` and run all cells. The notebook generates data in memory, assigns labels, trains the classifier, and prints a classification report.

Key configuration at the top of the final pipeline cell:

```python
NORMAL_HOURS = (8, 18)
TREAT_WEEKENDS_AS_OFFHOURS = True
ROWS_PER_SIGNATURE = 600
RANDOM_STATE = 42
```

### 2. Unsupervised scoring

Place `Signature_logs.csv` in the project root (same directory as the notebook). Expected columns:

```
timestamp, host, msg_id, severity, message, src_ip, src_port, dst_ip, dst_port,
protocol, ingress_iface, egress_iface, ingress_zone, egress_zone, acl_name, user,
action, conn_id, is_anomaly, anomaly_type
```

Run `Signature_logs.ipynb`. Output files:

| File | Description |
|------|-------------|
| `asa_ftd_scored.csv` | Full dataset with `iforest_score` and `iforest_pred` |
| `asa_ftd_top50_iforest.csv` | 50 lowest-scoring (most anomalous) events |
| `asa_ftd_spikes.csv` | Minute-bucket spikes where z-score > 4.0 |

## Anomaly labeling logic

Rules in `Cisco_bd.ipynb` mark a row as anomalous when any condition matches:

| Rule | Condition |
|------|-----------|
| `blocked_or_vpn` | `action` in `deny`, `teardown`, `vpn` |
| `low_severity_alert` | `severity` ≤ 4 |
| `off_hours` | Outside configured business hours, or weekend if enabled |
| `high_traffic_volume` | `bytes_transferred` > mean + 3σ |

Multiple reasons are stored in `anomaly_type` as a semicolon-separated string.

## Isolation Forest settings

| Parameter | Value |
|-----------|-------|
| `n_estimators` | 200 |
| `max_samples` | 0.8 |
| `contamination` | 0.03 |
| Training data | Rows where `is_anomaly == False` |
| Features | `msg_id_code`, `severity`, `src_port`, `dst_port`, `protocol_num`, `hour`, `minute`, `weekday` |

Lower `iforest_score` values indicate stronger anomaly signals.

## Project structure

```
.
├── Cisco_bd.ipynb          # Synthetic data + supervised Random Forest
├── Signature_logs.ipynb    # Isolation Forest scoring + spike detection
├── requirements.txt
└── README.md
```

## Supported log signatures

```
ASA-6-302013, ASA-6-302014, ASA-6-302015, ASA-6-302016
FTD-6-302015, FTD-6-302014, FTD-6-302016
ASA-4-106023, ASA-6-106100, ASA-6-106102, ASA-6-106103
ASA-6-605005, ASA-5-111008, ASA-5-111010
FTD-6-430003, FTD-6-430002
```

## Notes

- Synthetic data is for development and model prototyping; tune thresholds before production use.
- Isolation Forest recall on rare anomalies depends heavily on `contamination` and the normal-only training set.
- Spike detection uses a 120-minute rolling window with a minimum of 30 observations per series.

## License

MIT
