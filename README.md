# Cisco ASA / FTD Log Anomaly Detection

Synthetic firewall log generation and anomaly classification for Cisco ASA and FTD syslog data.

## What it does

`Cisco_bd.ipynb` runs end to end:

1. Generates synthetic ASA/FTD log events
2. Labels rows as normal or anomalous using configurable rules
3. Trains a Random Forest classifier and evaluates results

No external dataset is required.

## Setup

```bash
pip install -r requirements.txt
jupyter notebook
```

Open `Cisco_bd.ipynb` and run all cells.

## Generated log fields

Each row includes:

- `timestamp`, `host`, `msg_id`, `severity`, `conn_id`
- `ingress_iface`, `egress_iface`, `ingress_zone`, `egress_zone`
- `src_ip`, `dst_ip`, `src_port`, `dst_port`, `protocol`
- `user`, `action`, `bytes_transferred`, `message`
- Derived: `hour`, `minute`, `weekday`, `msg_id_code`, `protocol_num`

Roughly 9,600 rows are created across 16 message signatures (600 per signature).

## Anomaly rules

A row is marked anomalous if any rule matches:

| Label | Trigger |
|-------|---------|
| `blocked_or_vpn` | `action` is `deny`, `teardown`, or `vpn` |
| `low_severity_alert` | `severity` ≤ 4 |
| `off_hours` | Outside business hours, or on weekends when enabled |
| `high_traffic_volume` | `bytes_transferred` exceeds mean + 3σ |

When multiple rules apply, reasons are joined in `anomaly_type` (semicolon-separated).

## Configuration

Adjust these values in the notebook:

```python
NORMAL_HOURS = (8, 18)              # normal window: 08:00–17:59
TREAT_WEEKENDS_AS_OFFHOURS = True
ROWS_PER_SIGNATURE = 600
RANDOM_STATE = 42
```

## Model

- Algorithm: Random Forest (`n_estimators=250`)
- Preprocessing: one-hot encoding for categorical columns, passthrough for numeric
- Split: 80/20 train/test, stratified on `is_anomaly`

The notebook also plots top feature importances and a confusion matrix.

## Message signatures

```
ASA-6-302013   ASA-6-302014   ASA-6-302015   ASA-6-302016
FTD-6-302015   FTD-6-302014   FTD-6-302016
ASA-4-106023   ASA-6-106100   ASA-6-106102   ASA-6-106103
ASA-6-605005   ASA-5-111008   ASA-5-111010
FTD-6-430003   FTD-6-430002
```

## Files

```
Cisco_bd.ipynb
requirements.txt
README.md
```

## License

MIT
