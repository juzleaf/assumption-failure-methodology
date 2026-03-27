# AFM Radiate v3.2 — OpenBSE Code Review Report

**Target**: `/tmp/afm-benchmark-round3/openbse-radiate`
**Methodology**: AFM Radiate v3.2 (Assumption-Failure Method with REVERSE/FORWARD/RADIATE)
**Prompt**: Empty — no guidance, no hints

---

## Phase 1: Seed Sweep Summary

Broad codebase sweep across 8 crates (~15,000+ lines). Key areas examined:
- `openbse-core` (simulation.rs, types.rs, graph.rs, ports.rs)
- `openbse-components` (fan, pump, chiller, boiler, heating_coil, cooling_coil, chw_cooling_coil, heat_recovery, humidifier, cooling_tower, heat_exchanger, vav_box, water_heater)
- `openbse-envelope` (heat_balance.rs, infiltration.rs, zone_loads.rs, ctf.rs, airflow_network.rs)
- `openbse-controls` (thermostat.rs, setpoint.rs, state.rs)
- `openbse-cli` (main.rs — 5169 lines)
- `openbse-psychrometrics` (lib.rs)
- `openbse-io` (input.rs)

---

## Phase 2: Interrogation Results

### FINDING 1 — Pump heat-to-fluid formula disagrees with EnergyPlus
**File**: `crates/openbse-components/src/pump.rs`, line 208
**Verdict**: VULNERABLE
**Severity**: Medium

**WHAT**: `self.heat_to_fluid = self.power * self.motor_heat_to_fluid_fraction`

**WHY this is wrong**: In EnergyPlus, pump heat to fluid = ShaftPower + MotorLosses × FracMotorLossToFluid. That is:
```
heat_to_fluid = shaft_power + (total_power - shaft_power) × fraction
```
where `shaft_power = motor_efficiency × total_power`. The code instead computes `total_power × fraction`.

When `fraction = 1.0` (the default), both formulas give identical results (total_power). But when `fraction < 1.0`, the results diverge significantly:
- **E+ formula**: shaft power always goes to fluid (it's mechanical work converted to friction heat in the water), only motor casing losses are split by the fraction.
- **This code**: reduces ALL heat to fluid proportionally, including shaft power that physically must become fluid heat.

Example: motor_efficiency=0.9, total_power=1000W, fraction=0.0:
- E+: heat_to_fluid = 900W (shaft power) + 0 = 900W
- OpenBSE: heat_to_fluid = 1000 × 0.0 = 0W

**RADIATE**: The fan component (`fan.rs` lines 149-153) implements this correctly:
```rust
let shaft_power = self.motor_efficiency * self.power;
self.heat_to_air = shaft_power + (self.power - shaft_power) * self.motor_in_airstream_fraction;
```
The pump should use the same decomposition. This is not just a theoretical difference — pump heat is a significant contributor to plant loop temperature rise, especially in chilled water systems where every 0.1°C matters. With `fraction=0.0` (motor outside airstream/fluid), the pump would add zero heat to the chilled water loop, underestimating supply water temperature and overpredicting cooling capacity.

---

### FINDING 2 — Core library `run_with_envelope()` never calls `update_bdf_history()`
**File**: `crates/openbse-core/src/simulation.rs`, lines 177-259
**Verdict**: VULNERABLE
**Severity**: High

**WHAT**: The `run_with_envelope()` method in the core crate solves the envelope once per timestep and simulates HVAC once, but never calls `envelope.update_bdf_history()` afterward.

**WHY this matters**: The `EnvelopeSolver` trait (`ports.rs` line 318) requires `update_bdf_history()` to be called exactly once per timestep after HVAC convergence. This shifts the backward-difference formula (BDF) temperature history that the next timestep's zone heat balance predictor depends on. Without it, the zone temperature extrapolation uses stale history, causing the predictor to output incorrect values. Over many timesteps this compounds into incorrect zone temperatures.

The heat_balance.rs implementation explicitly warns about this in comments at lines 4536-4554: "The actual CTF history shift is done in update_bdf_history(), called once after HVAC convergence. Shifting here would [corrupt the extrapolation]."

**RADIATE**: The CLI (`main.rs` lines 1693, 1906, 2472) and the sizing module (`sizing.rs` line 332) correctly call `update_bdf_history()`. So in practice, users running via the CLI are fine. But anyone using the core library's `run_with_envelope()` directly (e.g., embedding the engine, writing tests, or building alternative frontends) will get silently wrong results. The function also lacks any convergence iteration (HVAC runs once, no zone temp feedback), unlike the CLI's 10-iteration predictor-corrector loop.

---

### FINDING 3 — Core library `run_with_envelope()` uses static ControlSignals with no convergence loop
**File**: `crates/openbse-core/src/simulation.rs`, lines 177-259
**Verdict**: VULNERABLE
**Severity**: High

**WHAT**: `run_with_envelope()` takes a single `&ControlSignals` reference used identically for every timestep. HVAC supply conditions (line 232-238) are copied from this static signal into `ZoneHvacConditions` every timestep without updating based on envelope results. No HVAC-envelope iteration occurs.

**WHY this matters**: Building energy simulation requires HVAC-envelope coupling: zone loads depend on supply conditions, and supply conditions depend on zone temperatures. A single-pass approach uses stale zone temperatures from the previous timestep, causing systematic error. The CLI solves this with a predictor-corrector loop (lines 1930-2464, up to 10 iterations, 0.05°C tolerance). The core library function has no such loop.

Combined with Finding 2, `run_with_envelope()` is a non-functional API for coupled simulation: no convergence iteration AND no BDF history update. The `SimulationConfig` fields `max_air_loop_iterations`, `max_plant_loop_iterations`, and `convergence_tolerance` exist but are never read by any simulation method, making them misleading configuration that has no effect.

**RADIATE**: The `run_with_controls()` method (line 116) has the same issue — static signals, no iteration — but this is expected since it doesn't involve envelope coupling.

---

### FINDING 4 — Inconsistent specific heat of water across codebase
**Files**: Multiple
**Verdict**: VULNERABLE
**Severity**: Medium

**WHAT**: Two different values of cp_water coexist:
- `openbse_psychrometrics::CP_WATER = 4180.0` J/(kg·K) — the central constant
- `chiller.rs` line 190: hardcoded `let cp_water = 4186.0`
- `main.rs` line 2026: hardcoded `let cp_water = 4186.0`
- `water_heater.rs` line 24: local `const CP_WATER: f64 = 4186.0`

**WHY this matters**: The 6 J/(kg·K) difference (0.14%) seems trivial in isolation, but it creates energy imbalance at system boundaries. When the chiller computes outlet temperature using cp=4186, but the coil receiving that water computes heat transfer using cp=4180, the energy "handed off" across the plant loop boundary doesn't balance. Over an annual simulation with millions of timesteps, this accumulates.

More importantly, this pattern reveals a maintenance hazard: components use local constants instead of the single source of truth. If someone corrects CP_WATER in psychrometrics, the chiller and water_heater would silently retain the old value.

**RADIATE**: The heating_coil tests use `4180.0` (matching the psychrometric constant), while chiller production code uses `4186.0`. The boiler, cooling_tower, and heat_exchanger all correctly import `CP_WATER` from psychrometrics. Only chiller, water_heater, and main.rs deviate.

---

### FINDING 5 — Chiller outlet temperature magic clamp
**File**: `crates/openbse-components/src/chiller.rs`, line 244
**Verdict**: VULNERABLE
**Severity**: Low-Medium

**WHAT**: `let t_outlet = (inlet.state.temp - delta_t).max(self.chw_setpoint - 2.0);`

**WHY this matters**: The `-2.0` magic number allows the chiller to cool water 2°C below setpoint, which is physically unrealistic for a controlled chiller. In EnergyPlus, the chiller outlet is clamped at the setpoint (not below). This hardcoded margin means the chiller can overcool, and the `2.0` value has no physical basis — it's neither documented nor configurable.

**RADIATE**: If the setpoint is 6.7°C (typical), the clamp allows cooling to 4.7°C. At very low loads with high PLR, the actual delta_t could be small enough that this clamp never activates. But at full load with undersized flow, `inlet_temp - delta_t` could go well below setpoint, and the 2.0°C margin prevents the safety clamp from engaging properly. The correct behavior is to clamp at the setpoint and adjust PLR/cycling accordingly (which the chiller already does via min_plr cycling).

---

### FINDING 6 — CHW cooling coil ignores dehumidification despite having rated_shr field
**File**: `crates/openbse-components/src/chw_cooling_coil.rs`, line 180
**Verdict**: VULNERABLE
**Severity**: Medium-High

**WHAT**: Comment says "Simplified: no dehumidification modeled (humidity ratio passes through)" — the outlet air humidity ratio equals the inlet, regardless of how cold the coil surface is. Yet the struct has a `rated_shr` field, suggesting dehumidification was intended.

**WHY this matters**: In humid climates, a CHW cooling coil removes 30-50% of its total cooling load as latent (moisture removal). Passing humidity through means:
1. Zone humidity ratio rises unchecked — no condensation on the coil
2. Total cooling load is underpredicted (only sensible counted)
3. Chiller is undersized if sized from this coil's demand
4. Zone comfort metrics (RH%) are wrong

The DX cooling coil (`cooling_coil.rs`) correctly models dehumidification via the apparatus dew point method with SHR splitting. The CHW coil should use a similar approach.

**RADIATE**: The humidifier (`humidifier.rs`) correctly models moisture addition. The DX coil correctly models moisture removal. Only the CHW coil has this gap. Any building model using CHW coils in humid climates will systematically underpredict cooling energy and overpredict indoor humidity.

---

### FINDING 7 — Heating coil UA computed with counterflow LMTD, effectiveness uses cross-flow formula
**File**: `crates/openbse-components/src/heating_coil.rs`, lines 63-84 (UA calc) and 318-328 (effectiveness calc)
**Verdict**: VULNERABLE
**Severity**: Low-Medium

**WHAT**: `compute_ua()` (line 63) uses counterflow LMTD to derive UA_design from rated conditions. But `simulate_hot_water_ua()` (line 319) applies a cross-flow effectiveness formula: `ε = 1 - exp((NTU^0.78 / C_ratio) * (exp(-C_ratio * NTU^0.22) - 1))`.

**WHY this matters**: Counterflow LMTD yields a smaller UA than cross-flow LMTD for the same heat duty (counterflow is more efficient). Using this smaller UA in a cross-flow effectiveness formula means the coil underperforms compared to the rated condition — at rated flows, the effectiveness won't recover the rated capacity. The test at line 731+ acknowledges this with a 15% tolerance.

EnergyPlus handles this correctly by using a consistent single framework (either NTU throughout, or iterating to match rated conditions). The mixed approach here means the coil is systematically derated by ~10-15%.

**RADIATE**: This is self-consistent within the codebase (the coil just performs slightly below rated), but it's inconsistent with the claimed E+ compatibility. For a building simulation aiming to match E+ results, this introduces a persistent bias in heating coil output.

---

### FINDING 8 — Autosize sentinel collision range
**File**: `crates/openbse-core/src/types.rs`, lines 57-62
**Verdict**: VULNERABLE
**Severity**: Low

**WHAT**: `AUTOSIZE = -99999.0` with `is_autosize()` checking `(val - AUTOSIZE).abs() < 1.0`, meaning any value in [-100000, -99998] triggers autosize.

**WHY this matters**: While the probability of a legitimate engineering value falling in this range is low, the tolerance of ±1.0 is unusually wide for a sentinel check. Standard practice uses exact equality or machine-epsilon tolerance. The 1.0 range means floating-point arithmetic that produces -99998.5 (e.g., from a sum or interpolation involving the sentinel) would silently trigger autosizing.

**RADIATE**: The heating coil already has a guard against unresolved autosize (line 382-387): `if self.nominal_capacity < 0.0` returns zero output. But other components don't have this guard — they'd pass the autosize sentinel through to physics calculations, producing nonsensical results (e.g., a fan with design_flow_rate = -99999 m³/s).

---

### FINDING 9 — Graph cycle-breaking: comment says "most incoming edges" but code picks minimum
**File**: `crates/openbse-core/src/graph.rs`, lines 129-130 vs 152-155
**Verdict**: VULNERABLE
**Severity**: Low

**WHAT**: Comment on line 129: "breaks cycles at the component with the most incoming edges." Code on line 155: `.min_by_key(|(_, &d)| d)` — selects the node with the *fewest* remaining in-degree.

**WHY this matters**: The comment and code contradict. The code picks the node with fewest remaining in-edges to break the cycle, which is actually a reasonable heuristic (it's the "closest to being ready" node). But the comment says the opposite. For anyone maintaining or reasoning about the algorithm, the wrong comment could lead to incorrect modifications.

The practical impact is limited because plant loops typically have simple cycles (pump → chiller → coil → pump), and either heuristic would produce a valid (if different) simulation order.

---

### FINDING 10 — Hardcoded non-leap-year calendar in 7+ locations
**Files**: `simulation.rs` (lines 122, 184), `types.rs` (line 33), and 5+ other locations
**Verdict**: VULNERABLE
**Severity**: Low

**WHAT**: `days_in_months: [u32; 12] = [31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31]` is hardcoded in at least 7 places with February always 28 days.

**WHY this matters**: For a building energy simulation, ignoring leap years means:
- Feb 29 weather data is skipped or misaligned
- Annual energy totals are off by 1/365 (~0.27%)
- Day-of-year calculations are wrong for March-December in leap years

EnergyPlus handles leap years. The repeated hardcoding (instead of a shared constant or function) also means fixing this requires changes in 7+ locations.

---

### FINDING 11 — Heat recovery assumes C_min is always on supply side
**File**: `crates/openbse-components/src/heat_recovery.rs`, lines 142-150
**Verdict**: HOLDS (with caveat)
**Severity**: Low

**WHAT**: Comment says "assumes balanced or supply-limited flow" and uses `c_supply` directly as the capacity rate for effectiveness calculation, without comparing to exhaust-side capacity.

**WHY**: The code has no access to exhaust mass flow rate — the exhaust side is modeled with fixed `exhaust_air_temp` and `exhaust_air_w` (set externally), not as a flowing stream with its own mass flow. So C_min is assumed to be the supply side because the exhaust side isn't modeled as a heat exchanger stream.

This is documented and intentional — it's a simplification. In practice, balanced flow (supply ≈ exhaust) is the design intent for most ERVs, making C_supply = C_min a reasonable assumption. The finding HOLDS as a deliberate simplification, but would become VULNERABLE if unbalanced flow scenarios are added later.

---

### FINDING 12 — Humidifier ignores sensible heat from steam injection
**File**: `crates/openbse-components/src/humidifier.rs`, line 144
**Verdict**: VULNERABLE
**Severity**: Low

**WHAT**: `let t_out = inlet.state.t_db;` — outlet temperature equals inlet temperature. Comment says "approximately isothermal for dry air (the steam energy goes into moisture, not sensible heating)."

**WHY this is slightly wrong**: Steam enters at ~100°C. The sensible heat added to air = mass_steam × cp_steam × (100 - T_air). For typical conditions (T_air=20°C, steam=0.001 kg/s, air=5 kg/s): ΔT ≈ 0.001 × 1860 × 80 / (5 × 1005) ≈ 0.03°C. This is negligible for most applications, making the isothermal assumption reasonable.

However, for high humidification loads (e.g., healthcare, cleanrooms) with large steam injection rates relative to airflow, the temperature rise could reach 1-2°C, which is significant for downstream coil control.

EnergyPlus does account for this sensible heat. The code correctly notes it's a simplification. This is a minor accuracy gap, not a bug.

---

### FINDING 13 — SetpointController::plant_setpoint() always emits load=0.0
**File**: `crates/openbse-controls/src/setpoint.rs`, lines 72-75
**Verdict**: HOLDS (dead code path)
**Severity**: Low

**WHAT**: The `plant_setpoint()` constructor creates a controller that always emits `SetPlantLoad { load: 0.0 }` regardless of the `setpoint` parameter value passed to it.

**WHY it's not currently a bug**: `SetPlantLoad` is never consumed anywhere in main.rs (grep confirms zero matches). Plant loop load determination happens entirely within main.rs by summing coil thermal outputs (lines 2026-2100+). The `SetpointController::plant_setpoint()` is effectively dead code — it exists in the controls framework but is bypassed by the CLI's direct plant loop simulation.

The comment at line 74 ("actual load determined by demand") confirms this is intentional: the controller was designed as a placeholder. But if someone wires it into a new simulation driver expecting it to work like `air_setpoint()`, they'd get zero plant load regardless of conditions.

---

## Board Summary

### Hot Leads (VULNERABLE — significant impact)
| # | Finding | Severity | Impact |
|---|---------|----------|--------|
| 1 | Pump heat-to-fluid formula wrong when fraction < 1.0 | Medium | Incorrect plant loop temperatures, chiller oversizing |
| 2 | Core `run_with_envelope()` missing BDF history update | High | Wrong zone temps for library users |
| 3 | Core `run_with_envelope()` no convergence iteration | High | No HVAC-envelope coupling for library users |
| 4 | Inconsistent cp_water (4180 vs 4186) | Medium | Energy imbalance at plant loop boundaries |
| 6 | CHW coil ignores dehumidification | Medium-High | Wrong humidity, undersized cooling in humid climates |

### Warm Leads (VULNERABLE — limited/conditional impact)
| # | Finding | Severity | Impact |
|---|---------|----------|--------|
| 5 | Chiller outlet magic clamp -2.0°C | Low-Medium | Allows overcooling past setpoint |
| 7 | Heating coil LMTD/crossflow mismatch | Low-Medium | Systematic 10-15% derating |
| 8 | Autosize sentinel ±1.0 collision range | Low | Unlikely but silent failure mode |
| 9 | Graph comment/code mismatch | Low | Maintenance confusion |
| 10 | No leap year support | Low | 0.27% annual error, weather misalignment |
| 12 | Humidifier isothermal assumption | Low | Negligible for typical loads |

### Cold Leads (HOLDS / Not VULNERABLE)
| # | Finding | Status | Reason |
|---|---------|--------|--------|
| 11 | Heat recovery C_min assumption | HOLDS | Documented, balanced flow is design intent |
| 13 | SetPlantLoad always zero | HOLDS | Dead code path, never consumed |

---

## Key Patterns Observed

1. **CLI vs Library gap**: The CLI (`main.rs`) contains sophisticated logic (predictor-corrector iteration, BDF history management, plant loop orchestration) that is absent from the core library's public API. The core library's `run_with_envelope()` is essentially non-functional for correct coupled simulation. Anyone building on the library without studying main.rs would get wrong results.

2. **Inconsistent E+ fidelity**: Some components faithfully reproduce EnergyPlus physics (fan heat, DX coil SHR, chiller cycling), while others cut corners (CHW coil no dehumidification, pump heat formula, heating coil LMTD/crossflow mismatch). The inconsistency makes it difficult to know which components can be trusted for E+ validation.

3. **Hardcoded constants instead of shared references**: cp_water, days_in_months, max_error=5.0 in thermostat — these appear as local constants or literals instead of importing from a central location, creating maintenance and consistency risks.

---

*Generated by AFM Radiate v3.2 — empty prompt, no guidance*
