## Context

The `ChargingSession` model already has a `realCharged_kWh` column, and the Grafana charges dashboard includes an "Amount" column computed as `COALESCE(realCharged_kWh, COALESCE((meterEnd_kWh - meterStart_kWh), (endSOC_pct - startSOC_pct) * battery_capacity / 100))`. However, `realCharged_kWh` is never populated automatically by the agent — it requires manual entry via the UI. The SoC-based fallback depends on battery capacity being configured in `vehicle_settings`, which is often unknown or unset.

The charge agent (`charge_agent.py`) already receives periodic `chargePower_kW` updates from the WeConnect API via the `__onChargePowerChange` observer and tracks session start/end times. This makes it possible to integrate power over time to compute actual energy delivered.

## Goals / Non-Goals

**Goals:**
- Compute actual charged energy (kWh) for each charging session automatically
- Store the computed value in `ChargingSession.realCharged_kWh`
- Display the computed energy in the Grafana charges table and charge session detail panel
- Handle session interruptions, conservation charging, and edge cases correctly

**Non-Goals:**
- Not adding new database columns or migrations (reuse existing `realCharged_kWh`)
- Not changing how `meterStart_kWh`/`meterEnd_kWh` work
- Not modifying the ABRP or HomeKit integrations
- Not changing the UI editing workflow for manual overrides

## Decisions

1. **Power integration approach**: Accumulate `chargePower_kW * time_delta` in `__onChargePowerChange` observer. Each time charge power changes, compute `last_power_kW * (current_timestamp - last_timestamp) / 3600` and add to a running accumulator. Store final value in `realCharged_kWh` when session ends.

2. **Accumulator storage**: Add an in-memory `_accumulated_energy_kWh` float on the `ChargeAgent` instance, paired with a `_last_power_timestamp` for the integration step. On session end, write the accumulated value to `chargingSession.realCharged_kWh`.

3. **Overwrite vs append**: Write `realCharged_kWh` only when the session ends (charging stops or plug disconnects). If `realCharged_kWh` already has a manually entered value, skip auto-computation to preserve manual data. Use `COALESCE` in SQL: `COALESCE(realCharged_kWh, computed_energy)`.

4. **Session resumption**: When resuming an interrupted session (within the 24h/300h window), continue accumulating into the existing session rather than starting fresh.

5. **Grafana changes**: No SQL changes needed — the existing `COALESCE` chain already picks up `realCharged_kWh`. The value will now be populated automatically.

## Risks / Trade-offs

- **Missing power samples**: If the car doesn't report charge power at regular intervals, integration may undercount. Mitigation: The WeConnect API fires `__onChargePowerChange` on every update, and the default query interval (180s) provides reasonable granularity.
- **Session not resumed**: If the agent restarts mid-session, the in-memory accumulator is lost and the session's `realCharged_kWh` won't be computed. Mitigation: This is acceptable — the SoC-based fallback in Grafana still provides an estimate.
- **Precision**: Time deltas are based on `carCapturedTimestamp` from the API, not wall clock. Small timing differences are negligible at typical charge rates (11kW+).
