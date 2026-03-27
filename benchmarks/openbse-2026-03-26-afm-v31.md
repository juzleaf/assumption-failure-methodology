# AFM v3.1 Report: OpenBSE (Open Building Simulation Engine)

## Seed Map

**Total seeds identified: 19**
**VULNERABLE: 7 | HOLDS: 9 | UNCLEAR: 3**

---

## Phase 1: Seed List

1. `simulation.rs:122` — Hardcoded `days_in_months` with Feb=28, no leap year support
2. `simulation.rs:346` — `day_of_year()` will panic on month=0 (underflow: `(month - 1) as usize`)
3. `simulation.rs:135-136` — Weather array indexed by `hour_idx as usize` with only `.min()` on upper bound, no bounds check on lower
4. `simulation.rs:232-238` — `run_with_envelope` passes static `ControlSignals` to HVAC but builds HVAC conditions from only `zone_supply_temps`/`zone_air_flows`, ignoring all other signal fields
5. `graph.rs:130-181` — Cycle-breaking fallback uses "fewest remaining in-degree" heuristic, comment says "most incoming edges" (line 129)
6. `types.rs:57-62` — AUTOSIZE sentinel (-99999.0) checked with tolerance `< 1.0`, meaning any value in range [-100000, -99998] is treated as autosize
7. `cooling_coil.rs:361-421` — Auto-SHR bypass factor initialized lazily on first call; if rated_capacity is 0 at first call, `bf_initialized` stays false forever and BF defaults to 0.0
8. `cooling_coil.rs:426-435` — DX coil power uses `available_cap * plr_power / available_cop` but `plr_power` is derived from `q_total / available_cap` which can differ from the PLR used for outlet temp calculation in auto-SHR mode
9. `fan.rs:150-153` — Fan heat-to-air formula: `shaft_power = motor_efficiency * power` — this is inverted; shaft power should be the useful work, so `shaft_power = total_power * (total_eff / motor_eff)` or equivalently motor losses = `power * (1 - motor_eff)`
10. `heating_coil.rs:240-250` — Hot water coil simple model: when `water_inlet` is `None`, falls back to `nominal_capacity` as available water capacity — silently assumes infinite hot water supply
11. `boiler.rs:168-220` — LeavingSetpointModulated mode: when `dt_set <= 0` (inlet >= setpoint), boiler returns `*inlet` without updating `fuel_used` to 0 — but `fuel_used` is set to 0 at line 192, so this is fine. However, the inlet mass_flow is 0.0 (passed as dummy) yet the outlet uses `m_actual` which could be 0 — meaning the plant loop sees zero flow.
12. `chiller.rs:242-244` — Chiller outlet temp: `(inlet.state.temp - delta_t).max(self.chw_setpoint - 2.0)` — the 2.0°C below setpoint is a magic number hardcoded with no user control
13. `heat_balance.rs:1549` — Yet another independent `days_in_months` array — duplicated in at least 8 locations across the codebase
14. `heat_balance.rs:4427-4440` — Diagnostic accumulator commit logic uses `sim_time_s` comparison for time advancement detection — floating-point equality check on accumulated time
15. `graph.rs:71-84` — Component name collision: `add_air_component` inserts into `name_to_node` HashMap without checking for duplicates; a second component with the same name silently overwrites the first
16. `schedule.rs:441-447` — `day_of_week` function hardcodes non-leap-year assumption
17. `convection.rs:117-120` — `DEFAULT_WEATHER_WIND_MOD_COEFF` comment says `(270/10)^0.14 = 1.5863` but the actual formula gives `(270/10)^0.14 = 1.586` while E+ uses `(370/10)^0.22 = 1.586` for suburbs — the derivation in the comment is wrong (uses country parameters) but the constant value happens to match
18. `simulation.rs:268-269` — `simulate_timestep` clones `simulation_order` into a new Vec every timestep — unnecessary allocation in the hot path
19. `heat_balance.rs:1916` — Outdoor air humidity ratio hardcoded to 0.008 kg/kg for air density calculation instead of using actual weather data humidity

---

## Phase 2: Assume + Verify (The Loop)

### SEED 1: Hardcoded non-leap-year calendar
**File:** `simulation.rs:122`, `types.rs:33`, `schedule.rs:444`, `heat_balance.rs:1549`, `ground_temp.rs:329`, `main.rs:759`

**"Why did you write it this way?"** — Building energy simulations conventionally use TMY (Typical Meteorological Year) data which is exactly 8760 hours (365 days). EnergyPlus itself assumes non-leap-year. The developer followed E+ convention.

**"What does it actually do?"** — Every `days_in_months` array hardcodes Feb=28. The calendar is duplicated in at least 8 separate locations across 6 files.

**"What does this promise?"** — That the simulation calendar is always 365 days and that all 8 copies stay synchronized.

**REVERSE**: The assumption that leap years never matter holds for TMY data. But the code also supports multi-year runs and arbitrary date ranges. If someone provides a leap-year EPW (actual-year data), hour 1417+ (March 1 onward) would be misaligned by 24 hours. The weather file would have 8784 entries but the simulation iterates only 8760 hours, silently dropping the last 24 hours of December.

**FORWARD**: The duplication is the bigger fragility. If one copy is changed (e.g., to support leap years), the other 7 must change too. This is a maintenance time bomb — `grep` confirms 8 independent definitions.

**Verdict: VULNERABLE**

**WHY**: The developer assumed "TMY is the only input format" because E+ historically enforced this. The false confidence comes from matching E+ behavior. The meta-pattern is **"reference implementation limitations inherited as design assumptions"** — the code copies E+'s constraints even where it doesn't need to.

**Radiation**: This meta-pattern also applies to the `hour` field being 1-indexed (E+ convention), which creates off-by-one risk at every boundary.

---

### SEED 2: Panic on month=0 input
**File:** `simulation.rs:346`, `types.rs:35`

**"Why did you write it this way?"** — Month is documented as 1-12, so `month - 1` is always valid.

**"What does it actually do?"** — `(month - 1) as usize` on a u32 where month=0 causes underflow to `u32::MAX`, then the slice index panics.

**"What does this promise?"** — That callers always provide valid month values.

**REVERSE**: All callers are internal (`month_day_from_hour` returns 1-based months, design day months are user-provided via YAML). YAML parsing uses `u32` which accepts 0. If a user writes `month: 0` in a design day definition, it propagates to `day_of_year()` and panics.

**FORWARD**: Checked YAML parsing — `DesignDayInput` deserializes `month` as `u32` with no validation. The validation module (`validate_model`) was not fully read but the path from YAML→u32→day_of_year is unguarded.

**Verdict: VULNERABLE**

**WHY**: The developer assumed input validation happens upstream. The meta-pattern is **"internal function trusts its caller without defensive checks"**. In a system where YAML input flows through deserialization → struct → computation, every boundary should validate.

**Radiation**: Same pattern applies to `day` (day=0 or day=32 would produce wrong results without panic). Grepped: `start_day` and `end_day` in `SimulationSettings` are also bare `u32` with no range validation.

---

### SEED 5: Graph cycle-breaking heuristic mismatch
**File:** `graph.rs:129-181`

**"Why did you write it this way?"** — Plant loops naturally have cycles. The fallback BFS uses a heuristic to break them.

**"What does it actually do?"** — Line 129 comment says "pick unvisited node with the most incoming edges" but line 155 code says `.min_by_key(|(_, &d)| d)` — it picks the node with the **fewest** remaining in-degree.

**"What does this promise?"** — A reasonable simulation order when cycles exist.

**REVERSE**: The comment and code disagree. The code picks the node with fewest remaining in-degree, which is actually a reasonable heuristic (closer to being ready to simulate). The comment is simply wrong.

**FORWARD**: The behavior is acceptable — picking the node with fewest unresolved dependencies is sensible for breaking cycles. The bug is documentation-only, not behavioral.

**Verdict: HOLDS** (comment is wrong but behavior is reasonable)

Challenge: Could the heuristic produce pathological orderings? For a plant loop A→B→C→A, the node picked depends on which edges have already been processed. Since `in_degree` is HashMap-based (unordered), the specific node chosen is non-deterministic across runs (HashMap iteration order). This means simulation results for models with plant loop cycles could vary between runs.

**Counter-evidence search**: Rust's `HashMap` uses randomized hashing, but `min_by_key` will deterministically pick one minimum if there are ties — it picks the first one encountered in iteration order, which is non-deterministic. However, for typical HVAC topologies, the cycle-breaking point is usually obvious (one node has much lower in-degree), so ties are rare.

**Verdict remains: HOLDS** — but with a noted fragility around non-deterministic ordering for complex multi-cycle topologies.

---

### SEED 6: AUTOSIZE sentinel tolerance too wide
**File:** `types.rs:57-62`

**"Why did you write it this way?"** — Sentinel value -99999.0 with tolerance `< 1.0` to handle floating-point imprecision.

**"What does it actually do?"** — `is_autosize(val)` returns true for any value in [-100000.0, -99998.0].

**"What does this promise?"** — That only the literal autosize sentinel triggers autosizing.

**REVERSE**: A user-provided capacity of -99999.5 (absurd but legal) would be treated as autosize. More realistically, if two autosize values are subtracted or manipulated arithmetically during sizing calculations, intermediate values near -99999 could accidentally trigger.

**FORWARD**: The tolerance of 1.0 is enormous for a sentinel check. Standard practice is epsilon (1e-6) or exact bit comparison. However, since -99999 is so far from any physically meaningful HVAC value, false positives from user input are essentially impossible.

**Verdict: HOLDS**

Challenge: Could arithmetic on autosize values produce false positives? Searched for arithmetic on autosize fields — `AutosizeValue::to_f64()` is the conversion point, and the code guards autosize checks before computation. The wide tolerance is ugly but practically harmless.

---

### SEED 7: DX coil bypass factor lazy initialization failure
**File:** `cooling_coil.rs:361-366`

**"Why did you write it this way?"** — Bypass factor depends on rated_capacity, which may not be set at construction time (autosizing sets it later).

**"What does it actually do?"** — On first `simulate_air` call, if `rated_capacity > 0.0` and `!bf_initialized`, calls `initialize_bypass_factor()`. If rated_capacity is still 0 (or negative/autosize sentinel), BF stays at 0.0 and `bf_initialized` stays false.

**"What does this promise?"** — That BF will be computed before it's needed.

**REVERSE**: If autosizing runs but fails to set rated_capacity (e.g., zero cooling load on design day), the coil enters simulation with `rated_capacity = 0` or `rated_capacity = AUTOSIZE (-99999)`. The guard `rated_capacity > 0.0` prevents initialization. Every subsequent call also fails the guard. The coil runs with `bypass_factor = 0.0`, meaning the apparatus dew point model treats ALL air as passing over the coil surface (BF=0 = no bypass), which produces incorrect SHR calculations.

**FORWARD**: With `bypass_factor = 0.0`:
- `h_out_full = self.adp_h + 0 * (h_in - self.adp_h) = self.adp_h` (which is also 0.0 since never initialized)
- `w_out_full = self.adp_w + 0 * (w_in - self.adp_w) = 0.0`
- This would produce nonsensical outlet conditions

However, the `autocalculate_shr` flag defaults to `false`. The auto-SHR path is opt-in. If the user doesn't enable it, this path is never reached.

**Verdict: VULNERABLE** (when `autocalculate_shr = true` and autosizing fails to set capacity)

**WHY**: The developer assumed autosizing always succeeds before simulation begins. The meta-pattern is **"lazy initialization assumes preconditions will be met by the time the code runs"**. No fallback or error is provided when the precondition fails.

**Radiation**: Same lazy-init pattern exists for `eir_normalization` in `with_curves()` — if `eir_at_rated < 0.01`, normalization stays at 1.0, which silently produces wrong COP at non-rated conditions.

---

### SEED 9: Fan heat-to-air formula
**File:** `fan.rs:150-153`

**"Why did you write it this way?"** — Comments reference E+ Fans.cc physics.

**"What does it actually do?"** —
```
shaft_power = motor_efficiency * power
heat_to_air = shaft_power + (power - shaft_power) * motor_in_airstream_fraction
```

**"What does this promise?"** — Matches EnergyPlus fan heat calculation.

**REVERSE**: Looking at E+ Engineering Reference: "ShaftPower = MotorEff * FanPower" — this IS the E+ formula. In E+ terminology, `FanPower` is the total electrical input power, and `ShaftPower = MotorEff * FanPower` represents the useful mechanical power delivered to the impeller. Motor losses = `FanPower - ShaftPower = FanPower * (1 - MotorEff)`. Heat to air = shaft power (converted to heat via friction) + fraction of motor losses in airstream.

My initial seed was wrong — this formula correctly matches E+. The naming is confusing (shaft_power is really the mechanical power that becomes heat through air friction), but the physics is correct per E+ convention.

**Verdict: HOLDS**

Challenge: Is the E+ formula itself physically correct? In E+ convention, ALL shaft power eventually becomes heat in the airstream (friction → thermal energy). Motor losses are split: fraction `motor_in_airstream_fraction` goes to air, rest goes to surroundings. The formula is: `heat_to_air = MotorEff*P + (1-MotorEff)*P*frac = P*(MotorEff + (1-MotorEff)*frac)`. This is physically sound.

---

### SEED 8: DX coil power calculation inconsistency in auto-SHR mode
**File:** `cooling_coil.rs:393-435`

**"Why did you write it this way?"** — Power should be based on total cooling (sensible + latent).

**"What does it actually do?"** — In auto-SHR mode, `q_total` is computed as `q_total_full * plr` where `plr` is based on sensible demand vs sensible capacity. The power calculation then uses `plr_power = q_total / available_cap`. But `available_cap` is the rated capacity modified by temperature curves, while `q_total_full` is computed from the bypass factor model. These can diverge.

**"What does this promise?"** — Accurate power consumption reflecting actual cooling delivered.

**REVERSE**: `q_total_full = inlet.mass_flow * (h_in - h_out_full)` comes from the bypass factor model, while `available_cap` comes from `rated_capacity * ft_mod * ff_mod`. At rated conditions these should agree, but at off-design conditions they can differ. If `q_total_full > available_cap`, then `q_total = q_total_full * plr` could exceed `available_cap`, making `plr_power > 1.0`, which is clamped to 1.0. This means the coil delivers more total cooling than rated but only pays power for rated capacity — undercounting power.

**FORWARD**: The clamp at line 428 (`clamp(0.0, 1.0)`) masks the inconsistency. The coil could deliver e.g. 110% of available_cap in total cooling (because latent adds to sensible) but only consume power for 100%.

**Verdict: VULNERABLE**

**WHY**: Two independent capacity calculations (curve-based and BF-based) are assumed to agree. The meta-pattern is **"parallel computation paths assumed to be consistent without cross-validation"**.

---

### SEED 10: Hot water coil silent infinite-capacity fallback
**File:** `heating_coil.rs:240-250`

**"Why did you write it this way?"** — Comment explains: "common in current simulation architecture where plant loops run independently."

**"What does it actually do?"** — When `water_inlet` is `None`, `water_capacity` falls back to `nominal_capacity`, meaning the coil acts as if the plant loop always delivers enough hot water.

**"What does this promise?"** — Backward compatibility during the transition to coupled plant-air simulation.

**REVERSE**: The developer explicitly acknowledges this is a simplification. The code is intentionally designed this way for the current architecture where plant loops are not yet fully coupled. The `None` case comment is clear.

**Verdict: HOLDS** (intentional design decision, well-documented)

Challenge: Could this silently produce wrong results if a user thinks plant coupling is active? The code logs nothing when falling back. A user who defines a hot water coil with a plant_loop reference might expect the water-side to limit capacity. If the coupling is not yet wired up, they get infinite hot water with no warning.

The defense is the explicit comment, but there's no runtime warning. This is a documentation issue, not a code defect.

---

### SEED 12: Chiller magic number for minimum outlet temp
**File:** `chiller.rs:244`

**"Why did you write it this way?"** — Prevents the chiller from cooling water unreasonably far below setpoint.

**"What does it actually do?"** — `t_outlet = (inlet.state.temp - delta_t).max(self.chw_setpoint - 2.0)` — hardcoded 2.0°C below setpoint as absolute minimum.

**"What does this promise?"** — Reasonable outlet temperatures.

**REVERSE**: The 2.0°C margin is arbitrary. E+ doesn't have this specific limit — it uses a low-temp cutout field. For low-flow conditions where `delta_t` is large, this clamp silently discards energy (the chiller delivers less cooling than calculated, but power consumption was already computed based on the unclamped PLR).

**FORWARD**: When the clamp activates: `t_outlet` is raised, meaning less cooling is actually delivered. But `self.actual_capacity` was already set (line 233) based on the unclamped calculation. Energy balance is broken: the chiller reports delivering X watts of cooling but the water only receives Y < X watts.

**Verdict: VULNERABLE**

**WHY**: The developer added a safety clamp without propagating the adjustment back to capacity and power calculations. The meta-pattern is **"output clamping without feedback to input calculations"** — the clamp fixes the symptom (unreasonable temperature) but creates an energy balance violation.

**Radiation**: Same pattern in `boiler.rs:177-179` where `max_outlet_temp` clamp correctly recalculates `q`, showing the developer knows the pattern but didn't apply it consistently to the chiller.

---

### SEED 13: days_in_months duplication (8+ copies)
**File:** Multiple files across 6 crates

**"Why did you write it this way?"** — Each module needs calendar math independently; avoiding cross-crate dependencies.

**Verdict: VULNERABLE** (maintenance/consistency risk)

Grouped with Seed 1. The meta-pattern is the same: **reference implementation limitations inherited as design assumptions**. The duplication means any calendar fix must touch 8 files.

---

### SEED 14: Floating-point equality for time advancement
**File:** `heat_balance.rs:4427`

**"Why did you write it this way?"** — Need to detect when the physical timestep advances (vs. HVAC iterations at the same timestep).

**"What does it actually do?"** — `ctx.timestep.sim_time_s != zone.diag_last_sim_time` — compares f64 values for exact equality.

**"What does this promise?"** — Reliable detection of timestep boundaries.

**REVERSE**: `sim_time_s` is computed as `start_hour * 3600.0 + n * dt`. For typical timesteps (dt = 3600, 1800, 900, 600, etc.), the values are exact multiples of powers of 2 and small integers, so floating-point representation is exact. The comparison works correctly.

**FORWARD**: The only risk is if `sim_time_s` computation involved accumulated floating-point error (e.g., repeated `+= dt`). Checking the code: `sim_time = start_hour as f64 * 3600.0` then `sim_time += dt` in a loop. After ~8760 iterations, accumulated error is ~1e-12, far smaller than `dt` (minimum 60.0). The comparison will still work.

**Verdict: HOLDS**

Challenge: Could non-standard timestep values (e.g., 7 timesteps per hour = dt=514.285...) cause issues? The code only accepts specific values: 1, 2, 4, 6, 10, 12, 15, 20, 30, 60 per hour. All produce exact or near-exact dt values. The comparison is safe.

---

### SEED 15: Component name collision in graph
**File:** `graph.rs:71-84`

**"Why did you write it this way?"** — HashMap insert is the simplest approach.

**"What does it actually do?"** — `self.name_to_node.insert(name, idx)` silently overwrites if a duplicate name exists. The old node remains in the graph but becomes unreachable by name.

**"What does this promise?"** — Unique component names.

**REVERSE**: If two components share a name (user error in YAML), the graph has both nodes but only the second is findable by name. Control signals addressed by name would only reach the second component. The first becomes an orphan node that still participates in simulation order but receives no control signals.

**FORWARD**: Searched for upstream validation — `validate_model` in openbse-io likely catches this, but the graph layer itself has no defense.

**Verdict: HOLDS** (if validated upstream) / **UNCLEAR** (didn't fully verify validation coverage)

**Verdict: UNCLEAR** — would need to read the full validation module to confirm.

---

### SEED 17: Wind modification coefficient comment error
**File:** `convection.rs:117-120`

**"Why did you write it this way?"** — Documenting the derivation of the constant.

**"What does it actually do?"** — Comment says `(270/10)^0.14 = 1.5863` (country terrain parameters). But E+ actually computes this as `(δ_met/z_met)^α_met` where met station is assumed to be country terrain (α=0.14, δ=270m, z=10m). So the comment is actually correct for the met station transformation.

**REVERSE**: Wait — re-reading the code more carefully. The constant `1.5863` is defined as "Converts from met station to free-stream wind." The met station IS country terrain (open/airport). The formula `(270/10)^0.14 = 1.586` is the correct derivation. But `27^0.14` = `e^(0.14 * ln(27))` = `e^(0.14 * 3.296)` = `e^0.4614` = 1.586. So the comment and value are correct.

**Verdict: HOLDS**

---

### SEED 18: Unnecessary Vec allocation in hot path
**File:** `simulation.rs:268`

**"Why did you write it this way?"** — Need owned copy because `graph.simulation_order()` returns a slice reference, but `graph.component_mut()` needs `&mut self`.

**"What does it actually do?"** — `let order: Vec<_> = graph.simulation_order().to_vec()` — clones the order vector every timestep.

**"What does this promise?"** — Correct borrowing (can't hold immutable ref while mutating graph).

**REVERSE**: This is a Rust borrow-checker workaround. The order is a `Vec<NodeIndex>` where `NodeIndex` is a lightweight `u32`-sized type. For a typical HVAC system with 5-20 components, this copies 40-160 bytes per timestep. Over 8760 hours × 4 timesteps/hour = 35,040 allocations of ~100 bytes.

**Verdict: HOLDS** (correct workaround, negligible performance impact for typical systems)

---

### SEED 19: Hardcoded outdoor humidity ratio for density
**File:** `heat_balance.rs:1916`

**"Why did you write it this way?"** — Quick approximation for outdoor air density used in infiltration calculations.

**"What does it actually do?"** — `let rho_outdoor = psych::rho_air_fn_pb_tdb_w(p_b, t_outdoor, 0.008)` — uses a fixed w=0.008 kg/kg regardless of actual weather conditions.

**"What does this promise?"** — Accurate outdoor air density for infiltration mass flow.

**REVERSE**: At w=0.008, moist air density at 20°C, 101325 Pa is ~1.194 kg/m³. At w=0.020 (hot humid day), density would be ~1.178 kg/m³. The error is ~1.3%. At w=0.001 (cold dry day), density would be ~1.200 kg/m³, error ~0.5%.

For infiltration, this 0.5-1.3% error is small compared to the uncertainty in infiltration coefficients themselves (which are typically ±20-50%).

**FORWARD**: The actual weather humidity IS available — `weather.dew_point` and `weather.rel_humidity` are right there in the `WeatherHour` struct passed to `solve_timestep`. This is a trivial fix that would eliminate a systematic bias.

**Verdict: VULNERABLE** (small magnitude but unnecessary — the correct data is available one function argument away)

**WHY**: The developer took a shortcut during initial implementation and never revisited. The meta-pattern is **"placeholder approximation that survived into production"**. The humidity ratio 0.008 is a "typical indoor" value, not even representative of outdoor conditions in all climates.

---

### SEED 3: Weather array bounds
**File:** `simulation.rs:135-136`

**"Why?"** — Upper bound is protected by `.min(weather_hours.len() as u32)`. Lower bound (`start_hour`) is computed from user-provided start_month/start_day.

**"Mechanism"** — If start_month=1, start_day=1, then start_hour=0, which is valid. The array is 0-indexed matching hourly data. No underflow is possible since `day_of_year` returns 0-based day count.

**Verdict: HOLDS**

---

### SEED 4: `run_with_envelope` static signals
**File:** `simulation.rs:232-238`

**"Why?"** — This is the core's simple envelope coupling path. The caller (main.rs) uses the more sophisticated per-timestep control loop instead.

**"Mechanism"** — `run_with_envelope` is the original coupling prototype. The real simulation in `main.rs` builds `ControlSignals` dynamically each timestep. `run_with_envelope` is only used for simple/test cases.

**Verdict: HOLDS** (intentional simple path; the real code path is in main.rs)

---

### SEED 11: Boiler LeavingSetpointModulated zero flow
**File:** `boiler.rs:184-220`

**"Why?"** — In SP-modulated mode, the boiler controls its own flow rate rather than using inlet flow.

**"Mechanism"** — The inlet's mass_flow is ignored (the dummy 0.0 in tests). The boiler computes `m_required` from load and delta-T, clamped to design flow. The outlet correctly has `outlet_mass_flow = m_actual`. When `dt_set <= 0`, the boiler correctly returns early with zero load.

**Verdict: HOLDS** (the early return at line 190-196 correctly zeros everything)

---

### SEED 16: Schedule day_of_week leap year
**File:** `schedule.rs:441`

Grouped with Seed 1. Same non-leap-year assumption.

**Verdict: VULNERABLE** (grouped with Seed 1)

---

## Output

### Seed Map Summary
| # | Location | Verdict | Category |
|---|----------|---------|----------|
| 1 | simulation.rs:122 + 7 locations | VULNERABLE | Calendar/Leap year |
| 2 | simulation.rs:346, types.rs:35 | VULNERABLE | Input validation |
| 3 | simulation.rs:135 | HOLDS | Bounds check |
| 4 | simulation.rs:232 | HOLDS | Architecture |
| 5 | graph.rs:129-155 | HOLDS | Documentation bug |
| 6 | types.rs:57-62 | HOLDS | Sentinel design |
| 7 | cooling_coil.rs:361-366 | VULNERABLE | Lazy init failure |
| 8 | cooling_coil.rs:393-435 | VULNERABLE | Energy balance |
| 9 | fan.rs:150-153 | HOLDS | Physics (correct) |
| 10 | heating_coil.rs:240-250 | HOLDS | Intentional |
| 11 | boiler.rs:184-220 | HOLDS | Correct |
| 12 | chiller.rs:242-244 | VULNERABLE | Energy balance |
| 13 | heat_balance.rs + others | VULNERABLE | Duplication |
| 14 | heat_balance.rs:4427 | HOLDS | Float comparison |
| 15 | graph.rs:71-84 | UNCLEAR | Name collision |
| 16 | schedule.rs:441 | VULNERABLE | Calendar (grouped w/1) |
| 17 | convection.rs:117 | HOLDS | Comment (correct) |
| 18 | simulation.rs:268 | HOLDS | Performance |
| 19 | heat_balance.rs:1916 | VULNERABLE | Hardcoded constant |

### Root Causes (Grouped by Shared Failed Assumption)

**RC1: Reference Implementation Limitations Inherited as Design Assumptions**
- Seeds: 1, 13, 16
- Meta-pattern: The codebase copies EnergyPlus constraints (non-leap-year, 365-day assumption) without evaluating whether those constraints are necessary in the new architecture. The duplication across 8+ files amplifies the risk.
- Impact: Silent misalignment for leap-year weather data; maintenance burden for any calendar change.

**RC2: Lazy Initialization Assumes Preconditions Will Be Met**
- Seeds: 7
- Meta-pattern: Bypass factor initialization defers to first use but has no fallback when the precondition (rated_capacity > 0) is not met. The code silently proceeds with zero-initialized state.
- Impact: Incorrect SHR calculations when autosizing fails to produce a valid capacity.

**RC3: Output Clamping Without Feedback to Input Calculations**
- Seeds: 8, 12
- Meta-pattern: Temperature or capacity clamps are applied to outputs without propagating the adjustment back to energy/power calculations. This creates energy balance violations where reported consumption doesn't match reported output.
- Impact: Chiller reports more cooling than water actually receives. DX coil power doesn't fully account for total cooling delivered.

**RC4: Internal Functions Trust Their Callers**
- Seeds: 2
- Meta-pattern: Functions that perform arithmetic on user-provided values (month, day) don't validate ranges. Defense relies entirely on upstream validation which may not cover all paths.
- Impact: Panic on invalid input (month=0).

**RC5: Placeholder Approximations Surviving to Production**
- Seeds: 19
- Meta-pattern: Hardcoded "typical" values used during development that were never replaced with actual data, even when the correct data is readily available.
- Impact: Small systematic bias in outdoor air density (~1%) for infiltration calculations.

### Assumption Radiation Map

```
RC1 (Inherited E+ limitations)
  ├── Calendar: 8 copies of days_in_months across 6 crates
  ├── Hour indexing: 1-based hours (E+ convention) → off-by-one risk
  └── Could also affect: solar position calculations, schedule lookups

RC2 (Lazy init failure)
  ├── DX coil bypass factor → incorrect SHR
  ├── EIR normalization → incorrect COP
  └── Could also affect: any future component with deferred initialization

RC3 (Clamp without feedback)
  ├── Chiller outlet temp clamp → energy balance violation
  ├── DX coil parallel capacity paths → power undercounting
  └── Boiler max_outlet_temp clamp → CORRECTLY handled (counter-example)

RC4 (Trust caller)
  ├── month=0 → panic
  ├── day=0 or day>31 → wrong results
  └── timesteps_per_hour=0 → division by zero
```

### Impact Chains

1. **Leap year misalignment**: Leap-year EPW → 8784 hours → simulation reads only 8760 → last 24 hours of December dropped → annual energy totals wrong → validation against E+ fails for actual-year runs

2. **DX coil auto-SHR failure**: Autosizing produces zero capacity → bypass_factor never initialized → BF=0, ADP=0 → all air contacts coil surface → outlet humidity ratio drops to 0 → MoistAirState with w≈0 → downstream psychrometric calculations produce extreme temperatures

3. **Chiller energy balance**: High load + low flow → large delta_t → outlet clamped → actual cooling < reported cooling → downstream controls see "load met" but zone is actually underserved → control oscillation

### Defense Map (HOLDS + Coverage Gaps)

| Defense | What It Protects | Coverage Gap |
|---------|-----------------|--------------|
| E+ formula matching | Physics correctness | Only protects formulae that were correctly transcribed |
| `clamp()` calls | Numerical stability | Several clamps don't feed back to dependent calculations |
| `.max(0.0)` guards | Negative values | Doesn't protect against zero denominators in some paths |
| Test suite (300+ tests) | Regression | Tests don't cover autosizing-failure paths or leap-year edge cases |
| `validate_model()` | YAML input | Didn't verify coverage of all numeric ranges (month, day, etc.) |

### Unexplored (UNCLEAR, needs-more-time)

1. **Seed 15**: Component name collision — need to verify if `validate_model()` catches duplicate names before they reach the graph builder.
2. **Solar cache fingerprinting**: The FNV-1a hash doesn't include ground reflectance or terrain in the fingerprint — a change to these parameters would use a stale cache. Would need to verify if this matters in practice.
3. **Multizone airflow network convergence**: Newton-Raphson solver in `airflow_network.rs` — didn't read the convergence criteria or failure handling. For complex multi-zone models, non-convergence could silently produce wrong pressures.
