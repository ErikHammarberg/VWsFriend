## 1. Charge Agent — Power Integration

- [x] 1.1 Add `_accumulated_energy_kWh` (float, default 0.0) and `_last_power_timestamp` (datetime, default None) fields to `ChargeAgent.__init__`
- [x] 1.2 Add energy accumulation in `__onChargingStatusCarCapturedTimestampChange`: compute `delta_hours * current_power_kW` when charging is active and add to `_accumulated_energy_kWh`
- [x] 1.3 First reading after session start sets `_last_power_timestamp` without accumulating (implicit, no prior timestamp)
- [x] 1.4 In `__onChargingStateChange` session end: write `_accumulated_energy_kWh` to `self.chargingSession.realCharged_kWh` only if currently NULL
- [x] 1.5 In `__onChargingStateChange` session resume: restore `_last_power_timestamp` from current carCapturedTimestamp; accumulator carries forward
- [x] 1.6 In `__onPlugConnectionStateChange` disconnect: write accumulated energy and reset accumulator
- [x] 1.7 In all new `ChargingSession` creation paths: reset accumulator to 0.0 and `_last_power_timestamp` to None

## 2. Grafana Dashboard — Verification

- [x] 2.1 Verified: `charges.json` "Amount" column uses `COALESCE(realCharged_kWh, ...)` with `kwatth` unit override
- [x] 2.2 Verified: `charge_session.json` detail panel uses `COALESCE(realCharged_kWh, ...)` for "Amount (real)", map panel references it — all dashboard SQL picks up the field automatically

## 3. Testing

- [x] 3.1 `pytest` — all 2 existing tests pass (coverage: no-data-collected warning is expected for simple fixture tests)
- [x] 3.2 Demo verification: `vwsfriend` package loads and imports correctly; full demo run requires working WeConnect/pillow environment; accumulator logic verified by code review
