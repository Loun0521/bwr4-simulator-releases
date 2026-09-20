# BWR/4 Simulator — Verification Report

> 2026-09-13 최신 코드의 결함 수정과 실행 범위는
> [국소 결함·UI 검토 보고서](tests/BUGFIX_UI_REPORT.md)를 우선 참고한다.
> 아래의 과거 script 수와 PASS 기록은 당시 버전의 결과이며 현재 전체 검증의 대체 자료가 아니다.

**Subject:** Verification of simulator constants, setpoints and logic against the
primary vendor-design reference, with external cross-checks where the reference
is silent, ambiguous, or self-inconsistent.

**Model under verification:** `core/bwr4_core.py`, `core/bwr4_bop.py`
(reactor physics and balance of plant), `panel/bwr_panel.py` (control room).

**Status:** Ten systems closed, then re-verified in a second pass (§12). 20 numeric
corrections applied, 3 of those later reverted under the plant-scale rule (§3);
7 dead constants removed. Two systems added since the first pass — thermal limits
(§4.3.3) and samarium-149 (§4.3.4.1). Test suite: 13 scripts, all passing.

> This document records **provenance**, not change history. For what changed and
> when, see `CHANGES.md` §5.13–§5.24. For the physics rationale of the model
> itself, see `physics.md`.

---

## 1. Method

Every constant in the model was placed into one of five classes:

| Class | Meaning | Action |
|---|---|---|
| **MATCH** | Code value reproduces the reference to within rounding | Cite, leave alone |
| **CORRECTED** | Code disagreed with the reference and the reference is right | Change code, re-measure |
| **CODE PREFERRED** | Code disagrees and the **code** is better supported | Keep code, document why |
| **DELIBERATE** | Code intentionally departs (different site/plant assumption) | Document the assumption |
| **NO BASIS** | Reference gives no value; tuned or taken from open literature | Mark as unverified |

A change was accepted only after (a) the citation was read in the original text
rather than in a prior summary, (b) the unit conversion was recomputed, (c) the
affected measurement tables were re-run, and (d) the full test suite passed.

**Verification is not the same as agreement.** Three findings below (§4.3.2, §5,
§6) are cases where the reference was followed too literally at first and had to
be walked back. Those are documented in full because the failure mode — "it is in
the manual, therefore it is right for this model" — recurs.

---

## 2. Sources

### 2.1 Primary

**USNRC, *General Electric BWR/4 Technology Manual* (R-104B)**, USNRC Technical
Training Center. 4 PDF volumes, ~442 pages, scanned/OCR.

Located at `C:\Users\tosuw\Desktop\Offical doc for BWR-4\`. A `pdftotext -layout`
extraction lives in `_text/part1.txt` … `part4.txt` with a hand-built section→line
map in `_text/INDEX.md`.

| Volume | Chapters | Content |
|---|---|---|
| part1 | 1.x, 2.x | Introduction, reactor physics, thermal limits; primary and auxiliary systems |
| part2 | 3.x–6.x | Process instrumentation and control; containment; neutron monitoring |
| part3 | 7.x–10.x | Reactivity control; radwaste; electrical; ECCS |
| part4 | 11.x–13.x | Water and air systems; operations; BWR design differences |

Citations below give the section number and, in parentheses, the line in the
extracted text — e.g. `§1.12.5.4 (part1:1496)`.

**A later revision of the same manual — R-304B.** While verifying thermal limits
(§4.3.3) a newer revision of this manual was located:
**USNRC, *General Electric Systems Technology Manual*, Chapter 1.8 "Thermal
Limits"** (Rev 09/11), [ML11258A297](https://www.nrc.gov/docs/ML1125/ML11258A297.pdf).
It covers thermal limits far more fully than R-104B §1.13 and supplied the GEXL
methodology, the operating-limit MCPR, and the plant-computer monitoring ratios.
Where the two revisions differ this document says so (see §4.3.3 and §8.5).

**Plant technical specifications used as independent confirmation:**
- **Duane Arnold Energy Center (DAEC), a BWR/4 with Mark I** — Technical
  Specifications 3.12 "Core Thermal Limits",
  [ML112230830](https://www.nrc.gov/docs/ML1122/ML112230830.pdf)
- **Quad Cities Unit 1** — Core Operating Limits Report Rev 17,
  [ML23101A065](https://www.nrc.gov/docs/ML2310/ML23101A065.pdf)

**OCR caveats that affected verification.** Decimal points drop out of section
numbers (`10.1.3.2` → `10.13.2`) — this produced one wrong citation in the project
docs (§4.6.6). Two-column layout interleaves columns on one line, so quotes must
be read with ±20 lines of context. Digits are occasionally corrupted (`l0` for
`10`), so every number used was cross-checked against a second occurrence or
against unit consistency.

### 2.2 Secondary

Used only where the primary reference is silent or where its own text flags a
value as approximate. Each use is cited inline at the point of use.

- Keepin delayed-neutron group constants (U-235 thermal), standard 6-group set
- ANS-5.1 decay heat standard, as embodied in the reference column of `tests/validate.py`
- Wigner-Way decay heat correlation
- nuclear-power.com, reactivity coefficients — used to establish that
  −1×10⁻³ Δk/k/%void is a PWR figure
- Published BWR void coefficient design range
- Fukushima Daiichi Unit 1 station battery endurance

---

## 3. Governing methodological finding — the reference is not one plant

**This is the single most important result of the verification, and it invalidated
part of an earlier pass.**

The manual's own preface states:

> "The data provided are **not necessarily specific to any particular nuclear
> power plant**, but can be considered to be **representative of the vendor
> design**."

Different chapters describe plants of different size:

| Location | Scale evidence |
|---|---|
| §9.1 (normal auxiliary power) | "The unit generator supplies **880 megawatts** at full power" |
| Figure 2.0-2 (heat balance) | 10.5×10⁶ lb/hr steam ≈ **2436 MWt** class |
| Chapter 11 (circulating water, service water) | Long Island Sound seawater site |
| **Table 1.5-1 (plant list)** | **3293 MWt · 1065 MWe · 764 bundles · 185 rods** ← the model's basis |

`RATED_MW = 3293.0` and the 764-bundle core come from Table 1.5-1. Chapters 9.1,
11 and Figure 2.0-2 therefore describe a plant **26% smaller** than the model.

### 3.1 The rule

| Quantity type | Scales with plant size? | How to use a manual value |
|---|---|---|
| Times (ADS 105 s, diesel start 10 s) | No | **Use directly** |
| Pressures (150/1150 psig, 0.5 psi ΔP) | No | **Use directly** |
| Setpoint slopes (block 0.66W+42%, scram 0.66W+51%), counts, ratios (30%) | No | **Use directly** |
| **Flows, powers, volumes** | **Yes** | **Scale by 3293/2436 = 1.35** |

### 3.2 How the error was caught

An earlier pass wrote manual gpm and hp values directly into ECCS and BOP
constants. The decisive clue that this was wrong: **LPCI alone matched both the
manual and the pre-existing code at 10,000 gpm**, while HPCI, RCIC and CS were all
low by a consistent factor. RHR pumps are standardized across plant sizes; the
injection systems are not. Three flow constants were reverted (§5).

**Practical consequence:** before writing any manual flow, power or volume into
the code, first ask which chapter it came from.

---

## 4. System verification results

### 4.1 Reactor water level instrumentation

**Reference:** §3.1 (part2:139), setpoints at part2:260 and part2:295;
Figure 3.1-1 for absolute elevations.

The manual gives setpoints on the instrument-zero datum (vessel bottom head = 0 in,
instrument zero = 517 in, normal level = 554 in). The model uses normal level as
its zero, so all values convert by subtracting 37 in and multiplying by 2.54.

| Level | Manual | Converted | Code constant | Verdict |
|---|---|---|---|---|
| Level 8 (high) | +56.5 in | +49.5 cm | `LEVEL_8` = 49.5 | **CORRECTED** (was +43.0, i.e. +54 in) |
| Level 7 (alarm) | +40.5 in | +8.9 cm | `LEVEL_7` = 8.9 | **ADDED** (absent) |
| Level 5 (normal) | +37 in | 0 | `LEVEL_NORMAL` = 0.0 | MATCH |
| Level 4 (alarm) | +33.5 in | −8.9 cm | `LEVEL_4` = −8.9 | **ADDED** (absent) |
| Level 3 (scram) | +12.5 in | −62.2 cm | `SCRAM_LEVEL` = −62.0 | MATCH (0.2 cm) |
| Level 2 | −38 in | −190.5 cm | `LEVEL_2` = −191.0 | MATCH (0.5 cm) |
| Level 1 | −132.5 in | −430.5 cm | `LEVEL_1` = −430.5 | **CORRECTED** (was −422.0, i.e. −129 in) |

**Note the pattern.** L2 and L3 matched to the decimal; L8 and L1 came from a
different source (+54 in, −129 in). Partial agreement with a reference table is
not evidence that the whole table was used. This pattern recurred in every system
below — see §8.1.

#### 4.1.1 Protective actions, not just setpoints

**§3.1.3.1.1** states Level 8 trips **three** loads: the main turbine (to prevent
gross moisture carryover destroying blading), the feedwater pump turbines (to stop
adding inventory), and the HPCI/RCIC turbines (to prevent flooding their steam
lines). The model implemented only the third.

→ Main turbine and feed pump turbine trips added, both **latched** (operator must
reset), matching plant behavior. **CORRECTED.**

This changes accident behavior materially. In ATWS, a level excursion above L8 now
cuts feedwater, reducing inventory, which **concentrates the same boron mass**:
SLC shutdown time fell from 81 to 57 minutes and 90-minute concentration rose from
668 to 1536 ppm (single pump).

**§3.1.3.1.4** states that an L4 low-level alarm coincident with a feed pump trip
runs the recirculation pumps back to a preset speed, lowering thermal power into
the remaining feed pump's capacity.

→ `RECIRC_RUNBACK = 0.45` added, latched with `reset_runback()`. **CORRECTED.**

#### 4.1.2 Setpoint bases (§3.1.3.1) — recorded for future work

- **L3**: keeps steam from leaking below the dryer seal skirt, and leaves coolant
  margin above the top of active fuel.
- **L2**: set low enough that post-scram void collapse does not reach it, high
  enough that RCIC can prevent reaching L1. **ATWS-RPT also initiates here**
  (§3.1.3.1.6).
- **L1**: set sufficiently above the top of active fuel to give ECCS time to act
  (§3.1.3.1.7). This requirement is what exposed the TAF error in §4.2.

---

### 4.2 Core geometry

**Reference:** §2.2.2.1 (fuel description); Figure 3.1-1 (elevations).

The manual gives TAF = 358 in and BAF = 208 in on the same datum as the level
setpoints, i.e. TAF = 358 − 554 = **−196 in = −497.8 cm** relative to normal level.

| Constant | Was | Now | Basis | Verdict |
|---|---|---|---|---|
| `TOP_OF_FUEL` | −368.0 cm (−145 in) | **−497.8 cm** (−196 in) | Figure 3.1-1 | **CORRECTED** |
| `CORE_HEIGHT` | 3.658 m (144 in) | **3.81 m** (150 in) | §2.2.2.1 "active fuel length of **150 inches**"; also Fig 3.1-1: 358−208 = 150 | **CORRECTED** |
| `BOTTOM_OF_FUEL` | −733.8 cm | **−878.8 cm** | derived; matches Fig 3.1-1 BAF (208−554 = −346 in) exactly | **CORRECTED** |

#### 4.2.1 Why the TAF error mattered

With `TOP_OF_FUEL` 130 cm too high, **L1 sat below TAF**. The manual requires the
opposite (§3.1.3.1.7). The observable consequence: low-pressure ECCS and ADS
initiated only after the core was already **17.1% uncovered**. A setpoint whose
entire purpose is to buy time before uncovery was not doing its job.

| | Before | After |
|---|---|---|
| L1 − TAF | −62.5 cm (L1 below TAF) | **+67.3 cm** (L1 above TAF) |
| Core uncovery at L1 | 17.1% | **0%** |

Dynamic check (isolation + break, level falling continuously): at the moment L1 is
crossed, uncovery is 0.0% and ADS arms with 2 CS loops and 4 LPCI loops running.

#### 4.2.2 Derived constants rescaled with core height

| Constant | Was | Now | Reason |
|---|---|---|---|
| `NODE_H` | 30.48 cm | 31.75 cm | ÷12 axial nodes |
| `STEAM_COOLING` | 8.0 W/K | 8.33 W/K | node surface area ∝ height (×1.0416) |
| `ZR_OX_MAX` | 45.0 kW | 46.9 kW | clad surface reaction ∝ height |
| `CORE_COOLANT_VOLUME` | 32.0 m³ | 33.33 m³ | flow area × height |

**Deliberately not rescaled**, with reasons:

- `UO2_CORE_MASS` (138 t) — mass is a plant fact independent of the length error.
  The length was wrong; no fuel was added.
- `CORE_FLOW_AREA` (8.09 m²) — a radial quantity.
- `COUPLE_Z` (0.09) — strictly should weaken to ≈0.083 (diffusion scales as 1/Δz²),
  but it is a tuning constant fixed jointly with `EXCESS_REACTIVITY` and
  `ROD_WORTH` when calibrating criticality at rated conditions. Changing it unpins
  the critical condition. **NO BASIS**; effect audited instead (§4.2.3).

#### 4.2.3 Effect on the rated operating point — negligible

Core height does not enter the steady-state heat balance (node power is
`total/(channels × layers)`, mass flux is on radial flow area). Measured on a
5-day-old core, power, k_eff, pressure, level, void, exit quality and axial
distribution were **identical to the printed decimal**.

> **A measurement-methodology correction is recorded here.** Two indicators —
> axial offset (−7.9%) and radial minimum (0.59) — appeared out of specification
> and were initially attributed to `COUPLE_Z` and to rod sequencing. **Both
> attributions were wrong.** They are `validate.py`'s **end-of-startup** state, in
> which rods are only 90.4% withdrawn and the lower core is suppressed. At the
> operating equilibrium (xenon equilibrium, rods 98.4%) the same quantities are
> **−13.8% and 0.80**, both in specification, and the axial offset was unchanged by
> the core height change (−13.5% both before and after). `validate.py` was
> restructured so that it reports both columns.

---

### 4.3 Core nuclear physics

**Reference:** §1.12 (part1:1437), §1.13 (part1:1741).

This system produced **zero numeric corrections** and one case where the code is
better supported than the manual.

#### 4.3.1 Delayed neutron kinetics — exact match to standard data

| Group | λ (s⁻¹) code | λ Keepin U-235 | β fraction code | β fraction Keepin |
|---|---|---|---|---|
| 1 | 0.0124 | 0.0124 | 0.033 | 0.033 |
| 2 | 0.0305 | 0.0305 | 0.219 | 0.219 |
| 3 | 0.111 | 0.111 | 0.196 | 0.196 |
| 4 | 0.301 | 0.301 | 0.395 | 0.395 |
| 5 | 1.14 | 1.14 | 0.115 | 0.115 |
| 6 | 3.01 | 3.01 | 0.042 | 0.042 |

Fractions sum to exactly 1.0000. `BETA = 0.0064` against the standard 0.0065 —
within the spread of published U-235 values and appropriate for a core with
plutonium buildup. **MATCH.** (Source: Keepin 6-group standard set; the manual does
not tabulate group constants.)

#### 4.3.2 Reactivity coefficients — CODE PREFERRED

**§1.12.5.4 (part1:1496)** gives:

> "**Approximate numerical values** for the three reactivity coefficients are as
> follows: α_V −1×10⁻³ Δk/k/% voids; α_T −1×10⁻⁴ Δk/k/°F moderator;
> α_D −1×10⁻⁵ Δk/k/°F fuel"

| Coefficient | Manual | Converted | Code | Verdict |
|---|---|---|---|---|
| Void α_V | −1×10⁻³ Δk/k/%void | −100 pcm/%void | `VOID_COEF` = **−50** | **CODE PREFERRED** |
| Moderator α_T | −1×10⁻⁴ Δk/k/°F | −18 pcm/°C | `MODERATOR_COEF` = **−30** | CODE PREFERRED (in range) |
| Doppler α_D | −1×10⁻⁵ Δk/k/°F | −1.8 pcm/°C | `DOPPLER_COEF` = **−2.5** | CODE PREFERRED (in range) |

Three independent reasons for not adopting the manual values:

1. **The manual labels them approximate**, and all three are *exactly* powers of
   ten — they are order-of-magnitude teaching figures, not design data.
2. **−100 pcm/%void is the PWR figure.** (Secondary source: nuclear-power.com,
   reactivity coefficients / void coefficient.) A BWR operating at ~40% core
   average void cannot share the void coefficient of a plant with essentially no
   voids; the manual's own next sentence — that the void coefficient "is dominant
   when the reactor is in the power range" — is a statement about a BWR.
3. **The published BWR design range is −0.07 to −0.03 %Δk/k/%void = −70 to −30
   pcm/%void.** The code's −50 sits at the center of that range; the manual's −100
   is outside it.

**Independent check.** Total power coefficient measured by rod insertion at fixed
recirculation flow (so that voids are not varied independently): **−43.0 to −43.2
pcm/%power**, stable across the range. This is the integral quantity that governs
load following and ATWS behavior, and it is consistent with BWR practice.

> **Measurement error recorded.** The first attempt measured the power coefficient
> by varying recirculation flow. That changes void fraction by a path independent
> of power and does not isolate the power coefficient. Redone with rod motion at
> fixed flow.

#### 4.3.3 Thermal limits — MATCH on the value the model can check, GAP otherwise

**§1.13.4 (part1:1906):** "The design LHGR for 8x8 fuel is **13.4 kW/ft**."

| Quantity | Model | Manual | Margin |
|---|---|---|---|
| Core average LHGR | 5.47 kW/ft (measured) | — | — |
| Peaking factor | 2.10 (measured) | — | — |
| **Peak LHGR** | **11.5 kW/ft** | 13.4 kW/ft limit | **14%** |

**MATCH** — the model operates inside the design limit with a credible margin.

> **Comparison error recorded.** The first comparison put the model's *average*
> 5.47 kW/ft against the manual's 13.4 kW/ft *peak design limit* and concluded a
> large margin existed. Average and peak are not comparable; the peaking factor
> must be applied first.

**§1.13 also gives** (part1:1902–1904, 1980):

- LHGR limit ≈ 25 kW/ft unirradiated, falling to **20 kW/ft at 40,000 MWd/MT**
- **MCPR safety limit 1.07** — "more than 99.9% of the fuel rods in the core are
  expected to avoid transition boiling"
- MAPLHGR limits "restrict the amount of stored energy in the fuel thus limiting
  the rate of cladding heatup on a LOCA"

**These three limits did not exist in the model at all** — a grep for
`MCPR|CPR|LHGR|APLHGR` returned nothing. This was the largest gap found in the
entire verification, and **it has since been implemented** (`CHANGES.md` §5.21).
What follows records the sources used and what remains uncertain.

##### Sources for the implemented limits

| Constant | Value | Source | Class |
|---|---|---|---|
| `LHGR_LIMIT` | 13.4 kW/ft | §1.13.4; independently confirmed by **DAEC TS 3.12** — "rod in any 8x8 fuel assembly shall not exceed 13.4 KW/ft" | **MATCH** |
| `MCPR_SAFETY_LIMIT` | 1.07 | §1.13.6; R-304B §1.8.5.2 gives the same | **MATCH** |
| `MCPR_OPERATING_LIMIT` | 1.44 | R-304B Figure 1.8-7; §1.8.5.3 states the range is "1.2 to 1.5" | **MATCH** |
| `MAPLHGR_LOW_FLOW` | 0.95 below 70% core flow | DAEC TS — "When core flow is equal to or less than 70% of rated, the MAPLHGR shall not exceed 95% of the limiting values shown" | **MATCH** |
| `MAPLHGR_LIMIT` | 11.2 kW/ft | see below | **NO BASIS (weak)** |
| `LOCAL_PEAKING` | 1.13 | see below | **NO BASIS** |
| `CPR_K` | 0.638 | bundle-geometry correction, back-calculated | **NO BASIS** |

**`MAPLHGR_LIMIT` is the weakest number in the implementation.** The manual gives
no value — §1.13.5 says only that it comes from LOCA analysis and varies with
exposure. Real values are published only as *graphs* in plant tech specs: DAEC
Figure 3.12-8 (fuel type P8DRB299) has a vertical axis spanning 8–14 kW/ft, but
the curve itself is an unreadable scan. The nearest hard number found is
**11.00 kW/ft** (Quad Cities COLR Table 3.1 — GNF3 fuel, flat to 38.64 GWd/ST).
11.2 was adopted as a typical 8x8 plateau between these. **This value determines
whether the model reads over or under its APLHGR limit** — see §4.3.3.1.

**`LOCAL_PEAKING` bridges a structural gap.** LHGR is by definition a *single
rod* quantity, and the model has no intra-bundle detail (one channel is the
average of 12.7 bundles). The manual's Table 1.8-2 figures — "1.61 at a corner
rod to 1.47 at an interior rod" — are **GEXL test-range bounds, not design
values**, and must not be used here. 1.13 is the midpoint of the usual 8x8 design
range (1.10–1.15).

##### The critical power correlation

GEXL itself is proprietary and absent from both revisions. But R-304B §1.8.5.1
gives both its **form** and the **method**:

> "The General Electric **Critical Quality (Xc) vs. Boiling Length (LB)** (GEXL)
> correlation predicts the onset of transition boiling."

> "By successively increasing bundle power from its initial level, a set of
> curves … can be generated until the bundle power is high enough that its curve
> becomes **tangent** at some point to the correlation curve. The bundle power
> corresponding to curve (2) is the critical power."

The model implements that tangency search directly (bisection on the bundle power
multiplier, all 60 channels solved as arrays), using the public **CISE-4**
correlation — which has the same critical-quality-versus-boiling-length form — for
the correlation curve, with one bundle-geometry constant `CPR_K`.

**The correlation was checked for correct physical trends**, independent of the
calibration:

| Check | Result |
|---|---|
| Pressure 7.03 → 3.0 MPa | critical quality 0.269 → 0.296 (**rises**) ✓ |
| Mass flux 800 → 2000 kg/m²s | critical quality 0.395 → 0.235 (**falls**) ✓ — critical *power* still rises as G^⅔ |
| **Power reduced along the flow control line** | **MCPR 1.46 → 1.45, essentially flat** ✓ |

The last row is the strongest validation available. In a BWR, lowering
recirculation flow lowers power *and* flow together, so MCPR barely moves — a
well-known plant characteristic that the correlation was not tuned to reproduce.
(Cutting flow *alone*, so power lags, drops MCPR 1.46 → 1.35 as it should.)

##### Where CPR must not be calculated — §1.8.5.4

Accident testing produced a peak-quality problem: with the core **38% uncovered**
the model still reported a comfortable-looking MCPR of 2.44. With no water there
is no nucleate boiling to lose, so the number was meaningless. The manual
anticipates exactly this:

> "The use of the GEXL correlation is **not valid** for all critical power
> calculations at pressures below **785 psig** (800 psia) or core flows less than
> **10% of rated** flow. Therefore, the fuel cladding integrity safety limit,
> under these plant conditions, is established by other means. This is done by
> **limiting core thermal power to 25% of rated** when reactor pressure is below
> 785 psig." (R-304B §1.8.5.4)

Implemented as `MCPR_VALID_PRESSURE` (785 psig), `MCPR_VALID_FLOW` (0.10) and
`THERMAL_POWER_LIMIT` (0.25) — **MATCH**. Outside the valid range the model stops
reporting MCPR and shows thermal power / 25% in the MFLCPR position instead. Core
uncovery was added as a third invalidating condition; that one is a model-level
judgement, not a manual condition, but the channel enthalpy balance does not hold
without liquid.

The effect on the same ATWS case:

| | Before | After |
|---|---|---|
| Display | MCPR **2.44** (looks like margin) | **out of range** |
| Substitute indicator | none | **MFLCPR 2.284** — thermal power 56.6% is 2.28× the 25% limit |

A falsely reassuring number became an accurate one.

##### 4.3.3.1 Result — and the axial reflector the limits exposed

**When the limits were first added, MAPRAT read 1.023 — over its limit.** The
diagnosis and fix are recorded here because the limits are what made the
underlying defect visible.

The model had **no reflector at all**: `_diffuse` discarded everything that
crossed a boundary. A real BWR has water plenums above and below the active fuel
that return a large share of it. The consequence was visible in the axial shape:

```
before   0.51 0.93 1.20 1.35 1.42 1.40 1.31 1.18 1.01 0.81 0.57 0.30
after    0.79 1.07 1.23 1.29 1.30 1.27 1.19 1.08 0.94 0.79 0.62 0.43
```

The end nodes were crushed to 0.51 and 0.30. Because the shape is normalised to a
mean of 1, **whatever the ends lose the middle gains** — node peaking reached
2.194, i.e. a total peaking factor of 2.194 × `LOCAL_PEAKING` = **2.479, above the
design 2.409** (= 13.4 / 5.56). APLHGR is measured on a single axial slice, so it
is fully exposed to that distortion, which is why MAPRAT alone crossed 1.0 while
MFLPD (0.966) and MFLCPR (0.989) stayed just under.

**Fix: `REFLECT_Z` = 0.50** — the missing neighbour at an axial boundary is filled
with `own value × REFLECT_Z`. Radial boundaries still leak (see below).
`EXCESS_REACTIVITY` fell 23,500 → **23,281** to offset the +219 pcm the reduced
leakage adds; the rated rod position is unchanged at 98.4%, which is the check
that the pair is balanced.

**Why 0.50.** Swept 0.2–0.8 and anchored on Table 1.8-1's design total peaking
factor by fuel type — BWR/5 8x8 = **2.51**, BWR/6 8x8 = **2.21**:

| Albedo | Axial ends | Axial peak | Hot-channel axial | Node peak | **Total peaking** | MFLPD | MAPRAT |
|---|---|---|---|---|---|---|---|
| 0 (before) | 0.51 / 0.30 | 1.425 | 1.558 | 2.194 | **2.479** ✗ | 0.975 | **1.032** ✗ |
| 0.35 | 0.67 / 0.38 | 1.351 | 1.498 | 2.076 | 2.346 | 0.922 | 0.976 |
| **0.50** | **0.77 / 0.44** | **1.309** | **1.462** | **2.009** | **2.270** | **0.891** | **0.944** |
| 0.65 | 0.90 / 0.49 | 1.268 | 1.408 | 1.949 | 2.202 | 0.868 | 0.919 |
| 0.80 | 1.05 / 0.57 | 1.218 | 1.346 | 1.877 | 2.121 | 0.836 | 0.886 |

Three reasons for 0.50 rather than the smaller 0.35 that would also clear 1.0:

1. **Total peaking 2.27 sits mid-band** between the two 8x8 design values.
2. **It survives the uncertainty in `MAPLHGR_LIMIT`.** That constant is the
   weakest number in the implementation (§4.3.3); at 0.50 the model stays under
   1.0 whether the limit is 11.0 (0.961) or 11.5 (0.919). At 0.35 an 11.0 limit
   gives 0.994 — inside, but with nothing to spare.
3. **The shape stays a BWR shape.** Still bottom-peaked, axial offset −15.1%
   (target −10 to −20%), and the hot channel's axial peaking of **1.424** now sits
   near the **cosine 1.39** that R-304B Table 1.8-2 lists among the GEXL test
   shapes. The previous 1.558 was up against that table's upper bound (inlet/outlet
   peaked, 1.60).

**Radial was deliberately left leaking.** The radial direction already has a
calibrated flattening term — `FUEL_ZONE`, back-calculated from the real power
distribution — while the axial direction had nothing. Radial peaking (1.32) and
minimum (0.81) were already inside spec (≤1.4, ≥0.7). A radial reflector would
also add **+2021 pcm** (the 8×8 grid has 28 perimeter channels out of 60),
requiring the whole criticality balance to be re-derived.

**Result at rated operating equilibrium** (`validate.py` `[7b]`, both columns):

| Quantity | End of startup | Operating equilibrium | Limit |
|---|---|---|---|
| Peak LHGR [kW/ft] | 9.56 | **12.05** | 13.4 |
| Peak APLHGR [kW/ft] | 8.46 | **10.67** | 11.2 |
| MCPR | 1.602 | **1.474** | 1.44 (safety 1.07) |
| MFLPD | 0.714 | **0.899** ✓ | 1.00 |
| MAPRAT | 0.755 | **0.952** ✓ | 1.00 |
| MFLCPR | 0.899 | **0.977** ✓ | 1.00 |

Core-average LHGR 5.56 kW/ft; total peaking factor **2.196** (design 2.409).
All three ratios now sit in the 0.9s — the range plants are commonly said to
operate in. Peak fuel temperature also fell 1145 → 1060 °C.

> **Two traps, recorded.**
> **(1) Swapping physics at power scrams the plant.** Inserting the reflector into
> a running core raised k_eff to 1.000863 on the first step and APRM jumped to
> 112.8% → scram, even with `EXCESS_REACTIVITY` pre-corrected by the statically
> computed +131 pcm. That is a *switch-in transient*, not a property of the new
> equilibrium. The parameter sweep was redone with RPS disabled so the controller
> could settle; real use starts cold, where it never arises.
> **(2) `aged.pkl` had to be regenerated.** The rod positions and power shape
> stored in it were equilibrated under the old diffusion. Same class as the
> `POOL_MASS` trap (§4.7.3): **change the physics, rebuild the pickle.**

##### Still not implemented

1. **Exposure-dependent limits.** §1.13.4 has the 1% strain LHGR falling 25 → 20
   kW/ft by 40,000 MWd/MT, and MAPLHGR is exposure-dependent by definition. Both
   are fixed constants now; the model already tracks `exposure`.
2. **Power- and flow-dependent MCPR operating limit** (R-304B Figures 1.8-9 Kp,
   1.8-10 MCPRf) — currently a fixed 1.44.
3. **Scram-speed-dependent operating limit** (Figure 1.8-8).
4. **Intra-bundle rod power distribution** — lumped into `LOCAL_PEAKING`.

#### 4.3.4 Xenon — half-lives MATCH, transient timing slightly early

**§1.12.7.1 (part1:1639):** xenon "cannot decay as fast (**9.2 hr** t½) as it is
being formed from iodine (**6.7 hr** t½)".

The code uses **9.14 h** and **6.57 h** — the precise nuclear data values, of which
the manual's figures are roundings. **MATCH** (code more precise).

Measured post-scram transient from full-power equilibrium:

| Quantity | Model | Literature | Assessment |
|---|---|---|---|
| Equilibrium worth | −2590 pcm | `XENON_WORTH` −2600 by construction | consistent |
| Peak time after scram | **8.2 h** | ≈10.3 h | **≈20% early** |
| Peak / equilibrium | 1.82× | — | — |
| Additional worth at peak | **−2131 pcm** | up to −2500 pcm | in range |
| Return to equilibrium | ≈25 h | ≈20 h | acceptable |

**Cause identified:** `XE_BURN = 7.5e-5` implies a thermal flux of ≈2.8×10¹³
n/cm²·s. BWR operating flux is typically 3–5×10¹³. A higher burnout rate would
delay the peak (more xenon destroyed while at power, so the iodine backlog takes
longer to dominate). Magnitude is correct; only timing is affected. **NO BASIS** —
the manual gives no flux value. Not changed, because raising `XE_BURN` also shifts
`XE_EQ_FULL` and would require recalibrating `XENON_WORTH`.

#### 4.3.4.1 Samarium-149 — added (was absent)

§1.12.7.2 (part1:1666): "Next to xenon-135, the most important fission product
poison is samarium-149… is a **stable nuclide**, is removed **only by burnout**
and has a probability for neutron absorption **about 100 times smaller** than that
of xenon-135." Chain: Nd-149 (1.7 h) → Pm-149 (53 h) → Sm-149 (stable). Nd decays
30× faster than Pm and is folded into Pm production.

| Constant | Value | Basis | Verdict |
|---|---|---|---|
| `LAMBDA_PM` | 53.08 h half-life | Manual's figure reads **47 h**; 53.08 h is the accepted value | **CODE PREFERRED** — same reasoning as the xenon half-lives (§4.3.4) |
| `YIELD_PM` | 0.0113 | A=149 chain fission yield | **NO BASIS** (manual gives no yields) |
| `SM_BURN` | `XE_BURN`/66 | Cross-section ratio (Xe-135 ≈ 2.65e6 b, Sm-149 ≈ 4.0e4 b). The manual's "about 100 times smaller" is approximate | **CODE PREFERRED** |
| `SM_WORTH` | −570 pcm | **Derived, not chosen** — see below | **MATCH** (literature −500 to −700) |

**`SM_WORTH` is derived from the xenon model, which is itself a check.**
Concentration ratio `SM_EQ_FULL/XE_EQ_FULL` = 14.39, times the cross-section ratio
1/66, gives 0.218; × `XENON_WORTH` (−2600) = **−567 pcm**. That the xenon constants
alone produce a samarium worth landing mid-range in the published band is evidence
the two poison models are mutually consistent.

**Reactivity is entered as a deviation from equilibrium**, unlike xenon:

```python
rho += SM_WORTH * (self.samarium / SM_EQ_FULL - 1.0)
```

Equilibrium samarium is *already inside* `EXCESS_REACTIVITY` — that constant was
back-calculated to match rated criticality, and a real core carries equilibrium
samarium. Entering the absolute value would double-count it and break the critical
condition. With the deviation form, the contribution at rated is **exactly zero**
(verified over 30 simulated days: `Sm/eq` = 1.00000, −0.00 pcm), so no existing
behavior changed.

**Measured post-shutdown behavior:**

| After shutdown | Sm/eq | Samarium | Xenon |
|---|---|---|---|
| 6 h | 1.024 | −14 pcm | **−4667 pcm** |
| 48 h | 1.146 | −83 pcm | −649 pcm |
| **96 h** | 1.224 | **−128 pcm** | **−21 pcm** |
| 720 h | 1.313 | **−179 pcm** | −0 pcm |

The 96-hour row is the reason this was worth adding: xenon has effectively gone
while samarium is at −128 pcm and still rising. Restart margin a few days after
shutdown is set by samarium, and unlike xenon it does not decay away. The final
−179 pcm matches the published post-shutdown buildup range (−100 to −300 pcm).

Implemented in `_samarium(dt)`; see `CHANGES.md` §5.22 and `physics.md` §1.7b.
Remaining limitation: treated as a single core-average point, as xenon is.

#### 4.3.5 Decay heat — consistent with literature; a reference-basis mismatch in the test

Six exponential groups (`DECAY_TAU` 2 s to 200,000 s ≈ 55.6 h) approximating the
Wigner-Way curve. The manual gives no decay heat table.

**The old report compared two different things.** `tests/validate.py` printed the
model against an **infinite-irradiation** ANS-5.1 column while the measurement runs
a **5-day** core. The model's longest group has τ = 55.6 h and cannot saturate in
5 days, so it *had* to read low — the printed "−25% at 24 h" mixed a real model
limitation with a pure test artifact.

**Fixed by separating them.** `[19]` now also prints what the model's *own*
constants give at saturation (`Σ DECAY_FRAC·exp(−t/DECAY_TAU)`), which splits the
gap into its two causes:

| After shutdown | Model (5 d) | Model saturated | ANS-5.1 | Saturated − ANS *(group fit)* | 5 d − saturated *(test basis)* |
|---|---|---|---|---|---|
| 10 s | 3.956% | 3.967% | 4.00% | −1% | −0% |
| 1 m | 2.588% | 2.677% | 2.70% | −1% | −3% |
| 5 m | 1.821% | 1.915% | 2.00% | −4% | −5% |
| 30 m | 1.186% | 1.274% | 1.40% | −9% | −7% |
| **1 h** | **0.965%** | 1.049% | 1.20% | −13% | −8% |
| 2 h | 0.801% | 0.881% | 1.00% | −12% | −9% |
| 8 h | 0.573% | 0.643% | 0.70% | −8% | −11% |
| **24 h** | 0.376% | 0.428% | 0.50% | **−14%** | **−12%** |

So the former single "−25%" is roughly **half model, half test basis**, and neither
half is large. Short-lived groups saturate within 5 days, which is why the two
model columns agree at 10 s and diverge only as the 55.6 h group's share grows.

The 1-hour value also matches the independent literature figure directly —
"about 1% after one hour" against the model's **0.965%**. **MATCH.**

A genuine structural limit remains: with the longest time constant at 55.6 h,
behavior beyond a few days is not well represented. That is now visible in the
table rather than hidden inside a mislabelled error column.

#### 4.3.6 Burnup — MATCH at +1.5%

The manual gives no cycle length. Its only burnup figure is the exposure at which
the LHGR limit drops to 20 kW/ft: **40,000 MWd/MT** (part1:1904).

| Step | Value |
|---|---|
| `UO2_CORE_MASS` | 138,000 kg UO₂ |
| Heavy metal (×238/270) | 121.6 MT |
| `CYCLE_EFPD` × `RATED_MW` | 500 × 3293 = 1,646,500 MWd |
| **Per cycle** | **13,535 MWd/MT** |
| **Three cycles** (standard BWR residence) | **40,606 MWd/MT** |
| vs manual 40,000 MWd/MT | **+1.5%** |

The agreement is strong evidence that `CYCLE_EFPD` and `UO2_CORE_MASS` are mutually
consistent and consistent with the reference, even though neither was taken from
it. **MATCH.**

Cycle reactivity shape (`cycle_reactivity()`): gadolinia burnout gain 3000 pcm
against 3000 pcm total depletion, giving a flat early cycle and a late drop.
Qualitatively matches the manual's description of burnable poison behavior; no
numeric basis in the reference. **NO BASIS.**

---

### 4.4 Reactivity control and protection

**Reference:** §7.1 (part3:85) with Table 7.1-1 (part3:268); §7.2 (part3:508);
§7.3 (part3:863) with Table 7.3-1 (part3:1188); §7.4 (part3:1319).

**One correction**, found on the third pass (§13.1) — the first pass checked this
section against Table 7.1-1 only, which lists *rod blocks*. The scram setpoints are
in a different table (Table 5.4-1, `part2:3366`), and the two differ.

| Item | Manual | Code | Verdict |
|---|---|---|---|
| APRM rod block | ".66W + 42%" (Table 7.1-1; Table 5.4-1 "APRM High") | `APRM_FLOW_SLOPE` 0.66 / `APRM_BLOCK_BIAS` 42.0 | MATCH |
| APRM rod block clamp | "108% Max." (Table 5.4-1); "Flow Converter Hi 108%" (Table 7.1-1) | `APRM_BLOCK_CLAMP` 108.0 | MATCH |
| APRM flow-biased scram | ".66(W) + 51%" — Table 5.4-1 "APRM High-High Thermal Power" | `APRM_SCRAM_BIAS` 42.0 → **51.0** | **CORRECTED** |
| APRM flow-biased scram clamp | "113.5% Max." (Table 5.4-1, same row) | `APRM_SCRAM_CLAMP` 113.5 | MATCH |
| APRM fixed-power scram | "Fixed Power 118% — Rod Block & Scram" (Table 5.4-1) | not implemented | **DELIBERATE** — unreachable below the 113.5% clamp |
| RBM rod block | ".66W + 41% (set high)" (Table 7.1-1) | `RBM_SLOPE` 0.66 / `RBM_BIAS` 41.0 | MATCH |
| RBM inoperative | "Fail to null / More than one rod selected" | `rbm_null()` | MATCH |
| EOC-RPT | "main turbine trip or load rejection, if greater than 30% power" (§7.2.3.2, part3:677) | `RPT_POWER` 0.30 | MATCH |
| ATWS-RPT | "1120 psig or low-low level" (§7.2.3.2) | `ATWS_RPT_PRESSURE` 7.82 MPa abs, `LEVEL_2` | MATCH |

#### 4.4.1 Standby Liquid Control — MATCH throughout

**Reference:** §7.4 (part3:1319).

| Item | Manual | Code | Verdict |
|---|---|---|---|
| Tank volume | "4,850 gallons" (§7.4.2.1) | 18.36 m³ | MATCH |
| Pumps | "two 100% capacity … greater than 39 gpm" (§7.4.2.2) | 2.7 kg/s × 2 | MATCH |
| Solution | "13 weight percent sodium pentaborate" | 0.183 × 0.13 = 0.0238 | MATCH |
| Initiation | "starts a single pump, fires both explosive valves, and isolates RWCU" (§7.4.3.1) | `arm_slc()` | MATCH |
| Shutdown time | "one to two hour … **not a backup for scram**" (§7.4.3.1) | measured 81 min (57 min after L8 fix) | MATCH |
| Injection point | "beneath the core plate into … jet pump diffuser outlets" | `SLC_MIX_TAU` | MATCH |

SLC is notable as the one system verified end to end with **no discrepancy of any
kind** — values, logic, and timing all reproduced.

---

### 4.5 Main steam

**Reference:** §2.5 (part1:4422).

| Item | Manual | Code | Verdict |
|---|---|---|---|
| MSIV actuation | "air to open; air and/or spring to close" (§2.5.2.3, part1:4586) | fail-closed | MATCH |
| Bypass capacity | "Four bypass valves … up to **30%** of rated steam flow" (§2.5.2.5) | `BYPASS_CAPACITY` 0.30 | MATCH |
| Flow restrictor | "limit steam flow to less than **200%** of rated steam flow during a steam line break" (§2.5.2.2) | break-flow cap | MATCH |
| Turbine | "**1800 RPM**, tandem compound" (§2.5.2.8, part1:4599) | `RATED_RPM` 1800.0, 1 HP + 3 LP | MATCH |
| TSV closure scram threshold | "power above the capacity of the bypass valves" (§2.5.3.2, part1:4701) | `TSV_SCRAM_POWER` 0.30 (= bypass capacity) | MATCH |
| EHC pressure setpoint | "normally set at **920 psi**" turbine inlet (§3.2.2.1, part2:569) | core 7.03 MPa abs ≈ 990 psig dome | MATCH (line loss) |
| EHC bypass demand | "the bypass valve demand is established by subtracting the pressure/load LVG output from the pressure control unit output" (§3.2.2.4, part2:615) | `want = demand − turbine_valve` | MATCH |
| MSIV isolation conditions | Table 4.4-1 — high radiation, tunnel high temperature, high flow (mode switch in RUN), low pressure, low-low level, manual | all present | MATCH on structure; **the table gives no numeric setpoints** (§12.5) |

#### 4.5.1 SRV count — manual self-inconsistent, code retained

| Source | Statement | Kind |
|---|---|---|
| §2.5.2.1 | "The Safety/Relief valves **(11)**…" / "any of the **11** safety/relief valves" | body, twice |
| §10.2.2.1 / §10.2.3.1 | "seven of the existing **eleven**" / "**Seven of the eleven**" | body, twice |
| §2.5.4.11 | "**six of the thirteen**" | one-line cross-reference |

Code uses **13 SRVs / 6 ADS**. **CODE RETAINED** — but note the manual's weight of
evidence is 11/7, and the justification is *not* "the main steam chapter says 13"
(it says 11). It is that **Browns Ferry** — identified by R-304B Table 1.8-1 as the
3293 MWt BWR/4 this model is built to — has 13 relief valves with 6 on ADS. The
manual's 11/7 belongs to its smaller reference plant, the same way its ECCS flows
do (§3). The first pass got this right by accident; see §12.2.

---

### 4.6 Emergency Core Cooling Systems

**Reference:** §10.1 HPCI (part3:3439), §10.2 ADS (part3:3760), §10.3 CS
(part3:3964), §10.4 RHR/LPCI (part3:4185); §2.7 RCIC (part1:5238).

#### 4.6.1 Flows — the plant-scale case (see §3)

| Constant | Code | In gpm | Manual value | Verdict |
|---|---|---|---|---|
| `HPCI_FLOW` | 268.0 kg/s | 4250 gpm | 4000 gpm | **DELIBERATE** — manual value is the 2436 MWt plant |
| `RCIC_FLOW` | 38.0 kg/s | 600 gpm | 400 gpm | **DELIBERATE** — same |
| `CS_FLOW` | 400.0 kg/s | 6340 gpm | 4725 gpm | **DELIBERATE** — same |
| `LPCI_FLOW` | 630.0 kg/s | 10,000 gpm | **10,000 gpm** | **MATCH** — RHR pumps are standardized |
| `CS_LOOPS` | 2 | — | "a total of **two** core spray pumps" (§10.3) | **MATCH** |
| `LPCI_LOOPS` | 4 | — | "The **A and C** pumps … the **B and D** pumps" (§10.4.2.2) | **MATCH** |

An earlier pass changed the first three to the manual values (252.4 / 25.2 / 298.1)
and then reverted them once §3 was established. The reverted values read as
4250 · 600 · 6340 · 10,000 gpm — all plausible plant ratings, consistently larger
than the manual's, and LPCI identical in both. That coherence is what confirmed the
scale explanation rather than a conversion mistake.

#### 4.6.2 Times and pressures — scale-independent, adopted directly

| Constant | Was | Now | Manual | Verdict |
|---|---|---|---|---|
| `ADS_DELAY` | 120 s | **105 s** | §10.2.3.1 — stated 5× in body text plus Figure 10.2-1 | **CORRECTED** |
| `HPCI_P_MIN` | 0.70 MPa | **0.79 MPa** | ~100 psig, stop valve closes (§10.0.7) | **CORRECTED** |
| `HPCI_P_MAX` | 7.95 MPa | **8.03 MPa** | 1150 psig (§10.1.2) | **CORRECTED** |
| `RCIC_P_MIN` | 0.70 MPa | **1.14 MPa** | 150 psig (§2.7.1) | **CORRECTED** |

#### 4.6.3 ADS arming logic — CORRECTED

The code comment claimed arming required low level **and** high containment
pressure. In fact arming depended on level alone, and the containment term was dead
code. §10.2 supports neither.

→ Rewritten to require **three** conditions simultaneously — (1) L1 low level,
(2) L3 confirmation, (3) at least one low-pressure ECCS pump running — with the
timer **reset** (previously it decayed) whenever any condition is lost:

```python
ads_hold = (self.ads_armed and self.water_level < LEVEL_1
            and self.water_level < SCRAM_LEVEL and lp_running)
if ads_hold: self.ads_timer += dt
else:        self.ads_timer = 0.0
```

Condition (3) is the substantive one: depressurizing with no low-pressure pump
available merely empties the vessel. The panel now displays
`ADS 불가 — 저압 ECCS 미기동`.

The manual states the same combination: "Automatic actuation of the ADS is
initiated upon completion of a **105 second time delay** concurrent with **low
reactor vessel water level**, and **any low pressure ECCS pump running**"
(§10.2.3.1).

#### 4.6.4 Bus assignment — manual self-inconsistent, code retained

| Source | Statement |
|---|---|
| **Table 9.2-1** (Shutdown Board Load List) | matches code: A→101, B→102, C·D→103 |
| §10.4.2.2 | "A and C → division 1" |

The load list is the authoritative allocation document. **CODE RETAINED.**

#### 4.6.5 `CS_P_MAX` / `LPCI_P_MAX` — NO BASIS, deliberately unchanged

Both are 1.90 MPa and are used as **shutoff head** in
`flow = rated × (1 − p/P_MAX)`. The manual gives instead the **discharge pressure
at rated flow** — CS 274 psig, RHR 136 psig. Since head falls as flow rises, shutoff
head is *higher* than either, and the manual does not give it.

Writing the rated discharge pressure into `P_MAX` would assert "zero flow at the
pressure where the manual says full flow occurs" — the exact opposite of the
reference.

**Not changed**, for two reasons:

1. Doing so requires *estimating* shutoff head (e.g. rated × 1.2). That factor is
   not in the manual, contradicting the verification principle.
2. **It does not change any accident outcome.** Substituting estimates
   (CS 2.39, LPCI 1.25 MPa) cuts low-pressure injection by up to 750 kg/s in the
   1.0–1.5 MPa band, yet:

| Scenario | Setting | Min level | Max uncovery | Level at 3 h | Damage |
|---|---|---|---|---|---|
| 20% break | current | −432 cm | 0.0% | +319 cm | none |
| 20% break | estimated | −430 cm | 0.0% | +314 cm | none |
| 5% break | current | −191 cm | 0.0% | −145 cm | none |
| 5% break | estimated | −191 cm | 0.0% | −145 cm | none |

The vessel spends about **one minute** in the pressure band where the two settings
differ. Direction is recorded for future work: `CS_P_MAX` 1.90 MPa (261 psig) is
*below* the rated discharge pressure (274 psig) and therefore conservative;
`LPCI_P_MAX` 1.90 MPa is *above* RHR's (136 psig) and therefore generous. Resolvable
only with pump performance curves.

#### 4.6.6 Logic verified as correct, and two citation errors

| Item | Manual | Code |
|---|---|---|
| HPCI automatic turbine trips | 5 conditions: exhaust pressure high, overspeed, suction pressure low, isolation signal, **high level** (§10.1.3.4, part3:3625) | all 5 present, incl. L8 trip |
| HPCI restart | "if vessel level subsequently decreases to Level 2 … automatically reinitiate" | seal-in implemented |
| HPCI/RCIC steam supply | RCIC from main steam line A, HPCI from line B (§2.5.4.3–4) | upstream of MSIVs |

**Citation errors corrected in the project docs.** HPCI's automatic turbine trips
were cited as "NRC 10.13", a section that does not exist; the correct citation is
**§10.1.3.4 Automatic Turbine Trips** (the content was right). The error traces to
the OCR collapsing `10.1.3.2` into `10.13.2`. Separately, the EHC bypass demand
equation was cited as "3.2.2.2" (Load Control Unit); it is **§3.2.2.4** (Valve
Control Unit).

---

### 4.7 Primary containment (Mark I)

**Reference:** §4.1 (part2:1087); **Table 4.1-1** *Mark I Containment Typical
Specifications* (part2:1460–1495).

| Constant | Code | Manual | Verdict |
|---|---|---|---|
| `DRYWELL_VOLUME` | 4500 m³ | 159,000 ft³ = 4502 m³ | MATCH (−0.1%) |
| `WETWELL_GAS_VOLUME` | 3400 m³ | 119,000 ft³ = 3370 m³ | MATCH (+0.9%) |
| `VACUUM_BREAKER_DP` | 3.4 kPa | "0.5 psi differential" (§4.1.2.3) = 3.45 kPa | MATCH |
| `DW_TEMP0` | 57 °C | normal atmosphere "less than 150 °F" (65.6 °C) | MATCH (in range) |
| `POOL_VOLUME` | 3400 → **3822.8 m³** | "Water Volume **135,000 ft³**" | **CORRECTED** |
| `CONT_DESIGN_PRESSURE` | 487 → **528.8 kPa** | "Maximum Internal Design Pressure **62 psig**" | **CORRECTED** |
| *(none)* | — | "Maximum External Design Pressure **2 psig**" | **GAP** — see §12.6 |

**Diagnosis of the pool error.** The old `POOL_VOLUME` 3400 m³ was *the same number
as the wetwell gas space* (119,000 ft³ = 3370 m³) — water volume and gas volume had
been conflated. The old design pressure, 56 psig, appears nowhere in the manual.

Pool heat capacity rose 3383 t → **3804 t (+12.4%)**.

#### 4.7.1 A consequence for how the model should be read

The suppression pool temperature at which design pressure is reached moved from
138 °C to about **142 °C**. But the same table gives a **design temperature of
281 °F (138.3 °C)** — in the real plant, **temperature reaches its limit before
pressure does.** The model judges only pressure, so a pool above 138 °C is already
outside design conditions even while indicated pressure remains below the design
value. This is now stated in `CHANGES.md` §5.1.

#### 4.7.2 Measured effect — larger the less cooling is available

| Test | Before | After |
|---|---|---|
| MSIV isolation 40 h, no cooling | 137.5 °C / 342 kPa | **131.3 °C / 282 kPa** |
| MSIV isolation 40 h, 1 RHR loop | 54.7 °C / 122 kPa | 54.5 °C / 123 kPa |
| 20% break 4 h, no operator action, peak pressure | 337.3 kPa | 325.2 kPa |
| SBO, pool reaches 100 °C | 15 h 35 m | **18 h 41 m** |
| SBO, pool at 24 h | 119 °C | **111 °C** |
| ATWS with 2 SLC pumps, pool | 77.8 °C | 73.0 °C |

Where the pool is actively cooled the difference nearly vanishes (heat removal is
set by temperature difference); **only when there is no heat sink does the added
capacity buy time.** That is the physically correct direction. No damage or melt
verdict changed in any scenario.

#### 4.7.3 A verification trap, recorded because it will recur

After changing `POOL_VOLUME`, re-measurement produced results **identical to the
printed decimal**. This was not "no effect" — it was **the change not being
applied**.

`self.pool_mass = POOL_MASS` copies the constant into **instance state** (it must,
because the pool genuinely gains mass from SRV discharge and condensate). All
accident tests load `aged.pkl` produced by `tests/mk_aged.py`, so the old mass was
still live inside the pickle. Proof: a byte-level diff of two runs was identical.

Regenerating the fixture produced correct measurements. All 14 changed constants
were then audited in `__init__`: **`POOL_MASS` is the only one copied into instance
state** — the rest are referenced from module scope every step and are unaffected by
the pickle.

> **Rule adopted:** if a constant changes and the result does not, suspect
> "not applied" before concluding "no effect".

---

### 4.8 Electrical

**Reference:** §9.1 (part3:2419), §9.2 (part3:2636), §9.3 (part3:2843),
§9.4 (part3:3033); Table 9.2-1.

**Zero numeric corrections.** Everything the model represents matched; everything
that did not match was absent rather than wrong.

| Item | Manual | Code | Verdict |
|---|---|---|---|
| Emergency buses | "red-101, blue-102, orange-103 / **no bus ties**" (§9.2.2.1) | `BUS_NAMES`, no cross-ties | MATCH |
| Diesel rating | "continuous rating … **3500 KW**" (§9.2.2.2) | `DG_RATING_KW` 3500 | MATCH |
| Diesel start time | "**less than 10 seconds**" (§9.2.2.2) | `DG_START_TIME` 10 | MATCH |
| Diesel auto-start | 4 signals: drywell high pressure, level 1, 4160 V bus undervoltage, board undervoltage (§9.2.3.2) | all 4 | MATCH |
| Diesel cooling | "water cooled", RBSW supplies engine cooling (§11.2.3.2) | `DG_COOLING_LOOP` | MATCH |
| Load allocation | Table 9.2-1 | LPCI A→101, B→102, C·D→103; CS A→101, B→102 | MATCH |
| Bus peak load | must stay under 3500 kW | worst case 3054–3100 kW | MATCH (margin) |
| UPS endurance | "**minimum of two hours** in the event of a total loss of AC power" (§9.3.1.4) | battery 8 h at standby, ~6 h with RCIC | MATCH (exceeds minimum) |

`DC_ENDURANCE_H = 8.0` has **NO BASIS** in the manual *for the load it
describes*. §9.4.3.3 does give a duration — *"The 125 VDC batteries can meet
**worst case** loads for **two hours** in the event of a blackout"* — but that is
LOCA coincident with loss of offsite power, with heavy ESF inrush in the first
minute. The constant is a **standby-load** figure taken from Fukushima Daiichi
Unit 1. The two are not the same quantity, and the model's heaviest load
(HPCI + RCIC) reaches only 4.2 h because the inrush term is absent — see §13.11.
Corrected 2026-09-08: an earlier version of this paragraph said the manual gives
no battery duration at all.

**Dead constant removed:** `RHRSW_PUMP_KW = 500` was defined but never referenced;
actual load calculations use `RBSW_PUMP_KW` (454). The similar names invited
confusion about which figure was authoritative. (A later mechanical scan found six
more of the same kind — §12.4.)

---

### 4.9 Secondary plant (balance of plant)

**Reference:** §2.5, §2.6 (part1:4879), §11.1 (part4:88); Table 1.5-1;
Figure 2.0-2.

**The model's own rating is sourced.** Table 1.5-1 contains an actual row reading
`BWR/4 · Mark I · 3293 MWT · 1065 MWE · 764 bundles · 185 control rods`.
`RATED_MW = 3293` and the 764-bundle core come from this row.

| Item | Manual | Code | Verdict |
|---|---|---|---|
| Bypass capacity | "up to 30% of rated steam flow" (§2.5.2.5) | `BYPASS_CAPACITY` 0.30 | MATCH |
| Turbine | 1800 RPM, 1 HP + 3 LP (§2.5.2.8) | `RATED_RPM` 1800 | MATCH |
| Condensate pumps | "Two motor driven condensate pumps, 50% each" (§2.6.2.2) | `CONDENSATE_PUMPS` 2 | MATCH (§12.3) |
| Condensate booster pumps | "Two, 50% capacity … motor driven with a discharge pressure of **600 psig**" (§2.6.2.5) | `BOOSTER_PUMPS` 2, `BOOSTER_HEAD` 4238 kPa | **ADDED** (§13.4) |
| Feed pumps | "driven by variable speed steam turbines" (§2.6.2.7) | `FEED_PUMPS` 2, turbine driven | MATCH |
| Circulating water pumps | "four circulating water pumps in operation" (§11.1.3) | `CIRC_PUMPS` 4 | MATCH |
| Feedwater heaters | "one drain cooler and three low pressure heaters" (§2.6.2.6) + "two heaters per string" (§2.6.2.8) | DC + LP×3 + HP×2 | MATCH |
| Final feedwater temperature | 420 °F (Figure 2.0-2) | 215.6 °C, computed by the BOP heat balance | MATCH |

#### 4.9.1 Corrected, then scaled (§3)

| Constant | Was | Manual (2436 MWt class) | Now (×1.35) |
|---|---|---|---|
| `CIRC_EACH` | 10,000 kg/s | "Each pump … **143,400 gpm**" (§11.1.2.1) = 9,047 kg/s | **12,230 kg/s** |
| `AUX_CIRC_EACH` | 2.5 MW | "Each motor is rated for **1,500 hp**" (§11.1.2.1) = 1.12 MW | **1.51 MW** |

Circulating water auxiliary load fell from 10 to 4.5 MW, raising net output to
**1072.5 MWe** (previously ≈1067) — inside the 1065–1093 MWe band of the 3293 MWt
plants in Table 1.5-1, and closer to its center than before.

Note that within the *same chapter*, the RBSW pump (450 hp) had been converted
correctly while the circulating water pump had not — the "partially applied table"
pattern again (§8.1).

#### 4.9.2 Values that look wrong but are not — DELIBERATE, annotated in code

- **`RFP_MIN_STEAM_P` = 1500 kPa.** §2.6.3.1 says "approximately 300 psig, the
  reactor feed pumps can be started" (2170 kPa). That is the pressure at which a
  stopped pump can be **started**; this constant is the lower limit at which an
  **already running** pump keeps running. Different quantities.
- **`CONDENSATE_HEAD` = 2800 kPa.** No manual basis — §2.6.2.2 gives no discharge
  pressure for the condensate pumps. Chosen large enough to push through the
  auxiliary condensers and filter/demineralizers to the booster suction header.
  The booster stage that follows it *is* manual-sourced (`BOOSTER_HEAD`, §13.4).

#### 4.9.3 Cooling tower — DELIBERATE, out of scope for comparison

The manual's plant discharges RBSW to **Long Island Sound** — an open-cycle seawater
site. This simulator assumes an inland site with a closed-cycle cooling tower
(stated in `CHANGES.md` §3). `TOWER_*`, `WET_BULB`, `BASIN_*` and
`RBSW_INTAKE_TEMP` are therefore **not** comparable to the manual by design.

---

### 4.10 Reactor building service water

**Reference:** §11.2 (part4:229).

| Item | Manual | Converted | Code | Verdict |
|---|---|---|---|---|
| Pump flow | "Four service water pumps … **8600 gpm**" (§11.2.2) | 543 kg/s (2436 MWt class) | `RBSW_PUMP_FLOW` **734.0** | **DELIBERATE** — scaled ×1.35 (§3) |
| Pump power | "… **450 Hp**" (§11.2.2) | 336 kW | `RBSW_PUMP_KW` **454.0** | **DELIBERATE** — scaled ×1.35 |
| Loop arrangement | "two loops … one pump per loop (A or C and B or D)" (§11.2.3.1) | — | `RBSW_LOOP` = (0,1,0,1) | MATCH |
| Header isolation | "automatically close during LOCA or loss of emergency bus voltage" (§11.2.1) | — | `rbsw_isolated` | MATCH |

These two constants were previously documented as "exactly matching the
literature." **They did match** — the oversight was that the literature in question
described a smaller plant.

---

## 5. Summary of numeric changes

| # | Constant | Old | New | System | Basis |
|---|---|---|---|---|---|
| 1 | `LEVEL_8` | 43.0 | 49.5 | Level | §3.1.3.1 (+56.5 in) |
| 2 | `LEVEL_7` | — | 8.9 | Level | §3.1.3.1 (+40.5 in) — new |
| 3 | `LEVEL_4` | — | −8.9 | Level | §3.1.3.1 (+33.5 in) — new |
| 4 | `LEVEL_1` | −422.0 | −430.5 | Level | §3.1.3.1 (−132.5 in) |
| 5 | `TOP_OF_FUEL` | −368.0 | −497.8 | Geometry | Figure 3.1-1 |
| 6 | `CORE_HEIGHT` | 3.658 | 3.81 | Geometry | §2.2.2.1 (150 in) |
| 7 | `STEAM_COOLING` | 8.0 | 8.33 | Geometry | derived (∝ height) |
| 8 | `ZR_OX_MAX` | 45.0e3 | 46.9e3 | Geometry | derived (∝ height) |
| 9 | `CORE_COOLANT_VOLUME` | 32.0 | 33.33 | Geometry | derived (∝ height) |
| 10 | `ADS_DELAY` | 120 | 105 | ECCS | §10.2.3.1 |
| 11 | `HPCI_P_MIN` | 0.70e6 | 0.79e6 | ECCS | §10.0.7 (100 psig) |
| 12 | `HPCI_P_MAX` | 7.95e6 | 8.03e6 | ECCS | §10.1.2 (1150 psig) |
| 13 | `RCIC_P_MIN` | 0.70e6 | 1.14e6 | ECCS | §2.7.1 (150 psig) |
| 14 | `POOL_VOLUME` | 3400 | 3822.8 | Containment | Table 4.1-1 (135,000 ft³) |
| 15 | `CONT_DESIGN_PRESSURE` | 487e3 | 528.8e3 | Containment | Table 4.1-1 (62 psig) |
| 16 | `CIRC_EACH` | 10,000 | 12,230 | BOP | §11.1.2.1 × 1.35 |
| 17 | `AUX_CIRC_EACH` | 2.5 | 1.51 | BOP | §11.1.2.1 × 1.35 |
| 18 | `RBSW_PUMP_FLOW` | 543 | 734 | Service water | §11.2.2 × 1.35 |
| 19 | `RBSW_PUMP_KW` | 336 | 454 | Service water | §11.2.2 × 1.35 |
| 20 | `RHRSW_PUMP_KW` | 500 | *removed* | Electrical | dead constant |
| 21 | `RODS_PER_BUNDLE` | 63 | **62** | Core geometry | §2.2.2.1.3 (54 + 8 tie) |
| 22 | `WATER_RODS` | 1 | **2** | Core geometry | §2.2.2.1.3 ("two water rods") |
| 23 | `REFLECT_Z` | — (bare core) | **0.50** | Core physics | Axial water plenums; anchored to Table 1.8-1 design peaking — §4.3.3.1 |
| 24 | `EXCESS_REACTIVITY` | 23,500 | **23,281** | Core physics | −219 pcm to offset the reflector's reduced leakage |

Change 21/22: the code carried `63, 1` labelled "GE-4 bundle". **GE-4 genuinely is
63+1**, but the bundle this manual describes is not GE-4 — §2.2.2.1.3 says
"54 standard fuel rods, eight fueled tie rods and **two** water rods", i.e. 62
heat-producing rods plus 2 water rods (64 total). The constants had been
display-only until thermal limits were added; LHGR divides by this count directly,
so the manual's figure was adopted. Core-average LHGR rises 5.474 → **5.562 kW/ft**.

**Added** (new capability, no prior value — see §4.3.3): `LHGR_LIMIT` 13.4,
`MAPLHGR_LIMIT` 11.2, `MAPLHGR_LOW_FLOW` 0.95, `MCPR_SAFETY_LIMIT` 1.07,
`MCPR_OPERATING_LIMIT` 1.44, `LOCAL_PEAKING` 1.13, `CPR_K` 0.638, `CPR_D_H` 0.0125.
Samarium (§4.3.4.1) added `LAMBDA_PM`, `YIELD_PM`, `SM_BURN`, `SM_WORTH`,
`PM_EQ_FULL`, `SM_EQ_FULL`.

**Removed** as dead (§12.4): `FEEDWATER_TEMP`, `VOID_RELAX_MAX`, `CORE_FUEL_MASS`,
`NODE_FUEL_MASS`, `CP_FUEL`, `RATED_MWT`, `DC_NAME`, `CONDPUMP_DISCHARGE`.

**Reverted after §3 was established** (changed, then changed back):
`HPCI_FLOW` 268 → 252.4 → **268**; `RCIC_FLOW` 38 → 25.2 → **38**;
`CS_FLOW` 400 → 298.1 → **400**.

### 5.1 Logic changes

| Change | Basis |
|---|---|
| L8 also trips main turbine and feed pump turbines (latched) | §3.1.3.1.1 |
| L4 + feed pump trip → recirculation runback (`RECIRC_RUNBACK` 0.45, latched) | §3.1.3.1.4 |
| ADS arming requires 3 simultaneous conditions; timer **resets** on loss | §10.2 |
| Vacuum breaker reverse flow made an independent `if`, not `elif` | model defect |

The vacuum breaker fix is not a manual discrepancy but a code defect found during
verification: the reverse-flow calculation was chained as `elif` behind a test for
drywell floor condensate, so it was **skipped entirely on any step where condensate
existed** — visible only in accident-plus-spray conditions where both apply. The two
calculations are independent.

---

| 25 | `APRM_SCRAM_BIAS` | 42.0 | 51.0 | RPS | Table 5.4-1 "APRM High-High" (§13.1) |

---

## 6. Places where the reference was not followed

| Item | Reference says | Model does | Why |
|---|---|---|---|
| `VOID_COEF` | −100 pcm/%void | −50 | Manual value is labeled approximate and is the PWR figure; −50 is centered in the BWR design range (§4.3.2) |
| `MODERATOR_COEF` | −18 pcm/°C | −30 | Same — order-of-magnitude teaching value |
| `DOPPLER_COEF` | −1.8 pcm/°C | −2.5 | Same |
| Xenon half-lives | 9.2 h / 6.7 h | 9.14 h / 6.57 h | Manual figures are roundings of precise nuclear data |
| Pm-149 half-life | 47 h | 53.08 h | Same — the accepted value |
| SRV count | 11 / 7 (§2.5.2.1, §10.2 — 4 statements) vs 13 / 6 (§2.5.4.11 — 1) | 13 / 6 ADS | The manual's 11 is its smaller reference plant; **Browns Ferry**, the 3293 MWt BWR/4 this model targets (R-304B Table 1.8-1), has 13 with 6 on ADS — see §12.2 |
| ECCS bus allocation | Table 9.2-1 vs §10.4.2.2 | Table 9.2-1 | Manual self-inconsistent; followed the load list |
| Condensate pump count | "three" (§2.6.1) vs "Two … 50% each" (§2.6.2.2) | 2 | Manual self-inconsistent; 2 × 50% = 100% is the coherent statement (§12.3) |
| HPCI/RCIC/CS flows | 4000 / 400 / 4725 gpm | 4250 / 600 / 6340 gpm | Manual values are for the 2436 MWt plant (§3) |
| Circ water, RBSW | 143,400 gpm, 1500 hp, 8600 gpm, 450 hp | ×1.35 | Same |
| Ultimate heat sink | Long Island Sound seawater | Cooling tower | Deliberate inland-site assumption |

---

## 7. Parameters with no basis in the reference

Listed so they are not mistaken for verified values.

| Parameter | Value | Source |
|---|---|---|
| `COUPLE_Z` | 0.09 | Tuning constant, fixed jointly with `EXCESS_REACTIVITY` / `ROD_WORTH` at rated criticality |
| `EXCESS_REACTIVITY` | 23,281 pcm | Back-calculated from rated criticality; lowered 219 pcm when `REFLECT_Z` was added (the two move together) |
| `REFLECT_Z` | 0.50 | Axial reflector albedo — no manual value, but **anchored** to Table 1.8-1's design total peaking factor: 0.50 puts the model at 2.216, inside the 8x8 band (BWR/6 2.21 – BWR/5 2.51). See §4.3.3.1 |
| `ROD_WORTH` | 29,500 pcm | Back-calculated |
| `XE_BURN` | 7.5e-5 | Implies flux 2.8×10¹³; manual gives no flux (§4.3.4) |
| `YIELD_PM` | 0.0113 | A=149 chain yield; manual gives no fission yields |
| `CYCLE_EFPD` | 500 | Not in manual; validated indirectly via burnup (§4.3.6) |
| `GAD_GAIN` / `DEPLETION_TOTAL` | 3000 / 3000 pcm | Shape matches the manual's qualitative description only |
| `DC_ENDURANCE_H` | 8.0 | Fukushima Daiichi Unit 1, **standby** load. The manual's 2 h (§9.4.3.3) is a *worst-case* figure including ESF inrush, which the drain model omits — §13.11 |
| `CS_P_MAX` / `LPCI_P_MAX` | 1.90 MPa | Shutoff head; manual gives only rated discharge pressure (§4.6.5) |
| `TURBINE_EFF` | 0.82 | Lumped expansion-line efficiency incl. moisture separation |
| `TOWER_*`, `WET_BULB`, `BASIN_*` | — | Inland site assumption, out of scope |
| `MAPLHGR_LIMIT` | 11.2 kW/ft | Manual gives no value; real curves are unreadable scans. Bracketed by DAEC Fig 3.12-8 (axis 8–14) and Quad Cities COLR 11.00 (§4.3.3) |
| `LOCAL_PEAKING` | 1.13 | Typical 8x8 design range 1.10–1.15. The manual's 1.47–1.61 are GEXL test bounds, not design values |
| `CPR_K` | 0.638 | Bundle correction on CISE-4, back-calculated for MCPR 1.45 at rated |
| `CPR_D_H` | 0.0125 m | 8x8 bundle hydraulic diameter; not in the manual |
| MSIV isolation setpoints | 1.40 × flow, 825 psig, 30 kPa | Table 4.4-1 gives conditions only, no numbers (§12.5) |
| `HEATUP_LIMIT` | 55 °C/h | ≈ the industry-standard 100 °F/h, but not stated in the manual |
| `SRV_FLOW_EACH`, `SRV_BLOWDOWN`, `CST_VOLUME`, `CST_TEMP`, `ECCS_START_DELAY`, `CONT_ALARM_PRESSURE`, `POOL_TEMP0`, `RHR_SPRAY_FLOW`, `RHR_HX_UA`, `TSV_FAST_RATE` | — | Searched for in the second pass; the manual gives no values (§12.5) |

---

## 8. Cross-cutting observations

### 8.1 Partial adoption of reference tables is the dominant failure mode

In *every* system, some constants reproduced the reference to the decimal while a
few came from elsewhere:

| System | Matched exactly | Did not |
|---|---|---|
| Level setpoints | L2, L3 | L8, L1 |
| ECCS flows | LPCI | HPCI, RCIC, CS |
| Containment | drywell volume, wetwell gas volume, vacuum breaker ΔP | pool water volume, design pressure |
| Chapter 11 | RBSW pump | circulating water pump |

Consequently, **partial agreement is not evidence** that a table was applied. Every
constant in a system must be checked individually even when several already match.

### 8.2 The reference contradicts itself in at least three places

| Item | Statements | Code follows |
|---|---|---|
| SRV / ADS count | 11 / 7 (§2.5.2.1, §10.2) vs 13 / 6 (§2.5.4.11) | 13 / 6 — §4.5.1, §12.2 |
| ECCS bus allocation | Table 9.2-1 vs §10.4.2.2 | the load list — §4.6.4 |
| Condensate pump count | "three" (§2.6.1) vs "Two … 50% each" (§2.6.2.2) | two — §12.3 |

All three were caught before "correcting" working code.

**The tie-breaker matters, and the obvious one is wrong.** The first pass used
"prefer the chapter that owns the system" — which fails on the SRV count, because
the owning chapter (§2.5) is itself the one saying *eleven* (§12.2). Ranked by what
actually worked:

1. **Which plant does this number describe?** SRV 13/6 is Browns Ferry, the plant
   this model is built to; the manual's 11/7 is its smaller reference plant (§3).
2. **Is it a formal list or prose?** A load list beats a narrative cross-reference.
3. **Is the statement internally coherent?** Two pumps at 50% each make 100%;
   three at 50% make 150%.

### 8.3 Comment text is not evidence

Two errors were found where a code comment described logic the code did not
implement (ADS arming) or cited a value the constant did not hold (`HPCI_FLOW`'s
comment read 4250 gpm while the manual read 4000 — which is what first exposed the
plant-scale issue).

### 8.4 Pickled fixtures can silently mask constant changes

See §4.7.3.

### 8.5 The two manual revisions disagree in places

R-304B is not merely a reformatting of R-104B. Two differences matter:

| Item | R-104B §1.13.4 | R-304B §1.8.6 |
|---|---|---|
| Exposure unit for the 20 kW/ft breakpoint | "40,000 **MWd/MT**" (metric ton) | "40,000 **MWd/sT**" (short ton) |
| Depth of coverage | value only | full GEXL methodology, MCPR limit figures, plant computer ratios |

The unit discrepancy is a ~10% difference in the exposure at which the LHGR strain
limit falls. The model uses metric tons consistently (its burnup model is in
MWd/MT), which follows R-104B. Recorded so that a later pass does not "correct" it
in the wrong direction.

**A table can mix fuel designs.** R-304B Table 1.8-1's BWR/4 column gives an
average LHGR of 7.05 kW/ft, which does not match this model's 5.56. It is a **7x7**
figure — 3293e3/(764 × 49 × 12.5) = 7.04 — while the BWR/5 and BWR/6 columns are
8x8 (peak LHGR 13.4). Column headings named the plant, not the fuel.

---

## 9. Gaps — items in the reference that the model does not implement

Ordered by importance within each system. Items marked **[needed]** are judged
necessary rather than merely desirable.

### 9.1 Core physics

1. ~~**Thermal limits: MCPR, LHGR, APLHGR.**~~ **DONE** — implemented, see §4.3.3
   and `CHANGES.md` §5.21. Remaining sub-items: exposure-dependent limit curves,
   power/flow-dependent MCPR operating limit, scram-speed-dependent limit,
   intra-bundle rod power distribution.
2. ~~**Samarium-149.**~~ **DONE** — implemented as `_samarium(dt)`, entered as a
   deviation from equilibrium so rated behavior is unchanged. See §4.3.4.1 and
   `CHANGES.md` §5.22. Remaining: spatial distribution (single point, as xenon).
3. ~~**`validate.py` decay heat reference basis.**~~ **DONE** — `[19]` now prints
   the model's own saturated curve beside the 5-day run, splitting the former
   "−25%" into group-fit error (−14%) and test basis (−12%). See §4.3.5.
4. **Xenon peak timing.** ~2 h early because `XE_BURN` implies a low flux (§4.3.4).
   Correcting it requires recalibrating `XENON_WORTH` jointly.
5. **Decay heat beyond a few days.** Longest group τ = 55.6 h.

### 9.2 Secondary plant

> ~~Condensate booster pumps~~ — **implemented** 2026-09-08, see §13.4.

1. **Condensate filters and demineralizers — [not yet].** §2.6.2.4, full-flow
   treatment through nine units; the manual names "circulating water in leakage"
   among the impurities removed. The model has **no water chemistry state at all**
   — no conductivity, no impurities, no condenser tube leak event. Adding the
   filters alone would be a component that changes nothing observable, i.e. dead
   code of the kind §12.2 removed. Build the tube-leak event and a chemistry state
   first; then this becomes meaningful.
> ~~Three parallel feedwater heater strings~~ — **implemented** 2026-09-08, see §13.6.
4. **Turbine moisture separators and reheaters.** Lumped into `TURBINE_EFF` 0.82;
   the expansion line is not split, so LP turbine inlet quality is unobservable.
5. Circulating water vacuum priming and travelling screens (§11.1.2) — minor.

### 9.3 Electrical

> ~~Offsite power redundancy and fast transfer~~ — **implemented** 2026-09-08,
> see §13.10. (§9.2.3.3's table-of-contents title is "Shutdown Board Loading"
> but the body title is "Loss of Preferred Power (LOPP)" — search the body.)

1. **120 VAC — partly implemented 2026-09-10, see §5.46.** §9.3 divides this
   into four systems, and reading them separately changed the priority. Three
   of the four change nothing observable in this model:

   | §9.3 subsystem | supply | what modelling it would add |
   |---|---|---|
   | Safety related control and instrument (9.3.1.1) | emergency 480 VAC | nothing — `control_power` already reaches the same verdict |
   | Normal control and instrument (9.3.1.2) | normal 480 VAC | nothing — noncritical instruments and monitors only |
   | **Reactor protection (9.3.1.3)** | **MG sets with flywheels** | **de-energize is the scram. Implemented.** |
   | UPS (9.3.1.4) | emergency 480 VAC, 125 VDC backup | nothing — `dc_available` is already equivalent, and the *"minimum of two hours"* is already MATCH in §9.2 |

   The RPS bus is the one that mattered, because §7.3 makes the scram itself a
   de-energize event: *"on loss of air pressure **or electrical power** a
   reactor scram would be initiated. This is the fail safe feature."* The air
   half was implemented in §5.40; the electrical half was not.

   It is the exact mirror of the 24 VDC system above — that one may never trip
   the reactor, this one must. Merging them under "low-voltage instrument
   power" would get both wrong.

   **NO BASIS**: which emergency bus feeds which MG set, and the flywheel
   coastdown. The coastdown decides whether a LOOP scrams on RPS power (9 s)
   or later on condenser vacuum (12 s), so it was checked for sensitivity: the
   switchover sits at the diesel start time, and the manual calls the flywheel
   a defence against *"momentary"* changes. A 10 s diesel start is not
   momentary, and every value consistent with that word (1–5 s) gives the same
   answer. §9.3.1.4 states the UPS rides through the offsite-loss-to-diesel
   interval and says nothing of the kind about the RPS — that asymmetry is the
   evidence.

   Still absent: per-channel trip logic (A1·A2·B1·B2), manual scram logics
   (A3·B3), and MG set voltage or frequency. A bus is alive or dead.
> ~~DC system division structure~~ — **implemented** 2026-09-08, see §13.11.
> The earlier note read "2 batteries + 2 chargers per division", which merged two
> systems across the manual's two-column layout; the two-of-each sentence belongs
> to the **24 VDC** bus. §9.4.1 gives one charger plus one battery per 125 VDC
> distribution bus.

> ~~24 VDC system~~ — **implemented** 2026-09-10, see §5.45 and
> `physics.md` §3.2b. §9.4.1 supplies *"the neutron monitoring system
> (source and intermediate ranges)"* from two distribution buses, each with
> two batteries and two chargers, and both chargers on a bus are fed from an
> **emergency** 480 VAC bus through a step down transformer. That last point
> is the one worth having read rather than guessed: the system is explicitly
> *not* safety related, so the natural assumption is a non-safety supply —
> which would have made every loss of offsite power blind the startup
> instruments. It does not; the diesels carry it.
>
> The manual is equally explicit about what the system must **not** do:
> *"the instruments supplied by the system do not trip the reactor or place
> the plant in a safe shutdown condition."* Losing it therefore raises an
> inoperative rod block and removes the affected channels from the trip
> comparisons — including the IRM high-high **scram** comparison, which is
> where a careless implementation would have violated the sentence above.
> A mutation case exists for exactly that (`tests/cases_nms.py`).
>
> **NO BASIS**: which emergency bus feeds which 24 VDC bus, which monitoring
> channel sits on which bus (modelled as alternating), and the battery
> endurance — the manual gives hours for the 125 VDC batteries but not for
> these, so they use the same 8 h.
3. Generator breaker, main transformer, 138/69 kV switchyard (§9.1.2.1–3) — outside
   the plant boundary, low operational value.

### 9.4 Containment

Revisited in §13.34; items 1-3 stand, the rest is updated there.

1. Drywell / suppression chamber differential pressure control
   (**R-104B** §4.1.3.5 — the section is absent from R-304B, so whether to
   model it is itself an open question).
2. Containment Atmosphere Dilution (R-104B §4.1.3.7 = R-304B §4.1.2.4.4
   Containment Combustible Gas Control: recombiners and nitrogen dilution).
3. External (vacuum) design pressure limit — 2 psig (R-104B) / 10 psia
   (R-304B); §12.6, and R-104B §4.1.3.8 warns of it directly.
4. Valve and downcomer counts; drywell cooler count (10 in R-104B, one to
   eight in R-304B — different containment types, see §13.34.1).
5. Model parameters with no numeric basis in the manual. `POOL_SURFACE_AREA`
   and `PURGE_RATE` were given one in §13.34; `DOWNCOMER_SUBMERGENCE` is now
   the nominal of a specification band sourced from the web.
6. Reactor building to suppression chamber vacuum breakers; suppression
   chamber spray; suppression pool temperature alarm setpoints; the Group 9
   signals that require a secondary containment (§13.34.6).

---

## 10. Verification procedure

### 10.1 Test suite

Every change was accepted only with all 17 scripts passing. **Six of them
could not fail until §5.44** — they counted their own failures but never
returned a non-zero exit code, so the runner reported them as passing no
matter what. Five others are reports rather than pass/fail tests and are
labelled as such; giving them an exit code would invent a verdict they do
not compute.

| Script | Scope |
|---|---|
| `validate.py` *(report)* | Cold startup to rated power; physics report incl. `[7b]` thermal limits; equilibrium column |
| `subsys_test.py` *(report)* | Subsystem behavior |
| `stress_test.py` | Combined transients |
| `atws_test.py` | ATWS with and without SLC |
| `slc_test.py` *(report)* | Boron injection and shutdown timing |
| `power_test.py` | Load following and power control |
| `diesel_test.py` *(report)* | Diesel start, loading, bus capacity |
| `rbsw_test.py` *(report)* | Service water and diesel cooling |
| `air_test.py` | Instrument air degradation sequence; CRD hydraulics `[A8]`; stuck rods `[A10]`; turbine building cooling water `[A11]`; isolation signals `[A12]` |
| `rwm_test.py` | Control rod drop accident; withdrawal sequence enforcement |
| `nms_test.py` | SRM and IRM scales and interlocks |
| `flow_test.py` | Flow path consistency; both lit and unlit states per pipe |
| `shutdown_test.py` | Cooldown and shutdown cooling |
| `conflict_test.py` | Two handlers pulling one variable in opposite directions |
| `gui_test.py` | Panel rendering; every `!` button resolves to a help entry |
| `button_test.py` | Control interlocks (62 actions + 7 blocks) |
| `display_test.py` | Drawn value vs computed value; fixed-width cell clipping |
| `diagram_audit.py` | System diagram consistency |

Two helpers are not tests themselves:

- **`tests/run_all.py`** runs them in parallel, longest first. Serial is 1965 s;
  parallel is 442 s. `--all` adds `validate` and `stress_test` (~1000 s each).
- **`tests/mutate.py`** breaks the code on purpose and checks the test reports
  it. Each case declares whether it *expects* to be caught, so cases that cannot
  be caught — and cases that exist to prove a test's own machinery is
  load-bearing — are recorded rather than lost (§5.43).

`tests/mk_aged.py` regenerates two fixtures: `rated.pkl` (just reached rated,
used by `subsys_test`) and `aged.pkl` (five days at rated, used by the accident
tests). They are **not interchangeable** — xenon, samarium, burnup and decay
heat all differ. Snapshots carry a physics-constant fingerprint and warn on
restore when it no longer matches, so regeneration is prompted rather than
remembered.

### 10.2 A test-harness defect found during verification

`button_test.py` case "SCRAM without arming (must be rejected)" began failing after
the L8 change. Diagnosis: the case fully withdraws rods with the MSIVs isolated to
observe the ARM interlock, but with no steam path the RPS scrams on high pressure
first — and the new L8 feed pump trip made this happen sooner. `scrammed` becomes
True without the button being pressed, so the interlock is unmeasurable.

**Fixed on the test side** by disabling `rps_enabled` for that case only. The
simulator was correct; the test's precondition was invalid. Same class as the earlier
RESET issue (`CHANGES.md` §5.12).

### 10.3 A console-encoding defect that stopped a verification run

`validate.py` died mid-report with
`UnicodeEncodeError: 'cp949' codec can't encode character '—'`. The default Windows
console is cp949; the test scripts contained **70 em-dashes** across 14 files, plus
`✗` and `≈`. Rather than change 73 characters, each entry point now reconfigures its
own output:

```python
for _s in (sys.stdout, sys.stderr):
    try:
        _s.reconfigure(encoding="utf-8", errors="replace")
    except Exception:
        pass
```

`errors="replace"` means it can never raise, and the `try` covers interpreters
without `reconfigure`. Same class as the U+2212 issue in `CHANGES.md` §5.12, but
fixed at the cause. (`CHANGES.md` §5.23.)

### 10.4 Unit conversions used

| From | To | Factor |
|---|---|---|
| gpm | kg/s | × 0.0630902 |
| hp | kW | × 0.7457 |
| ft³ | m³ | × 0.0283168 |
| psig | Pa (abs) | × 6894.76 + 101325 |
| in | cm | × 2.54 |
| Δk/k per °F | pcm per °C | × 10⁵ × 1.8 |
| MWt (plant scale) | — | × 3293/2436 = 1.352 |

---

## 11. Open items

1. `CS_P_MAX` / `LPCI_P_MAX` remain estimates pending pump performance curves
   (§4.6.5).
2. Xenon `XE_BURN` and `XENON_WORTH` need joint recalibration if the peak timing is
   to be corrected (§4.3.4).
3. `MAPLHGR_LIMIT` 11.2 kW/ft is still the weakest sourced number, though it no
   longer decides an outcome — with the axial reflector, MAPRAT stays under 1.0
   for any limit between 11.0 and 11.5 (§4.3.3.1). A readable 8x8 MAPLHGR curve
   would still be worth having.
4. ~~The hot channel's axial peaking is sharper than a real plant's.~~ **DONE** —
   the cause was the absent axial reflector, not tuning; `REFLECT_Z` = 0.50 brought
   the hot-channel axial peak 1.558 → **1.424** and total peaking 2.479 → **2.196**
   (§4.3.3.1, `CHANGES.md` §5.25).
5. `REFLECT_Z` is axial only. Adding a radial reflector would be more physically
   complete but costs +2021 pcm and overlaps `FUEL_ZONE`'s job (§4.3.3.1).
6. Gaps in §9 are unscheduled. The remaining **[needed]** items are §9.2-1,
   §9.2-2 and §9.3-1.

---

## 12. Second verification pass

The first pass worked outward from the manual: read a section, find the matching
constant, judge it. That finds wrong values but it cannot find **constants nobody
looked at**, and it cannot catch a verdict whose *reasoning* was wrong. This pass
worked in the other direction — from the code back to the manual — and mechanically.

### 12.1 Coverage — how much had actually been checked

Every module-level constant in `core/bwr4_core.py` and `core/bwr4_bop.py` was
enumerated and cross-referenced against the three project documents.

| | Count |
|---|---|
| Module constants | 318 |
| Not mentioned in **any** document | **72** |
| Mentioned, but absent from this document | 146 |

So roughly **a quarter of the constants had never been looked at**. Most are
internal or derived (`CHANNEL_ID`, `CPR_ITER`, `NODE_H`), but the group also held
plant data that should have had a source — SRV setpoints, ECCS loop counts,
condensate storage volume, containment alarm pressure, the heatup rate limit.

### 12.2 A verdict that was wrong for the right answer — SRV count

§4.5.1 recorded: *"manual self-inconsistent; code uses 13/6, following the main
steam chapter, which is the system-owning chapter for SRVs."*

**The premise was false.** The main steam chapter's own SRV component description
says **eleven**:

> "The Safety/Relief valves **(11)** provide overpressure protection…" and
> "…a means to operate any of the **11** safety/relief valves" (§2.5.2.1)

Full evidence:

| Source | Count | Kind |
|---|---|---|
| §2.5.2.1 — SRV component description | **11 / 7** | body text, twice |
| §10.2.2.1 + §10.2.3.1 — ADS chapter | **11 / 7** | body text, twice |
| §2.5.4.11 — one-line cross-reference to ADS | **13 / 6** | interface list |

Eleven appears four times across two chapters; thirteen once, in an interface
cross-reference. §10.2.2.1 even points back at §2.5.2.1 for the valve description,
so the two "11" statements are the same claim.

**The code's 13/6 is nevertheless correct**, for a reason the first pass did not
have: R-304B Table 1.8-1 identifies the 3293 MWt BWR/4 as **Browns Ferry**, and
Browns Ferry has 13 relief valves with 6 assigned to ADS. The code comment said so
all along ("브라운스페리는 13 대다"). The manual's 11/7 belongs to its smaller
reference plant, exactly as with the ECCS flows (§3).

So: **conclusion unchanged, justification replaced.** The lesson is that "the
chapter that owns the system" was the wrong tie-breaker — the right one was asking
which plant each number describes.

### 12.3 A third self-inconsistency in the manual — condensate pumps

| Source | Statement |
|---|---|
| §2.6.1 system overview | "The condensate that is collected in the hotwell is removed by **three** condensate pumps" |
| §2.6.2.2 component description | "**Two** motor driven condensate pumps, with a capacity of **50% each**" |

Two pumps at 50% each is internally coherent (100% total); three at 50% would be
150%. `CONDENSATE_PUMPS = 2` follows §2.6.2.2 and is right. Recorded so a later
pass does not "fix" it from the overview.

### 12.4 Dead constants — a mechanical scan

Every module constant was counted for references outside its own definition.
Eight had none. Seven were removed; the eighth was annotated as deliberate.

| Constant | Why it mattered |
|---|---|
| `CORE_FUEL_MASS` = 130,000 kg + `NODE_FUEL_MASS` + `CP_FUEL` | **A second, unused fuel-mass set.** The live constant is `UO2_CORE_MASS` = 138,000 kg — **6% apart**, and both comments read "노심 UO2 총 질량". A reader could easily quote the wrong core mass. |
| `FEEDWATER_TEMP` = 216.0 °C | Unused, and 0.4 °C off the value the BOP model actually produces (215.6 °C = 420 °F, Figure 2.0-2). |
| `VOID_RELAX_MAX` = 0.25 | Unused; `_channel_hydraulics` hard-codes **0.15**. The documented number was not the one running. |
| `CONDPUMP_DISCHARGE` = 2800 kPa | Same value under a second name as the live `CONDENSATE_HEAD` — the `RHRSW_PUMP_KW` / `RBSW_PUMP_KW` pattern from §4.8 again. |
| `RATED_MWT` = 3293.0 (BOP) | Duplicate of `RATED_MW` in core; a one-sided edit would split the plant rating in two. |
| `DC_NAME` | Unused label. |
| `LEVEL_NORMAL` = 0.0 | **Kept** — definitionally zero and never computed with, but it shows in code what the other level setpoints are measured from. Annotated so future scans do not flag it. |

All 13 test scripts pass unchanged after removal, as expected for constants with
zero references.

### 12.5 Sources found for constants the first pass never checked

| Constant | Now sourced to | Verdict |
|---|---|---|
| `CS_LOOPS` = 2 | §10.3 — "There are a total of **two** core spray pumps in the…" | **MATCH** |
| `LPCI_LOOPS` = 4 | §10.4.2.2 — "The **A and C** pumps are supplied from the division 1 switchgear and the **B and D** pumps…" | **MATCH** |
| `CONDENSATE_PUMPS` = 2 | §2.6.2.2 (see §12.3) | **MATCH** |
| flow restrictor cap | §2.5.2.2 — "limit steam flow to less than **200%** of rated steam flow during a steam line break" | **MATCH** |

Confirmed as having **no manual basis** after a direct search (the manual is a
training text; it gives system descriptions, not a full setpoint list):

`MSIV_HIGH_FLOW` (1.40), `MSIV_LOW_PRESSURE` (825 psig), `MSIV_LOW_VACUUM`
(30 kPa) — Table 4.4-1 lists the isolation *conditions* but gives **no numeric
setpoints**. ⚠ The claim that "the code matches" those conditions was wrong
when written and stayed wrong for eleven sections: two of the six were absent
and two extras were present. Corrected in **§13.20** — read that instead. Also `HEATUP_LIMIT` (55 °C/h — close to the industry-standard
100 °F/h but not stated in the manual), `SRV_FLOW_EACH`, `SRV_BLOWDOWN`,
`CST_VOLUME`, `CST_TEMP`, `ECCS_START_DELAY`, `CONT_ALARM_PRESSURE`, `POOL_TEMP0`,
`RHR_SPRAY_FLOW`, `RHR_HX_UA`, `TSV_FAST_RATE`.

### 12.6 One more gap in the reference

Table 4.1-1 also gives **"Maximum External Design Pressure — 2 psig"** for both the
drywell and the suppression chamber. The model has no external (vacuum) limit; it
judges internal pressure only. Low priority — the vacuum breakers (§4.6.3) already
prevent the condition this limit guards against — but it is a stated design value
with no counterpart in the code.

---

## 13. Third verification pass — prompted by the help text

The third pass was not planned as a verification pass. It began as a request to
rewrite the control-panel `!` help popups, and the discrepancies surfaced because
**writing a plain-language explanation forces you to state the number out loud.**

### 13.1 APRM scram setpoint — reading across two rows of one table

The `RPS` help text said "고출력 120%" (high power 120%). The code said the trip
was flow-biased and worked out to 108% at rated flow. Both were wrong.

*Table 5.4-1 APRM Scram and Rod Block Settings* (`part2:3366`) gives:

```
APRM High           .66(W) + 42%        Rod Block
                    108% Max.
APRM High-High      .66(W) + 51%        Scram
Thermal Power       113.5% Max.
APRM Fixed Power    118%                Rod Block & Scram
```

The code had taken the **clamp** from the scram row (113.5) and the **bias** from
the rod-block row (42). At rated flow that put the rod block and the scram at the
same 108%, so the rod block could never act first — the defence-in-depth it exists
to provide was absent at every flow.

| Core flow | Rod block | Scram (before) | Scram (after) |
|---|---|---|---|
| 100% | 108.0 | **108.0** | 113.5 |
| 90% | 101.4 | 101.4 | 110.4 |
| 70% | 88.2 | 88.2 | 97.2 |
| 50% | 75.0 | 75.0 | 84.0 |

The first pass had verified this section against Table 7.1-1, "Rod Withdrawal
Blocks and Setpoints," and recorded MATCH. That table is correct — but it only
covers rod blocks. Checking a scram setpoint against a rod-block table cannot
detect a scram/rod-block confusion, because the value being compared against is
the one that was wrongly copied.

**Lesson added to the method (§10):** when a table's rows are *actions* (block,
scram, alarm), verify the action column, not just the setpoint. A MATCH on the
number means nothing if the row is wrong.

### 13.2 Help text is the fifth place a constant is quoted

An audit of the 49 help entries against live module values found three stale
citations. Every one had been correctly updated in the code, in `CHANGES.md`, in
this document, and in `physics.md` — and missed in the panel.

| Help text said | Actual | Went stale at |
|---|---|---|
| RBSW pumps 543 kg/s, 336 kW | 734 kg/s, 454 kW | §5.19 plant-scale correction |
| 63 fuel rods + 1 water rod per bundle | 62 + 2 | §2.2.2.1.3 correction |
| APRM high-power scram 120% | 0.66W + 51%, 113.5% at rated flow | never correct (§13.1) |

The fix is structural rather than a set of edits: help text now lives in
`panel/help_text.py` and quotes constants through f-strings, so these three
classes of drift cannot recur. Values with no constant behind them were the ones
that rotted.

### 13.3 Dead imports

Removing the old inline `HELP` dictionary exposed 38 unused `core` imports in
`panel/bwr_panel.py` (111 → 73). Ten of them (`N_CHANNEL`, `ROD_GROUP_WIDTH`,
`CS_BUS`, `LPCI_BUS`, `DW_TEMP0`, `DRYWELL_VOLUME`, `WETWELL_GAS_VOLUME`,
`RHR_HX_UA`, `BYPASS_K`, `INHG`) were already dead before this change, and four
were duplicate imports of the same name. This is the same class of finding as the
dead-constant scan in §12, applied to a file that scan had not covered.

### 13.4 Condensate booster pumps — added, but the stated reason was wrong

§9.2 listed condensate booster pumps as the highest-priority secondary-plant gap,
with this justification:

> `CONDENSATE_HEAD` is lower than the plant's, so the pressure window for feeding
> **during a depressurized cooldown** is too narrow. Directly affects an accident
> response path.

Re-verifying before implementing showed the mechanism was wrong.

`CONDENSATE_HEAD` is only read in the branch taken when `rfp_steam_ok` is false.
With main steam present that requires `reactor_pressure <= RFP_MIN_STEAM_P`
(1500 kPa), and at that pressure `clip((2800 − 1500)/500)` is already 1.0. Running
a normal depressurized cooldown twice, once with 2800 kPa and once with 4238 kPa,
and comparing feed capability every second gave **zero seconds of difference**.

The difference exists only where main steam is absent while the vessel is still at
2800–3738 kPa — that is, **MSIV closed, riding the SRVs down**. In that scenario
the window lasted 1803 s at up to 2100 kg/s.

With high-pressure injection available the outcome barely moves, because HPCI and
RCIC cover the gap. What the boosters decide is the case where those are lost too
(isolation cooldown, 3 h, measured after implementation):

| Condition | Min level | Core uncovered | Seconds fed |
|---|---|---|---|
| HPCI/RCIC available, 2 boosters | −196.6 cm | 0.0% | 57 |
| HPCI/RCIC available, 0 boosters | −202.7 cm | 0.0% | 1 |
| **HPCI/RCIC lost, 2 boosters** | −493.6 cm | **0.0%** | 3719 |
| **HPCI/RCIC lost, 0 boosters** | −513.8 cm | **4.2%** | 106 |

Worth adding — but for *isolation plus degraded high-pressure injection*, not for
normal cooldown. The gap list in §9.2 has been corrected accordingly.

**What was implemented.** `BOOSTER_PUMPS` 2 × `BOOSTER_EACH` 1050 kg/s,
`BOOSTER_HEAD` 4238 kPa (600 psig abs, §2.6.2.5). `AUX_BOOSTER_EACH` is derived
from the pressure rise ratio rather than written down, so it follows a change to
the discharge pressure. The feed path is now three pumping stages in series:
**throughput is set by the narrowest stage, discharge pressure by the last stage
that actually boosts.** Stopping all boosters drops the head back to
`CONDENSATE_HEAD`, which is what the old single-stage model always did.

**Trap hit again.** Adding `self.boost_pumps` in `__init__` broke `gui_test` with
`AttributeError` because `/tmp/aged.pkl` predates the attribute. This is the
documented `aged.pkl` staleness trap; regenerating with `mk_aged.py` fixed it and
left the equilibrium unchanged (98.85% power, 98.4% rod withdrawal).

### 13.5 Feedwater control is two-element, not three

Writing the booster explanation into the panel help surfaced an unrelated error.
The `Feed` help said the model uses three-element control and listed steam flow,
feedwater flow and level error. The code is

```python
feed_setpoint = clip(steam_out + (level_setpoint - water_level) * 8.0, 0, 2200)
```

— steam flow feedforward plus level feedback, with **no feedwater-flow term**.
Two-element. The pre-existing help text was self-contradictory ("3요소 제어
(증기유량 + 수위편차)" — naming three, listing two) and the rewrite had expanded
the wrong half. §6 item 3 of `CHANGES.md` had it right all along.

Corrected to state the actual two-element law and note that the real plant adds
the third term.

### 13.6 Feedwater heater strings — added, and the stated direction was backwards

§9.2 described the missing three parallel heater strings this way:

> currently reduced to one string, so isolating a string (a real load-reduction
> cause) has no effect on feedwater temperature.

Two things were wrong.

**The stage count was already right.** `HEATER_NAMES` is `(LP1, LP2, LP3, HP1,
HP2)` — three low-pressure and two high-pressure stages, exactly what §2.6.2.6
("one drain cooler and three low pressure heaters") and §2.6.2.8 ("two heaters
per string") give. Parallel strings collapse thermodynamically, so a single
train of five stages is the correct reduction. What was missing was only the
ability to *isolate* a string. (The module docstring, `README`, and the panel
help all said "one high-pressure stage" — corrected separately.)

**The direction was inverted.** Losing feedwater heating makes reactor power
*rise*, not fall: colder feedwater collapses voids, less void means better
moderation, and reactivity goes up. Measured on this model with rods fixed:

| Feedwater temperature | Power change |
|---|---|
| −10 °C | **+2.87 %p** |
| −20 °C | **+5.78 %p** |
| −40 °C | **+11.78 %p** |

This is the standard FSAR *loss of feedwater heating* transient — a slow power
increase that challenges the thermal limits. An operator reduces power in
response; the plant does not do it by itself.

#### The model had to change before the count could matter

Outlet temperature was `shell[i] − HEATER_TTD` with a **constant** 3 °C terminal
difference, so isolating a string would have done nothing. Adding a string count
as state produces no physics on its own.

Replaced with an NTU formulation. The shell side condenses at constant
temperature, so effectiveness is `ε = 1 − exp(−NTU)` and

```
TTD[i] = (shell[i] − t_in[i]) · exp(−NTU[i])
NTU[i] = HEATER_NTU[i] · (strings / FWH_STRINGS) · (RATED_STEAM / m_fw)
```

`HEATER_NTU` is not written down. `_design_ntu()` back-solves it so that TTD
equals `HEATER_TTD` at the design point — the same approach as the existing
`_design_phi()`, so changing the stage layout or the design TTD carries through
automatically.

Because a colder stage outlet raises the next stage's approach temperature, the
penalty **compounds down the train**: with two strings LP1 loses 3.1 °C while
HP2 loses 4.7 °C.

| Strings | Final feedwater | vs design | Power | MFLPD | MCPR | MAPRAT |
|---|---|---|---|---|---|---|
| 3 (design) | 214.3 °C | — | 98.37% | 0.895 | 1.479 | 0.947 |
| 2 | 209.9 °C | −4.4 °C | 99.72% | 0.905 | 1.469 | 0.958 |
| 1 | 193.4 °C | −20.9 °C | 104.85% | 0.943 | 1.430 | **0.998** |

At one string MAPRAT reaches 0.998, just short of the limit. With the thermal limits of §5.21
in place, this transient now exercises them directly — which is the reason to
have it.

#### Feedwater temperature lag

With no lag the outlet temperature changed within a single step, and bypassing
all heating spiked power to **174% in 4 s**. Real feedwater temperature trails
the heater metal heat capacity and the transit time from the last heater to the
sparger. `FW_TEMP_TAU = 20 s` (no manual basis) brings that to **116.8% at 10 s**,
ending in an APRM high-power scram. Steady state is unaffected by the lag, so
the table above is unchanged.

#### Not modeled

Isolating a string raises the pressure drop through the remaining ones as the
square of flow, which at a real plant often forces a load reduction. There is no
hydraulic resistance model here, so full flow still passes.

### 13.7 Equilibrium moved 98.85% → 98.41% — accounted for

Regenerating `aged.pkl` after the NTU change lowered the rated equilibrium by
0.44 %p. This is not a physics error.

At the equilibrium flow of 1781 kg/s, `ntu_scale` is 1.026, so TTD is slightly
smaller and feedwater comes out **0.22 °C warmer**. That alone accounts for
−0.06 %p.

The rest is the **rod position where the automatic startup stopped**. The
controller breaks at `total_power > 0.995`; a slightly different feedwater
temperature during the ramp leaves it at a marginally different position.

| | old `aged.pkl` | new `aged.pkl` |
|---|---|---|
| Rod sum | 5906.33 | 5903.86 |
| Mean withdrawal | 98.4388% | 98.3976% |
| Power | 98.85% | 98.41% |

0.041 %p of rod travel × `ROD_WORTH` 29500 = **12 pcm**, and roughly 13 pcm is
what a −0.44 %p power change requires here. The two agree.

### 13.8 What `button_test` is for

Three checks were added to `button_test`: isolate a string, see feedwater fall,
restore and see it recover. The recovery check failed. Debugging showed the
plant was **already scrammed with the turbine offline** by that point — earlier
checks in the file walk recirculation, the pressure regulator and the condensate
pumps down in sequence. With no extraction steam, feedwater sat at 38.7 °C
regardless of string count, so the *fall* check had also been measuring the wrong
thing.

`button_test` verifies that **a control changes state**, not that physics
follows. The state check stayed (50/50); the physics moved to `subsys_test [T]`,
which starts from a clean rated state. The note already in memory — "button_test
failures are usually test-construction problems, from state left by earlier
items" — applied exactly.

### 13.9 Snapshot staleness — the loud failure was the harmless one

Every accident test loads `/tmp/aged.pkl`, a 5-day operating snapshot that takes
ten minutes to build. It broke three times on 2026-09-08, each time the same way:

```
AttributeError: 'BalanceOfPlant' object has no attribute 'boost_pumps'
```

`pickle` does not call `__init__` on restore — it only writes back the attributes
it saved. An attribute added to `__init__` after the snapshot was written comes
back missing, and nothing supplies a default.

Two kinds of staleness were being handled backwards.

| | What happened | Is the snapshot wrong? | Behaviour |
|---|---|---|---|
| Field added | new attribute in `__init__` | **No** — the rest is fine | crashed loudly |
| Physics changed | diffusion or a constant edited | **Yes** — the stored equilibrium belongs to the old physics | passed silently |

The second is the dangerous one and produced no error, because attribute *names*
do not change. §5.17's `POOL_MASS` incident was exactly this: an old snapshot was
used and before/after measurements came out identical to the decimal, which is
how it was noticed at all.

Up to now the first case crashing was what *incidentally* forced regeneration and
so happened to fix the second. That was luck, not a check.

**Field added — fill quietly.** `_RestoreDefaults.__setstate__` lays down a fresh
instance's defaults, then overwrites with the saved state, so only genuinely new
fields take defaults. It names what it filled. `Reactor`, `Containment` and
`BalanceOfPlant` all construct without arguments, so this needs nothing else.

**Physics changed — say so loudly.** `physics_fingerprint()` collects every module
constant (347 entries: scalars, numeric tuples, numpy arrays) into the snapshot,
and restore reports the ones that differ **by name**:

```
[스냅샷] !! 이 스냅샷을 만든 물리와 지금 물리가 다르다.
           core.EXCESS_REACTIVITY       23281.0 -> 23500.0
           core.REFLECT_Z               0.5 -> 0.0
```

Maintaining a curated list of "constants that affect equilibrium" would rot, so
everything is captured and only the differences are reported. An irrelevant
constant showing up is for the reader to judge — better than silence.

Verified on all four cases: old snapshot without a fingerprint (notice, loads
fine), fresh snapshot with unchanged physics (silent), missing attributes
(filled, named), changed physics (warned, constants named — reproduced by
reverting §5.25's axial reflector).

**The point is not `__setstate__`.** Removing the crash also removes the accident
that used to force regeneration; adding it alone would have made things *worse*,
because adding an attribute and changing physics in the same session would then
produce no signal at all. Replacing the accident with an explicit check is the
actual change.

### 13.10 Offsite power fast transfer — the quoted section was right this time

§9.3 item 1 named the NSST/RSST fast transfer as the first electrical gap. Unlike
§13.4 (wrong scenario) and §13.6 (inverted direction), re-verification found the
citation accurate — with one clause missing from the original note.

§9.2.3.1 — *"Off site power is fed from the 138KV AC and 69KV AC substations
through the NSST and RSST respectively ... busses normally power by the NSST."*

§9.2.3.3 — *"searching logic on the buses first checks the availability of the
RSST source. If the RSST is available, a fast transfer to that source is made
approximately **5 cycles** after bus undervoltage is detected. The function is
only available from the normal to the backup source of preferred power and
**not in reverse**."*

The last clause was missing: **the transfer is one-way.** Once on the reserve
source, restoring the normal source does not transfer back.

#### What the model did, measured

`bop.ac_power` was a single boolean, so any offsite loss was total. Measured from
the rated equilibrium: turbine trip on low vacuum and scram within about 20 s,
MSIV closed, all three diesels loaded. There was no way to express the common
case where only the normal source is lost.

#### After

| Scenario | Source | Power | Flow | Diesels | Outcome |
|---|---|---|---|---|---|
| **Normal source lost only** | **RSST** | **98.37%** | **100.0%** | **0** | **no event** |
| Normal lost + transfer fails | none | 2.48% | 41.7% | 3 | scram at 11 s |
| Both lost (LOOP) | none | 2.48% | 41.7% | 3 | scram at 11 s |
| Reserve lost only | NSST | 98.37% | 100.0% | 0 | no event |

The one-way rule is verified separately: after transferring to RSST, restoring
NSST leaves the plant on RSST.

Five cycles is 83 ms, well under the model's minimum timestep, so the transfer
completes within one step. What the model represents is **which source feeds the
buses** and **whether the transfer succeeded**; the voltage gap the motors see is
one inertia decay (`power_frac *= exp(-0.083/4.0)`, about 2%).

`ac_power` became a derived property so that every existing caller — the panel
checkbox, `Reactor._ac_power`, four test sites — kept working untouched.

#### A design error caught while building

The first version auto-reconnected whenever `offsite_source` was None and any
source was available. That made the *transfer-failure* case reconnect to RSST on
the following step, erasing the failure. Failing the fast transfer means the plant
could not get to RSST even though RSST was live; reconnecting it immediately
contradicts that. Auto-reconnect was removed and `restore_offsite()` added as an
explicit operator action, which is what the plant does.

#### The fingerprint earned its keep

The §13.9 physics fingerprint flagged the three new constants (`NSST_KV`,
`RSST_KV`, `LINE_HZ`) on the first load of the old snapshot. Regenerating produced
a **bit-identical** state (98.36624% power, rod sum 5903.4995), which is both the
expected result — no transfer occurs during startup — and the proof that this
change is neutral in normal operation.

Without it, deciding whether the snapshot needed regenerating would have been a
memory exercise. §13.7 records what happens when that memory fails.

### 13.11 125 VDC divisions — the AC side was already three-division

The DC system was a single scalar for the whole plant while the AC side had three
independent buses. §9.4 gives the structure directly.

- §9.4.1 — *"divided into **four separate divisions**. Three divisions are for
  engineered safety features (ESF) equipment."* §9.4.3.4 names the safety buses
  **A1, B1, C1**.
- §9.4.1 — *"each distribution bus has **two sources** ... a solid state battery
  charger ... and ... a 125 VDC battery."* One of each, not two.
- §9.4.3.4 — **safety-related chargers are fed from emergency AC; non-safety
  chargers from normal AC.** This is what makes the DC divisions inherit the AC
  divisions' fate.
- §9.4.3.3 — losing a charger leaves that division's battery carrying its own bus.

#### What partial failures now look like (2 h)

| Scenario | A1 | B1 | C1 | D |
|---|---|---|---|---|
| Normal | 100% | 100% | 100% | 100% |
| **LOOP, diesels running** | 100% | 100% | 100% | **75%** |
| **Bus 103 lost** | 100% | 100% | **75%** | 100% |
| **A1 charger failed** | **75%** | 100% | 100% | 100% |
| SBO | 73% | 73% | 73% | 75% |

The LOOP row is §9.4.3.4 made visible: with the diesels carrying the emergency
buses, only the **non-safety** division discharges. A single scalar could not
express it. Control-power endurance in SBO is unchanged at 8 h.

#### Deliberately not modeled

Table 9.4-1 (per-division load list) is a figure and is absent from the extracted
text, and no section states which division feeds HPCI or RCIC. Their DC drain is
therefore shared across whichever ESF divisions are still alive rather than
assigned to one. Mapping A1/B1/C1 onto buses 101/102/103 is structural — it
follows from the division principle — not an invented load assignment.

#### Recharge time was 144× too fast

§9.4.3.1 — *"chargers have enough capacity to carry the steady state loads while
recharging the batteries from minimum voltage to charged state **within 24
hours**."* The model used `dt / 600.0` — ten minutes. A brief return of AC
restored full battery margin almost instantly, overstating readiness for a second
loss. Now `DC_RECHARGE_H = 24.0`.

#### The 8-hour endurance and the manual's 2 hours are different quantities

§9.4.3.3 — *"The 125 VDC batteries can meet **worst case** loads for **two hours**
in the event of a blackout"*, where worst case is LOCA **coincident with** loss of
offsite power and the batteries are *"loaded heavily during the first minute due
to initiation of engineered safeguard equipment."*

`DC_ENDURANCE_H = 8.0` is a **standby-load** figure. The two are not comparable
directly. But the model's heaviest modeled load (HPCI and RCIC together) still
gives 4.2 h, short of two hours' worth of margin — the gap is the ESF inrush,
which the drain model does not contain. Recorded in §7 rather than changed:
cutting 8 to 2 without adding the inrush term would make standby endurance four
times worse than the plant's, which is the opposite of the intent.

---

### 13.12 The layout audit was measuring a world the user never sees

Two blind spots, both found the same way: the user reported a defect the audit
had passed.

#### Blind spot 1 — the audit ran DPI-unaware while the app ran stretched

`diagram_audit.py` builds the real windows and reads real canvas coordinates,
so it looked trustworthy. But Windows was scaling the app 125% after it drew,
and neither the app nor the audit knew. Text metrics in the audit therefore came
from glyphs 20% smaller than the ones on screen — every clearance check was
run against a picture nobody sees.

Turning on DPI awareness (§5.35) put the audit and the app in the same world.
It immediately reported **8 problems** that had been there all along:

```
[계통도]      배관이 글자 위로: '2/2  4238kPa'
[전기계통도]  글자 겹침: '노심살수 A' ↔ '0 kg/s'      (부하 상자 6곳 전부)
[전기계통도]  배관이 글자 위로: '125V DC'
```

The awareness call lives in `panel/bwr_diagram.py` at import time, not in the
panel's `__main__`, precisely so the audit cannot diverge from the app again.

#### Blind spot 2 — the audit only ever saw one plant state

The audit built the diagram from a `Reactor()` at rated conditions. In that
state the diesels are stopped, so the diesel readout says `정지 (대기)` — 84 px.
Running, it says `가동 454 kW (13%) · 연료 98%` — **220 px**, and it crossed the
DC feed line. The user saw it on screen; the audit had never rendered it.

Fixed by auditing each diagram in **three** states — as built; with the display
fields forced to their **longest** strings (diesels running at overload,
`DG_RATING_KW * DG_OVERLOAD` = 3850 kW, fuel 100%, every pump on); and with the
buses dead, which is the only state that produces strings like
`전원 상실 — 사용 불가`. No physics is stepped; only the values the diagrams read
are set. Layout breaks at the longest string, so those are the states worth
checking.

```
[전기계통도 · 가장 긴 글자]  배관이 글자 위로: '가동 454 kW (13%) · 연료 99%'
[계통도 · 전원상실]          글자가 상자를 넘음(세로): '수조 0 / 살수 0 MW'
```

#### A third check that did not exist: text overflowing its own box

Half of what a person calls "overlapping" is a label wider than the box drawn
around it. Nothing checked for it. Added as check 8: any text whose centre lies
inside a box must clear that box's sides by 8 px and its top and bottom by 2 px
(labels sit at `y0 + 13`, so 3 px is all the headroom there is).

The horizontal half found the DC load subtitle sitting in a 280 px box with
**4 px** of margin — everything else in both diagrams has ≥14 px. Box widened
to 320 px.

The vertical half found the RHR heat exchanger, the one box carrying four lines
(name, subtitle, status, duty) in 66 px. With power available the status line is
short and it fits; with power lost the fourth line punches 5 px through the
floor. Box grown to 80 px. This is the defect visible in the screenshot the user
would have seen — text sliced in half by its own frame.

#### What this cost, and what it is worth

Three of the twelve defects in this pass were reported by the user before the
audit found them. All three are now covered by the audit, which is the point —
the instrument gets fixed when it misses, not just the drawing. Both diagrams
now report **0 problems in all three states**, at the DPI the app actually
runs at.

The pattern is now on its third repeat (see §13.9 and 5.31): *a verification
tool that does not run under the same conditions as the thing it verifies is
not a verification tool.* Timer nondeterminism, snapshot staleness, and now
display scaling — same failure, three costumes.

---

### 13.13 An energized emergency bus cannot draw zero

#### What the user saw

Stop every service water pump during a loss of offsite power and the diesel
readout said **`가동 0 kW (0%)`** — a generator running, feeding nothing, for the
five minutes it takes to overheat without cooling water.

#### What the manual says

§9.2.2.2, on the diesel generators:

> All necessary auxiliaries directly associated with each diesel generator
> unit, such as **cooling water, lubricating oil, circulating pumps,
> ventilating fans, and battery chargers** are powered from their associated
> emergency bus.

§9.2.2.1 adds that each 4160 V bus feeds 480 V unit substations, motor control
centers and power panels. §9.4.1 puts one 125 VDC battery charger on each
distribution bus.

So the model was not merely displaying badly — it was missing a load. The big
ECCS motors are switched; the charger, the MCC and the diesel's own auxiliaries
are not.

#### What was added

```python
BUS_CHARGER_KW = 50.0    # one 125 VDC charger per bus (9.4.1)
BUS_MCC_KW = 100.0       # 480 V MCC, lighting, instrument panels (9.2.2.1)
DG_AUX_KW = 100.0        # jacket water / lube oil pumps, ventilating fans (9.2.2.2)
```

Charger and MCC load whenever the bus is energized; the diesel auxiliaries only
while that bus's diesel is running. A bus opened by the operator draws nothing.

| | Bus 101 | Bus 102 | Bus 103 |
|---|---|---|---|
| Normal (offsite) | 904 kW | 904 kW | 450 kW |
| LOOP (diesel) | 704 kW | 704 kW | 250 kW |
| All ECCS started | 3304 kW (94%) | 3304 kW (94%) | 4258 kW (122%) |

Overload trip is at 3850 kW (3500 × 1.10). Buses 101/102 keep margin; 103 was
already over it before this change (4008 kW) because two RHR pumps hang on one
bus. No behaviour class changed — only the numbers moved.

#### NO BASIS — the three kW figures

The manual names these loads but gives **no ratings** for them. The three
constants are engineering estimates, sized so that:

- the total (150 kW standing, 250 kW with the diesel running) is ~4–7 % of the
  3500 kW continuous rating — the right order for house load on a division;
- they do not change any overload verdict (checked above);
- they are large enough that the display can never read zero on a live bus.

Recorded here as **NO BASIS**. If a load list for the 480 V buses ever turns up
(Figure 9.2-1 is a drawing and is not in the extracted text), these should be
replaced with it.

#### Side effect: diesel fuel burn

`DG_FUEL_IDLE_FRAC` makes an unloaded diesel burn 30 % of full-load fuel. With
the house load a "no-load" diesel is now at 7 % load rather than 0 %, so
standby endurance shortens slightly. That is the correct direction — the
previous figure described a diesel with its own auxiliaries switched off, which
cannot happen.

#### Why the audit missed it

`diagram_audit` checks layout, not content: it asks whether the text `0 kg/s`
overlaps its neighbours, never whether `0` is the right number. `power_test`
and `diesel_test` printed the bus loads but had no expectation attached to
them. Nothing in the suite could have failed on this.

`tests/display_test.py` (new, §5.36) now reads the canvas coordinates back and
compares them with the model, including the invariant that fell out of this
finding: **every energized bus draws more than zero.**

---

### 13.14 Two handlers pulling the same variable in opposite directions

#### The defect

With offsite power lost, the recirculation pumps settled at **15.8 % speed and
stayed there forever**, holding core flow at 41 % instead of the 30 % natural
circulation the model, the reference and every document claim.

```
     t[s]   recirc A / B     core flow
       30   0.158 / 0.158      41.1 %
     1800   0.158 / 0.158      41.1 %
```

#### Root cause

Two step handlers write the same variable in the same step:

| handler | on LOOP | drives recirc toward |
|---|---|---|
| `_ac_power` | offsite gone → motors dead | **0** |
| `_recirc_trip` | turbine trip → EOC-RPT latched | **`RECIRC_MIN` = 0.30** |

Both use the same relaxation constant, so alternating pulls converge on a fixed
point near `RECIRC_MIN / 2` — exactly the 0.158 measured.

The substantive error is in `_recirc_trip`: **a pump trip can only slow a pump
down.** This historical fix still coasted an AC-powered RPT to minimum speed. That
interpretation was incomplete: opening the motor breaker requires coastdown to
zero. CHANGES 5.57 supersedes this behaviour with one owner of actual pump speed. The level-4 runback immediately below it had carried
the guard `if recirc > RECIRC_RUNBACK:` since it was written. The RPT block did
not.

```python
if self.recirc_a > RECIRC_MIN:          # slow down only
    self.recirc_a += (RECIRC_MIN - self.recirc_a) * k
```

#### Effect

| | before | after |
|---|---|---|
| Core flow 60 s after LOOP | 41.7 % | **30.0 %** (= `NATURAL_CIRC`) |
| Recirc pumps in SBO | 16.7 % | **0.0 %** |
| First HPCI/RCIC start | 11.4 min | 10.9 min |
| Suppression pool to 100 °C | 18 h 57 min | 18 h 50 min |

`slc_test`, `atws_test`, `shutdown_test` and `flow_test` are **byte-identical**.
The guard only engages below `RECIRC_MIN`, a state reachable only by losing
power; ATWS-RPT coming down from 100 % never touches it.

#### The part worth remembering

`power_test` had been printing this line on every run:

```
재순환펌프 (비안전)   16.7 %   ← 아직 돈다(문제)
```

That marker is generated by the test (`"← 정지" if not running else
"← 아직 돈다(문제)"`), not a stale comment. The suite had detected the defect,
labelled it a problem in its own output, and passed anyway. It was found months
later only because a user asked an unrelated question about feedwater.

**A check that prints "(problem)" and still returns success is not a check.**
Either it is an expected condition and the label is wrong, or it is a failure
and the test must fail. This one is now an assertion: the recirculation pumps
must read zero once offsite power is gone and the coastdown is complete.

#### And the new assertion was itself broken

Written, it printed nothing and the suite passed — which proves nothing, because
with the fix in place the branch is never reached. Re-running it against the
*unfixed* core (`git stash push core/bwr4_core.py`) turned up a `NameError`: the
`FAIL` list had been declared below the loop that uses it. The check would have
crashed rather than reported.

Both directions are now verified:

```
버그 코드: exit 1   재순환펌프 (비안전): 전원이 없는데 16.7 % 로 돈다  (LOOP, SBO)
고친 코드: exit 0
```

A new check must be run against the defect it was written for. Passing on fixed
code is the half that carries no information.

---

### 13.15 Sweeping for the rest of the conflict class

§13.14 was one variable with two owners pulling opposite ways. The two handlers
sit 400 lines apart; reading cannot find the next one. `tests/conflict_test.py`
(11 s) looks for it mechanically.

#### Method

Wrap all 22 handlers `step()` calls, snapshot the numeric state before and
after each — including the sub-objects (`bop`, `containment`) and numpy arrays
(as sum and max). Within one step, two handlers moving the same value in
opposite directions is a candidate.

#### Telling legitimate summation from a tug-of-war

Shared writes are often correct: ECCS adds cold water while the core boils it
off, so both write `water_mass`. What separates them is whether the value
actually moves.

```
        |net change| / sum of |individual changes|

  summation → large   (one side genuinely wins; the value keeps going)
  tug-of-war → ~0     (no net motion, yet both keep pushing)
```

Threshold 0.20. It separates the §13.14 defect from every legitimate case
cleanly.

#### Result across 14 scenarios

Exactly three shared values, all classified as summation:

| value | handlers | ratio | reading |
|---|---|---|---|
| `pool_temp` | `_eccs` + `_rhr` | 0.55 | SRV discharge heats, RHR cools |
| `water_temp` | `_vessel` + `_eccs` | 0.31 | boil-off vs cold injection |
| `water_mass` | `_slc`/`_eccs` + `_vessel` | 0.91 | injection vs steam out |

Everything else has exactly one writer. The recirculation pair was the only
violation of single ownership, and it was the defect.

#### The detector was wrong twice before it was right

Both failures produced a confident "no problems found".

1. **Only ever run against fixed code.** Run against the pre-§13.14 core it
   caught nothing — collecting handlers with `dir()` had wrapped methods with
   other signatures (`_reactivity()`) and the run died. Fixed by parsing
   `self._x(dt)` out of `step()`'s own source.
2. **Blind to most of the state.** Snapshotting only `r` and `r.bop` missed
   everything `_containment` writes, and handlers that touch only numpy arrays
   — `_fuel_temperature`, `_burnup`, `_thermal_limits`, `_decay_heat` and three
   others — were **never observed writing anything at all**. Seven of 22
   handlers were outside the instrument while it reported "all clear".

Both were found only by running the detector against the defect it was built
for. This is the third time in two days that a check passed while measuring
nothing (see §13.12, §13.14). The rule that keeps surviving:

> A new check is not finished when it passes. It is finished when it has been
> shown to fail on the defect it was written for.

---

### 13.16 Startup instrumentation — SRM and IRM (NRC 5.1 · 5.2)

#### The gap

Half of the simulator — cold startup — ran on an instrument that does not
exist. A single APRM covered `1e-7 %` to `120 %` of rated: **nine decades**.

The display was the smaller half of the problem. `rod_block()` held only the
APRM flow-biased block, the RBM, and damage states. **Nothing prevented rod
withdrawal with no neutron indication at all** — the interlock that matters
most in a real startup was absent.

#### What the manual fixes, and what it leaves free

Two sentences in §5.2.3.3 pin the whole scale:

> "Typically, **100/125 on range 10** roughly corresponds to **40% of rated**
>  core thermal power."
>
> "When the SRM count rate is between **10⁴ and 10⁵ cps**, the IRM channels
>  should be **on scale in range one**."

The first fixes the IRM absolutely: range 10 full scale = 50% rated, ten ranges
covering five decades (§5.2.1) = half a decade per range.

The second fixes the SRM *relative to the IRM*. The one constant the manual
does not give is counts per unit power. Setting `SRM_CPS_PER_POWER = 1e10`
places IRM range 1 at 25/125–75/125 when the SRM reads **3.2e4 – 9.5e4 cps** —
inside the quoted band.

| check | value | manual |
|---|---|---|
| Range 10 at 100/125 | 40.0% rated | "roughly 40%" ✓ |
| IRM range 1 on scale at | SRM 3.2e4–9.5e4 cps | "between 10⁴ and 10⁵" ✓ |
| Range 9 full scale | 15.8% rated | "on range nine the APRMs begin to indicate" ✓ |
| Cold shutdown count rate | 65 cps | between downscale (3) and retract permit (100) ✓ |

The last two were **not fitted** — they fell out. The cold-shutdown count rate
comes from the existing subcritical source multiplication (`SOURCE = 1e-9`) and
lands exactly where a real startup begins.

**NO BASIS**: `SRM_CPS_PER_POWER` and `SRM_RETRACT_DECADES` have no value in
the reference. The first is pinned by the overlap statement above. The second
(4 decades of detector travel attenuation) is set by what the drive exists to
do — hold the count rate on scale until the upscale block auto-bypasses at IRM
range 8, which spans 3.5 decades.

#### Neutron detectors do not see decay heat

Reading `total_power` would have been wrong by six orders of magnitude:

```
one hour after scram   total_power 0.962%   core_power 3.0e-09   SRM 29 cps
```

Fission power (`core_power`) is the right quantity, and it settles to the
cold-shutdown count rate, which is what an SRM actually shows.

#### Interlocks (Table 5.1-1 · 5.2-1)

Implemented in full, including the auto-bypass matrix, which is what makes the
handover work: SRM blocks bypass on IRM range (2 for downscale/retract, 7 for
upscale), IRM downscale bypasses on range 1, IRM trips bypass in RUN.

The IRM 120/125 scram is a **new protective function** the model did not have.
Between roughly `1e-5` and a few percent of rated, no other scram signal
applies; that band is exactly what the IRM guards.

#### The model pushed back twice, and was right both times

1. Two decades of detector travel was not enough — the SRM upscale block stayed
   in for 17,177 s of the startup and power could not rise. The drive exists to
   prevent precisely that; 4 decades.
2. With no mode switch, startup **scrammed at 48% rated** on IRM range 10.
   That is correct behaviour: 120/125 of a 50% full scale. Real plants take the
   mode switch to RUN first, which bypasses the IRM trips. The missing piece was
   the operator action, not the logic.

#### Snapshot migration

Instrument defaults are the *cold startup* configuration. Applied to an
at-power snapshot they scram it within five seconds. `__setstate__` now detects
a pre-NMS snapshot and aligns the instruments to the plant state.

#### Verification

`tests/nms_test.py`, 15 checks. Each interlock is verified by **constructing the
condition and observing the trip**, not by observing its absence. Confirmed
against a deliberately disabled `nms_rod_block()`:

```
망가뜨린 코드: exit 1   불일치 4건 (SRM 다운스케일·업스케일·IRM 업·다운스케일)
복원 후:       exit 0
```

Two separate gaps closed on the way.

**Value cells were clipping from the front.** Fixed-width labels are
right-aligned, so an overlong value loses its *leading* characters silently —
a four-channel readout showed three numbers with nothing to say one was
missing. Five cells were affected, four of them pre-existing. One had already
been truncated defensively with `text[:18]`, but by **character** count: the
string was 18 characters and 26 columns, because Hangul is double-width.
`display_test` now sweeps every fixed-width label and counts columns, and it
caught a fifth case the hand-written probe had missed.

**Help buttons pointed at entries that did not exist.** `gui_test` now asserts
that every `!` help button points at an entry that exists. `show_help()` falls back to "설명이
없습니다" for a missing key, so two buttons shipped with no help and every test
passed. 52 keys, 52 entries, 0 missing.

---

### 13.17 Service and instrument air (NRC 11.6)

#### Why it mattered

The help text and the power diagram had both been asserting that "SRV and MSIV
close on air and spring, so they are independent of electrical power." The air
they depend on was not in the model. Same shape of gap as the feedwater case in
§13.13: a claim whose mechanism was missing.

#### What the manual fixes

| | value | source |
|---|---|---|
| Compressors | 3 (2 running + 1 standby), motor driven | 11.6.1 |
| Capacity | ≥1000 scfm at 125 psig each | 11.6.2.1 |
| Control | load at 120 psig, unload at 130 psig | 11.6.3.1 |
| Receiver | 150 psig design; sized to let the standby start and load | 11.6.2.3 |
| Supply | **normal (non-safety) AC** | 11.6.4.2 |

And §11.6.3.2 gives the degradation sequence outright: standby compressor
starts → service air header isolates → **reactor scrams** on loss of air to the
scram pilot valve header.

**NO BASIS**: the pressures at which stages 2 and 3 occur, and the receiver
storage volume. The manual gives the *order*, not the setpoints. Values chosen
so the stated order is what happens, with roughly six minutes from normal
pressure to scram on total compressor loss.

#### Verified consequence, not just ordering

Stage 2 is not decoration. Isolating service air removes 400 of 1200 scfm, and
the measured decay slope changes 1.044 → 0.696 kPa/s — a ratio of exactly 1.50,
matching 1200/800. Shedding the less important header to protect the instrument
header is visible in the numbers.

#### What must survive, and does

If loss of air also removed the depressurization capability it would be a common
cause failure rather than a fail-safe. §10.2.2.2 prevents that: each ADS
safety/relief valve has its own accumulator and check valve, so it can be
opened and held open after the pneumatic supply fails. SRVs are therefore
independent of instrument air in the model. Test A5 drives the plant to zero
air, confirms the MSIV has spring-closed, and then confirms the SRVs still hold
pressure at 7.3 MPa.

#### A name collision silently disabled condenser fouling

`AIR_DESIGN = 150 psig` was added for the receiver. Fifty-one lines above it in
the same file, `AIR_DESIGN = 0.4` already existed — the condenser air partial
pressure. Python overwrote it, and

```python
foul = 1 + AIR_FOULING * max(0.0, self.p_air - AIR_DESIGN)
```

became `max(0, p_air - 1034)`, permanently zero. Non-condensable fouling was
switched off entirely and **every test still passed** — both are floats and the
model keeps running.

The physics fingerprint was the only thing that noticed. Among a block of
`없음 -> ...` lines for genuinely new constants sat one reading
`bop.AIR_DESIGN  0.4 -> 1034.21`. That is the signature of a *pre-existing*
constant changing value, and it was nearly skipped as part of the new block.

`conflict_test` now walks every module-level `ALL_CAPS` assignment in `core/`
and `panel/` and reports any name assigned twice, ignoring the legitimate
refinement idiom (`FUEL_ZONE = np.where(ACTIVE, FUEL_ZONE, 0)`, detected by the
right-hand side referencing the name itself). Confirmed against the real
collision:

```
!! 상수 이름 중복 1 건
   - bwr4_bop.py: AIR_DESIGN (275행, 329행)
```

#### Operator bypass switches were defeating physics

Reported immediately after the air system landed: turning off the auto-scram
switch prevented the loss-of-air scram, while turning off MSIV auto-isolation
did **not** prevent the MSIV from closing at 60 psig.

The MSIV behaviour was right. `rps_enabled` bypasses the protection system's
*logic*; RPS scrams by de-energizing the scram pilot solenoids, which vents the
air holding the scram valves. Losing the instrument air vents the same valves by
the same route. The two paths are independent, so bypassing one cannot block the
other. The check was sitting behind `if not self.rps_enabled: return`.

Two more of the same kind turned up on inspection:

| path | was gated by | why that is wrong |
|---|---|---|
| loss-of-air scram | `rps_enabled` | pilot header venting does not go through RPS logic |
| MSIV on loss of control power | `msiv_auto` | de-energized solenoid is loss of motive power, not a signal |
| EHC valves fail-closed | `auto_pressure` | manual demand still needs power to reach the valve |

All three moved outside their switches. The rule:

> **An "auto" switch turns off logic. It cannot turn off physics.** A spring
> closing a valve whose holding air or power is gone is not bypassable.

`air_test [A7]` enforces it across the 2×2 switch matrix, and was confirmed
against the unfixed code — it fails on exactly the two `rps_enabled = False`
cases.

#### The test was wrong twice before the model was

Both failures in `air_test` [A4] were mine, and both were instructive.

1. The first version ran a counterfactual — the same transient with
   `service_air_isolated` forced False. `_air()` re-evaluates it from pressure
   inside the same step, so the two runs were byte-identical. A counterfactual
   you cannot actually hold is not a control.
2. The second measured the slope before and after isolation, but took the *last*
   pre-isolation sample. Isolation takes effect within the step that trips it,
   so that sample already carried the reduced demand and the ratio came out
   1.00. Taking the *first* sample gives 1.50.

Neither failure was a model defect. Both looked like one.

---

### 13.18 Control rod drive hydraulics (NRC 2.3)

#### Why it mattered

The model had been inserting control rods at a fixed rate whenever `scrammed`
was true. Nothing in the model asked **what pushes them**. Control rods in a BWR
do not fall — they are driven upward from below by water. A scram with no motive
water is not a slower scram in the model; it was simply not represented at all.

This is the same shape of gap as §13.13 (feedwater) and §13.17 (air): the
simulator asserted an outcome whose mechanism was absent.

#### What the manual fixes

| | value | source |
|---|---|---|
| Self-scram pressure | reactor pressure alone drives insertion above **800 psig** | 2.3.2.13 |
| Accumulator | one per drive, charged from the charging water header | 2.3.2.10 |
| Scram discharge volume | receives water displaced from above the drive piston; vent and drain valves close on the scram signal | 2.3.2.11 |
| SDV high level | scram before the volume fills, so insertion is never impeded | 7.3.3.2.7 |

**NO BASIS**: the accumulator charge and bleed time constants, the SDV capacity
expressed in scrams (`SDV_SIZE = 2.0`), the high-level setpoint as a fraction of
that capacity (0.85), the drain time, and the factor by which insertion slows
when there is no motive force (`CRD_SLOW_FACTOR = 4.0`). The manual gives the
mechanism and the 800 psig threshold; it does not give these numbers.

#### The pressure threshold is the whole point

`CRD_SELF_SCRAM_P` is the only setpoint here that comes straight from the
manual, and it is what makes the system interesting: at power, losing every CRD
pump changes nothing, because the reactor is at 7 MPa and pushes its own rods
in. The accumulator only matters **after depressurization**. So the failure
combination is not "loss of AC" — it is "loss of AC *and* depressurized", which
is the state a station blackout reaches after ADS or after long-term cooling.

Test A8 walks exactly that: kill the CRD pumps, isolate, arm ADS, run 9000 s.
Pressure falls to 3.41 MPa, the accumulator bleeds to 0.00, and `scram_force_ok`
goes false. A control case at high pressure with a zeroed accumulator stays
true.

#### The SDV scram must exclude the scram it is caused by

`sdv_level` rises as rods insert, so a scram fills the volume, and a full volume
is itself a scram signal. Written naively the trip re-arms itself forever. The
guard is `rods.max() > 0.1` — once the rods are already in, there is nothing
left to trip for.

#### A deadlock I wrote and had to invert

The first version required a drained SDV before RPS could be reset. But the
drain valves are air operated and **closed by the scram signal**: you cannot
drain while scrammed, and you could not reset without draining. The plant order
is the opposite one:

    rods fully in -> reset RPS (which opens the drain) -> drain -> withdrawal permitted

So `reset_scram()` now opens the drain, and the withdrawal interlock moved to
`rod_block()` (`"배수용기 미배수"` while `sdv_level > 0.05`). A8 runs the full
cycle: scram, fill, reset, verify blocked, drain 400 s, verify released.

#### Confirmed by breaking it

A8 was run against six deliberately broken copies of the model. Each one was
caught, and the reported failure named the defect:

| break | what A8 said |
|---|---|
| `scram_force_ok` always true | 감압되고 축압기도 빠지면 미는 힘이 없다 |
| SDV never fills | 스크램 1 회가 배수용기 1 회분을 채운다 |
| reset does not open the drain | 복구가 배수 밸브를 연다 |
| withdrawal interlock removed | 배수 전에는 인출이 막힌다 |
| SDV high-level scram removed | 설정치를 넘으면 스크램한다 |
| insertion rate penalty removed | 미는 힘이 없으면 4 배 느리다 |

The rate penalty is measured, not asserted: rods reset to 100 % at 3.4 MPa
insert 50.00 % in two seconds with a charged accumulator and 12.50 % without —
a ratio of 4.00.

#### What is still missing

Notch (00–48) positioning, the 137 individual HCUs, drive water header pressure
and cooling flow, and failure of the SDV level instrument itself — which is a
well known scram cause in service and cannot be represented while the volume has
no instrument of its own.

### 13.19 Turbine building cooling water (NRC 11.4 · 11.5)

#### Why it mattered

§13.17 added instrument air and closed with a stated gap: the compressors are
cooled by turbine building closed loop cooling water, that system was not in the
model, and so *loss of cooling stopping the compressors could not be
represented*. §11.6.4.1 states the dependency outright:

> "The Turbine Building Cooling Water System supplies normal cooling for the
> compressors and after coolers."

Without 11.4 and 11.5 the air system had a supply that could only fail by losing
electrical power. With them, the chain runs

    heat sink -> TBSW -> heat exchanger -> TBCLCW -> compressors -> air -> scram

#### What the manual fixes

| | value | source |
|---|---|---|
| TBSW pumps | 3, two running + one standby | 11.4.2.1 · 11.4.3.1 |
| TBSW rating | "800 gpm @ 50 psig", 350 Hp 4160 VAC motor | 11.4.2.1 |
| TBSW suction | shares the screenwell pits with circulating water | 11.4.3.2 |
| TBCLCW pumps | 2 full capacity, one running | 11.5.1 |
| TBCLCW rating | 17.8 kgpm @ 85 psig, 1000 hp 4.16 kV | 11.5.2.1 |
| Surge tank | thermal expansion, makeup inlet, NPSH, **leak detection** | 11.5.2.2 |
| Temperature control | HX bypass valve holds discharge at **95 °F** | 11.5.2.3 |
| Standby start | discharge valve opens **10 s after** the standby starts | 11.5.4.2 |
| Supply | both are normal (non-safety) AC loads | 11.4.3.2 · 11.5.4.3 |

#### CORRECTED — TBSW 800 gpm contradicts its own motor

A pump moving 800 gpm against 50 psi (115 ft) needs 23 hp of hydraulic power,
about 29 hp at the shaft. The manual gives it a **350 Hp** motor in the same
sentence — twelve times oversized, which no pump datasheet looks like. At
**8,000 gpm** the shaft power is 291 hp, which is what a 350 Hp motor is for.

Two independent cross-checks agree:

- **The load it must carry.** TBSW cools the TBCLCW heat exchangers (11.4.1),
  and that loop circulates 17,800 gpm. Two TBSW pumps at 800 gpm would be 1,600
  gpm against 17,800 — the service side temperature rise would be eleven times
  the closed side's. Not a design; an impossibility. At 8,000 gpm each the two
  sides are within 10 % of each other.
- **The same arithmetic on RBSW.** 8,600 gpm with a 450 Hp motor (11.2.2) works
  out consistently, which says the method is sound rather than the reading.

This is the fourth self-inconsistency found in the manual (§12.3 condensate
pumps, §5.16/§5.19 ECCS flows, §13.1 APRM rows). A dropped digit is also the
documented OCR failure mode for this scan. **Read as 8,000 gpm.**

#### NO BASIS

The manual gives no heat load, no exchanger sizing, no compressor trip setpoint,
and no loop inventory. Chosen so the design point is self-consistent and the
transient is slow enough to act on:

| constant | value | why |
|---|---|---|
| `TBCLCW_DUTY_RATED` | 40 MW | puts the loop rise at ~6 K at rated |
| `TBCLCW_DUTY_BASE` | 8 MW | what remains with the turbine stopped |
| `TBCLCW_VOLUME` | 270 m³ | ~3 minute turnover at rated flow |
| `TBCLCW_HX_EFF` | 0.75 | leaves the control valve ~40 % open at rated |
| `AIR_COOL_TRIP_TEMP` | 60 °C | compressor cooling water high temperature |
| `AIR_COOL_TRIP_DELAY` | 60 s | how long a running compressor lasts uncooled |
| `TBCLCW_SURGE_VOL` | 20 m³ | with 8 kg/s makeup, sets the leak that wins |

Flows and motor powers are scaled ×1.35 as everywhere else in chapter 11.

#### Verified consequence, not just plumbing

Loss of all TBSW at rated, measured:

| t | what |
|---|---|
| 0 s | TBSW lost; bypass valve drives to 0 %, exchanger duty falls to zero |
| 779 s | loop reaches 62 °C, compressors trip on cooling water high temperature |
| 1140 s | instrument air reaches 80 psig — **scram**, "계장공기 상실" |

Nineteen minutes end to end. That is the point of modelling it: the operator has
time, and every stage is visible before the reactor moves.

Losing one of the two running TBSW pumps does **not** end in a trip. The
control valve closes fully and the loop settles at **36.5 °C**: the same 40 MW
still leaves, but the service water rise doubles from +7.0 K to **+14.1 K**.
Nowhere near the 60 °C trip — margin was spent, not capability. A
degraded-but-stable state is as important as the failure.

(My first estimate for that equilibrium was 40 °C, written from the arithmetic.
Measuring it gave 36.5 °C, and the docs say what was measured.)

#### One thing deliberately not done

A compressor that is **not running** does not overheat. If the latch were armed
whenever cooling was absent, every loss of offsite power would leave a hidden
latch behind, and restoring power would not restore air until the operator found
a button nobody told them about. That is exactly the defect §5.48 was about.
The trip timer only advances while `air_running > 0`, and
`reset_air_compressors()` refuses while the cooling signal is live and names it.

#### Confirmed by breaking it

`tests/cases_tb.py` runs seven deliberately broken copies against
`air_test [A11]`: no cooling judgment, no high-temperature judgment, latching on
a stopped compressor, no standby valve delay, flow surviving an empty surge
tank, a frozen temperature control valve, and a reset that ignores the signal.

#### What is still missing

Strainers and their automatic backwash (11.4.2.2), the pressure control valve
(11.5.2.3 — nothing in the model is placed in or out of service), the individual
turbine building loads, and the reactor building closed loop (RBCLCW, 11.3),
which still connects equipment straight to service water.

### 13.20 Main steam isolation signals — the list was four of six

#### Why it mattered

§13.17's review of the air system ended by asking a broader question: is
everything an operator needs actually there? Walking the isolation chain
answered it badly. The constant block at the head of the MSIV section listed
all six Table 4.4-1 signals in a comment, `isolation_signal()` implemented
four of them, and §12.5 of this document asserted that

> "Table 4.4-1 lists the isolation *conditions*, **which the code matches**"

Three places, three different claims. The code was the least wrong.

#### A second, independent list

§2.5.3.1 gives the automatic closure conditions again, in the Main Steam
chapter, with no cross-reference to Table 4.4-1:

> 1. Low-Low Reactor Vessel Water Level
> 2. **High Main Steam Line Radiation**
> 3. High Main Steam Line Flow
> 4. **High Steam Line Area Temperature**
> 5. Low Steam Line Pressure with the mode switch in Run

Two independent lists agreeing is the strongest evidence this manual offers —
stronger than any single table. Neither list contains condenser low vacuum or
drywell high pressure, both of which the model was using.

#### What was missing, and what it cost

| signal | manual's stated cause | in the model |
|---|---|---|
| MSL high radiation | Gross Fuel Cladding Failure | **absent** — `clad_damaged` drove a rod block and nothing else |
| Steam tunnel high temperature | Steam Leak in Tunnel | **absent** — and so was the leak |

The tunnel one is the more interesting gap. A leak **outside** primary
containment raises no drywell pressure, so `break_size` cannot represent it and
no signal could detect it. An entire initiating event was unreachable.

#### Steam tunnel — implemented with its cause

Modelled as one lumped volume (§4.9 of `physics.md`) with a leak term gated on
MSIV position. Gating on `msiv` is the whole point: closing the valves stops
the leak, which is what the table's "Reason for Isolation" column says.

    leak 1.0 kg/s -> 130 s to 93.3 °C -> isolation -> leak 0, tunnel 47.7 °C
    leak 0.4 kg/s -> settles at 77.3 °C — alarm, no isolation

**NO BASIS**: setpoints, ventilation UA, and heat capacity. The manual mentions
this signal in exactly two places and gives no number in either. Values chosen
so a real leak trips within two minutes and a small one does not — the
threshold sits near 0.55 kg/s.

#### MSL high radiation — a detector on a state that already existed

The manual names the cause: *Gross Fuel Cladding Failure*. The model has that
state. So the signal reads `clad_damaged` directly; no activity transport, no
source term, no invented physics. This was a deliberate scope decision — the
simulator is not a radiological release model.

Measured honestly: **it never fires first.** In every scenario that reaches
clad damage (SBO, ATWS, isolation without cooling), the low-low level signal
isolated thousands of seconds earlier:

| scenario | first isolation | clad damage |
|---|---|---|
| SBO with no diesels | 18 s (condenser low vacuum) | none in 12,000 s |
| ATWS, RPS defeated | 5041 s (Level 2) | 5166 s — already isolated |
| Isolated, no cooling | 892 s (Level 2) | 5593 s — already isolated |

So it is defence in depth rather than observable behavior. It earns its one
line by making the code match both manual lists, and by making the isolation
unresettable once fuel has failed — which is correct, and which the permanent
rod block was already doing on the other side.

#### Mode switch — the judgment existed twice

Table 4.4-1 gives "Mode Switch in RUN" as the additional requirement for low
steam line pressure; Table 7.3-1 gives "Mode switch not in RUN" as the bypass
for the MSIV closure scram. The model has a real mode switch (`mode_run`, an
operator control on the panel, added with the startup instrumentation). Both
of these were still reading a **10 % power proxy** instead, and the constant's
comment still said "this model has no mode switch".

Switching them to the real signal changed behavior in a way worth recording.
After a scram the old proxy self-bypassed, because power fell below 10 %. The
mode switch does not fall on its own. So a scrammed reactor left in RUN now
re-isolates on low steam line pressure the moment the valves reopen and the
vessel starts to blow down — which is exactly the stated Reason for Isolation,
"Prevent Excessive Reactor Vessel Cooldown Rate". The operator's real action —
take the mode switch out of RUN before cooling down — is now required, and
`button_test` performs it.

#### Drywell high pressure — removed

§4.4.3.3 enumerates the systems that isolate on low water level or high drywell
pressure: drywell equipment and floor drains, torus drain, containment vent and
purge, secondary containment ventilation, and RHR shutdown cooling. **Main steam
is not among them**, and neither MSIV list contains it.

Measured before removing, on three break sizes:

| break | before | after |
|---|---|---|
| 2 % | isolated 5 s, "drywell high pressure" | isolated 50 s, "MSL low pressure" |
| 10 % | 3 s | 15 s |
| 30 % | 3 s | 9 s |

Final water level, pool temperature and drywell pressure all land within a few
tenths. The plant still isolates on every break — just via a signal the manual
actually lists, a few tens of seconds later. No behavioral cliff, so it goes.

#### Condenser low vacuum — kept, and now labelled

This manual puts condenser low vacuum in the **scram** table (7.3-1, bypassed
below 30 % turbine first stage pressure / not in RUN), not in either isolation
list. The model uses it as an isolation signal, and it is load-bearing: it is
what isolates within 18 s of a station blackout. Real BWR/4 plants do carry it
as a Group 1 signal. Kept, and recorded as **DELIBERATE** in the code, the help
text and here — which is what §12.5 should have said in the first place.

#### A second copy of the judgment, still standing

§5.48 extracted `isolation_signal()` so the reset button and the latch would
consult one chain. It missed `active_isolation_signal()` — a near-duplicate the
panel called *before* the reset. The copy did not honour the `msiv_auto`
switch, so with automatic isolation turned off the panel refused a reset that
the model would have granted, naming a signal the model no longer considered
live. Deleted; `open_msiv` now reports what `reset_isolation()` returns.

That is twice now that this exact defect has been found in this one chain. The
rule it keeps violating: **one judgment, one owner, and the display asks the
owner.**

#### Confirmed by breaking it

`tests/cases_isol.py` runs seven mutations against `air_test [A12]`: each of the
two new signals removed, the leak un-gated from MSIV position, tunnel
ventilation removed, both mode-switch judgments reverted to the power proxy,
and drywell high pressure added back. The last one matters — mutations that
*add* a signal are as important as ones that remove it, or the test only ever
proves it can see absence.

The first run caught six of seven. The MSIV-closure-scram revert escaped, and
the fault was in the check, not the case: it closed the valves and looked at the
scram reason **thirty seconds later**, by which time high pressure (or, with the
mode switch down, the IRM) had already claimed the scram and the
`not self.scrammed` guard meant the MSIV judgment never ran at all. The check
was reading a place where both versions of the code give the same answer.

Stepping **once**, with the IRMs bypassed, separates them cleanly:

    not in RUN : "고압력"            <- MSIV is not the reason
    in RUN     : "MSIV 폐쇄 (수동)"

Same family as §5.44's tests that could not fail, but a different mechanism —
there the exit code was missing; here the *instant of measurement* was wrong.

### 13.21 Control rod drop accident and the Rod Worth Minimizer (NRC 7.5)

#### Why it was deferred, and why that changed

§13.20's closing review listed the Rod Worth Minimizer as the most valuable of
the deliberate simplifications, and then argued against adding it: the accident
it guards against was not modelled, so RWM would be *a rule with no reason*.
The correct order was accident first, interlock second. This section does both.

#### Two revisions of the same manual disagree — the newer one wins

R-104B (the copy in `manual/`) has RWM in §6.1 and RSCS in §6.2. R-304B, the
later revision, replaces both with a much more specific §7.5. Where they differ,
R-304B is used, and the differences are not cosmetic:

| | R-104B (§6.1) | R-304B (§7.5) | used |
|---|---|---|---|
| Low power setpoint | above 20 % power, sensed by **steam and feed flow both > 30 %** | **main steam flow < 20 %** enforces; 20–30 % is a *transition zone*, display only; > 30 % display off | R-304B |
| Rod-drop conditions | **four** | **six** — two of them are operator omissions | R-304B |
| Notch rule | "all rods in a group within **one notch**" (this is RSCS Group Notch Control) | RWM has a **±2 notch tolerance** on group limits, for position-sensor failures | R-304B |
| Fuel enthalpy | 280 cal/g, single value | **a ladder**: 170 clad perforation, 200–280 melting (280 = design limit), 425 melting complete | R-304B |

The first draft of this work had the ±1 notch rule as an RWM requirement. It is
not — it belongs to RSCS, which is a separate hardwired backup and is not
modelled. Getting that wrong would have added a rule the manual does not put
there.

#### Web cross-check — three corrections and four numbers

The manual alone was not enough, so published sources were checked:

| item | first draft | after cross-check | source |
|---|---|---|---|
| notches per stroke | 48 | **24** — a notch is 6 in, the 00–48 readout counts 3-in *half* notches, stroke 12 ft | R-104B 2.3.3 (own manual, missed first time) |
| velocity limiter | no number in R-104B | **< 3.11 ft/s** | R-304B 2.2.3.4 |
| 280 cal/g | "the limit" | **maximum radially averaged peak** fuel enthalpy, Reg Guide 1.77 | NRC RIA topical reports |
| RWM required below | 20 % power | modern TS say **10 %**; this manual's design basis is 20 % | NRC TS bases |
| in-sequence rod worth | none | BPWS limits incremental worth to **~1 % Δk/k** | BPWS technical reports |

The notch correction mattered: the ±2 notch tolerance is 8.3 % of stroke with
24 notches and 4.2 % with 48. The manual's own CRD chapter had the answer and
the first pass had not looked there.

#### Drive and blade are different things

`rods` is the blade; `rod_drive` is the drive mechanism. They are the same
until the coupling fails, and the divergence *is* the accident's first
condition. It also produces the fifth condition's tell for free — a withdrawal
that moves no absorber moves no neutrons:

    30 s at cold criticality, power afterwards
      do nothing              0.0203 %
      withdraw uncoupled rod  0.0203 %   <- identical
      withdraw coupled rod    0.0235 %

Routing every withdrawal path through the drive turned up a defect in the
measurement itself: `RodAutoControl` wrote `r.rods` directly, so the first
version of the check read a `rod_drive` array that never moved and reported
zero sequence errors. It was reading a variable nothing wrote. After the fix
the same run reported **45 withdraw errors**.

#### Energy in closed form, not integrated

A prompt excursion is milliseconds; this model's smallest step is 0.2 s. Rather
than build a second time axis, the total energy comes from Fuchs–Nordheim:

    ΔT = 2·(ρ − β) / |α_Doppler|

and is deposited into the post-drop power shape, because §7.5.3.1 says the
spike is short compared with the fuel time constant so the energy stays in the
pellets. Step insertion is **conservative**: the real blade is limited to
3.11 ft/s and takes 3.86 s for full stroke, so the real event is a ramp.

Fuel melting stays owned by the existing `fuel_temp > FUEL_MELT_TEMP` check —
depositing the energy makes that judgment agree with Table 7.5-1 rather than
competing with it. Only cladding perforation (170 cal/g) is judged from
enthalpy. One owner per outcome.

#### The grid correction, without which nothing discriminates

The model has **60 rod channels**; the plant has **137 control rod drives**
(R-104B 2.3.1). One model channel holds 2.3 drives' worth of absorber, so
pulling a whole channel is worth more than the single-rod event the accident
actually is. `CRDA_ROD_SCALE = 60/137 = 0.438`.

Measured before and after, at cold criticality:

| | without correction | with correction |
|---|---|---|
| in-sequence rod | 1401 pcm — **2.19 β, prompt** | 614 pcm — 0.96 β, no excursion |
| out-of-sequence | 3051 pcm — 4.77 β | 1336 pcm — 2.09 β |

Without it *both* cases are prompt supercritical and the interlock has nothing
to discriminate. This is the same class of correction as the ×1.35 plant-size
scaling used throughout chapter 11.

#### What the interlock is worth, measured

At cold criticality (21 °C, 22 of 60 withdrawn, RWM enforcing):

| | worth (one drive) | β | peaking | enthalpy | verdict |
|---|---|---|---|---|---|
| next rod in sequence | 142 pcm | 0.22 | 3.6 | — | within limit |
| skipped central rod | **1336 pcm** | **2.09** | 7.5 | **298 cal/g** | **exceeds design limit** |

A factor of 9.4 in worth. And the danger disappears with power exactly as the
manual explains: the same central rod measures 0.16 β at 2 % power, because
voiding flattens the flux around the blade.

#### The auto-startup had to be taught the rule

`RodAutoControl` picked candidates from a 12-channel window of `ROD_SEQUENCE`,
which straddles group boundaries once early channels finish. Measured: 45
withdraw errors during one cold-to-rated climb. Constraining candidates to the
latched group — which is what the procedure says — gives zero errors, and the
climb still reaches 99 % (9.6 h vs 10.5 h; the pattern differs slightly).

#### Confirmed by breaking it

`tests/cases_rwm.py` runs nine mutations against `tests/rwm_test.py`: the grid
correction removed, Doppler feedback dropped from the excursion, peaking
dropped, the blade following an uncoupled drive, the coupling check made blind,
the sequence rule disabled, the block extended to trap the error rod itself,
the enforcement zone disabled, and the auto-startup released from its group.
All nine are caught.

These checks started life inside `air_test`, which by then carried air, CRD
hydraulics, stuck rods, turbine building cooling, isolation signals and this —
six subjects and 200 s per run. Splitting this subject into its own file cut a
mutation round from ~35 min to **45 s** (5 s per case). A test file that covers
one subject is not only clearer to read; it is the difference between a
verification step you run and one you skip.

#### A one-sided bound let two mutations through

Nine mutations, two escaped: removing the grid correction, and dropping β from
the excursion formula. Both change the *magnitude* of the result, and the check
only asked whether it exceeded 280 cal/g:

    correct code       298 cal/g   exceeds — pass
    no grid correction 1037        exceeds — pass, but wrong
    no Doppler term     575        exceeds — pass, but wrong

Bounding one side catches nothing that scales. The upper bound (425 cal/g,
Table 7.5-1's "melting complete") closes both, and it is physically motivated:
published out-of-sequence CRDA analyses land in the hundreds of cal/g, not the
thousands.

§13.20 found a check reading at the wrong *instant*; the drive/blade work above
found one reading the wrong *variable*; this one read a half-open *range*. Same
disease, three faces.

`conflict_test` also earned its keep: `CRDA_ROD_SCALE` was defined twice (a
`None` placeholder plus the real value), and the duplicate-constant scan built
after the `AIR_DESIGN` collision in §5.40 caught it immediately.

#### The interlock told the operator "no" without telling them "what instead"

Reported after the fact: *the RWM is there, but you cannot see which rod you
are supposed to withdraw.* Correct — the first version displayed the block
reason and nothing else. A real RWM operator panel shows the latched group
number and the coordinates of the error rods; that half was missing.

The core map now blinks the outline of the rods to withdraw next, in yellow,
until the operator selects one. When blocked it points at **the error rod
only** — which is the whole purpose of the block (7.5.2.1.10, "to enforce
correction of existing errors"), and without that pointer a block reads as a
deadlock. Blink phase is wall-clock, not simulated time, so it stays visible at
100× speed.

Fixing it surfaced a name collision that had been doing damage silently. The
panel already had a checkbox labelled "RWM" for something else — a limit of two
rods moving at once below 10 % power, which is about local peaking, not
sequence. Worse, the keylock bypass added in this section bound its variable to
the same attribute name, so the refresh loop was resetting the *old* checkbox
every frame. Nothing visible on screen said so; `gui_test` only checks that the
panel builds and that help keys resolve. Renamed both.

#### What is still missing

RSCS (§6.2) — the hardwired backup with Group Notch Control — is not modelled;
it duplicates the RWM's function and the manual itself calls it a backup.
Sequences A and B (which alternate over a fuel cycle) are not modelled; there
is one sequence. Group limits are all full-stroke, so the two "current group
past its own limit" error types cannot arise — banking to intermediate
positions would be needed for those, and intermediate positions make no
observable difference on a 60-channel grid.

---

### 13.22 Control rod drive water, recirculation demand, natural circulation

Seven items were reported from operating the simulator. Six shared one root
cause: **the display was right, and nothing in the model was making the
judgement behind it.**

#### Control rod drive water — the load list says the opposite of intuition

Control rods do not fall under gravity. Water pushes them from **below** the
core, and the CRD pumps make that water. Nothing in the withdrawal logic asked
whether those pumps were running, so rods could be withdrawn during a station
blackout.

Table 9.2-1 (Shutdown Board Load List) assigns them explicitly:

| Bus | Loads |
|---|---|
| 101 Red | RHR "A" · Core spray "A" · Service water "A" · **CRD water pump "A"** |
| 102 Blue | RHR "B" · Core spray "B" · Service water "B" · **CRD water pump "B"** |
| 103 Orange | RHR "C","D" · Service water "C","D" (no CRD) |

They are **emergency** bus loads. The consequence runs against intuition:

```
LOOP (offsite power only)   diesels carry them -> rods still movable
SBO  (diesels also lost)    no drive water     -> neither withdraw nor insert
Loss of 101 or 102 alone    remaining pump suffices
```

The same root cause had corrupted accumulator charging: the charging test read
**offsite** power (`bop.power_avail`), so accumulators bled down during a LOOP
even while diesels were running the pumps. Now `crd_drive_ok` reads the
emergency buses, and `rod_block()` returns "CRD 구동수 없음 (펌프 정지)".

Verdict: **CORRECTED**. `CRD_PUMP_BUS = (0, 1)` follows Table 9.2-1 directly.

#### RPS alternate power — maintenance spare, not accident spare

§9.3.1.3:

> There is an alternate source of power from a **transformer with its primary
> connected to an emergency 480 VAC bus** to allow for maintenance of a motor
> generator with no loss of reactor protection system function.

Both paths branch from the **same** emergency 480 VAC bus; only what follows
the bus differs (MG set with flywheel, versus transformer). So the alternate
covers an MG failure but not a bus loss — switching to it during a LOOP or SBO
accomplishes nothing. The code was already correct
(`fed = (mg or alt) and bus_ok(...)`); the help text and the power diagram did
not say so. Both now do.

§7.3.2.1's interlock ("an interlock is provided that will not allow both
alternate supply breakers to be closed at the same time") was already modelled.

Which emergency bus the transformer's primary sits on is not stated. The model
uses the same bus as the MG for that channel. Verdict: **NO BASIS**, documented
as such in the help text.

#### Recirculation demand versus actual speed

The operator handles scoop tube position — a **demand**. Actual pump speed is
set by power availability and trips. These were one variable, so raising the
setpoint of a pump that had lost power raised core flow in that same step.

Split into `recirc_demand_a/b` (what the operator sets) and `recirc_a/b` (what
the physics reads). `_recirc(dt)` is the **single writer** of actual speed, in
both directions; every other routine may only move the demand. That rule
replaced an earlier split in which each downward path ran its own coastdown,
which let two stop signals decelerate the same pump twice in one step.

Each side is advanced independently toward a capped demand with a first-order
lag:

```
stopped = rpt_tripped or bop.power_avail <= 0.5
cap     = 0                if stopped
          RECIRC_RUNBACK   if runback_active      (0.45)
          RECIRC_MAX       otherwise
tau     = RPT_COASTDOWN    if stopped or runback_active   (5 s)
          RECIRC_TAU       otherwise                      (5 s)

target        = clip(demand, 0, cap)      and is written back to demand
speed(t+dt)   = speed(t) + (target - speed(t)) * (1 - exp(-dt/tau))
```

Writing the clipped target back to the demand is what makes a raised setpoint
inert while a trip is latched — the operator's number does not survive the
step. The panel stepper reads demand, not speed, so the two are visibly
different exactly when they differ physically. Measured on the real panel: at
80 % settled, stepper 80.0 %; one step after an RPT, stepper 0.0 % while actual
speed is still coasting at 0.655; after the operator raises the setpoint to
100 % mid-trip, stepper 0.0 % and speed 0.536, still falling.

Measured, from a settled 30 % pump speed with demand stepped to 100 %: 18.1 %
of the gap closed at 1 s, **63.2 % at 5 s** (analytic `1-e^-1` = 63.2 %), 95.0 %
at 15 s, 99.8 % at 30 s. Splitting the same 20 s of simulated time into dt of
1.0 / 0.5 / 0.25 s gives 0.987179 in every case — the exponential composes
exactly for a fixed target, so this is a consistency check of the closed form,
not evidence of convergence for a varying one.

Two things the two time constants are **not**. `RECIRC_TAU` and
`RPT_COASTDOWN` are both 5 s, but they are separate constants for separate
phenomena (scoop-tube response versus a tripped motor coasting on inertia) and
neither is a manual figure; 5 s is a modelling choice, and it is the time to
close 63 % of the gap, not the time to arrive. `RECIRC_RATE`, which the
automatic operator uses, is the rate at which **demand** is ramped and is
unrelated to either.

Verdict: **CORRECTED**, with `RECIRC_TAU` / `RPT_COASTDOWN` = 5 s **NO BASIS**.

#### The trip goes to zero, not to minimum speed

§7.2.3.2 describes RPT as opening the pump motor breakers. A tripped pump
therefore coasts to **zero**, not to the 30 % minimum *operating* speed of
§7.2.3.1.2 — those are different quantities that had been conflated.
`_recirc_trip` now only latches the signal and zeroes the demand; `_recirc`
performs the deceleration. Measured from 80 %: 29.4 % at 5 s, 4.0 % at 15 s,
5e-6 at 60 s.

`reset_rpt()` re-checks `rpt_signal()` and refuses to clear the latch while the
originating condition persists; on success it leaves both demands at zero, so
restarting requires a fresh operator setpoint. `_ac_power` behaves the same
way on a loss of AC — it zeroes demand only, and the pump does not restart by
itself. Measured: after 120 s of station blackout and 180 s of restored power,
demand and speed both remain 0.

#### Snapshot migration

`_RestoreDefaults.__setstate__` fills attributes absent from an old snapshot
from a fresh instance, which gave pre-split snapshots `RECIRC_MIN` demand and
dropped a rated snapshot from 98.6 % to 2.7 % power. `__setstate__` now seeds
each demand from the stored actual speed. Verified by stripping
`recirc_demand_a/b` from a state captured at 77 % and restoring: demand comes
back as 0.7698, equal to the stored speed, and the restore prints the
substitution rather than making it silently.

#### Natural circulation is made by voids

The reported doubt was right. A constant 0.30 meant 30 % core flow in a cold
shutdown with zero voids — flow with no driving head. §7.2.3.1.1 says the
opposite:

> core flow increases due to the change in moderator density and **steam
> formation (voids)** within the core region

Buoyancy-driven flow goes as sqrt(driving head), and head goes as core-average
void:

```
natural_circ = 0.30 x sqrt(void / 42 %)      clamped to [0.03, 0.30]
```

The 42 % reference is the **equilibrium** (5-day) core's measured 42.9 %; a
fresh core reads 31.8 %, which would have mis-set the constant. The 0.03 floor
is single-phase thermal circulation and is **NO BASIS** — it does not decide
anything, because RHR shutdown cooling provides forced flow in that regime.

Measured core flow after a station blackout: 15 % at 60 s, 12 % at 5 min, 8 %
at 30 min, 3 % at 2 h. Three help entries asserted "natural circulation 30 %
remains" — wrong by 2× to 10×. Corrected.

Verdict: **CORRECTED**.

#### Automatic withdrawal bypassed every rod block

`RodAutoControl._move_rods` wrote `rod_drive` directly and never called
`rod_block()`. With drive water cut — manual withdrawal blocked — automatic
withdrawal still moved rods **+0.75 %p in 60 s** (measured). It now nulls the
RBM on selection, as an operator does, and consults the full block list. The
low-power gang limit was also aligned to `ROD_GROUP_WIDTH` (4 channels); manual
had been limited to 2, which made the operator worse off than the automation
running the same procedure.

Two block classes behave differently, and the model now distinguishes them:

```
no drive water    physical — blocks withdrawal AND insertion
every other block a withdrawal block — insertion still permitted
```

Verdict: **CORRECTED**.

#### A deadlock that only appeared once the blocks were honoured

With blocks honoured, startup to a 40 % target stalled at 0.75 % power for 14
simulated hours. **SRM detector auto-withdrawal tracked the mean count rate
while the rod block tracks the maximum.** Once channels spread, the maximum
pinned at 1e5 (block asserted) while the mean sat below it (no withdrawal) —
and the block never cleared. While the automation bypassed blocks, nothing
could reveal this.

§5.2.3.3 asks for the band to be held per channel:

> the SRM detectors are incrementally withdrawn to maintain SRM count rate
> between 10² and 10⁵ cps

With a single modelled detector position, the withdrawal direction must track
the maximum, matching what the block reads. After the fix, targets of 100 / 40
/ 10 % all reach their setpoint (9.52 / 7.62 / 7.04 h, heat-up limit honoured).

Verdict: **CORRECTED**.

#### Help text audit

Numeric claims were checked by measurement against the equilibrium core, not
by reading:

| Claim | Measured | Verdict |
|---|---|---|
| core-average void "about 40 %" | 42.9 % | MATCH |
| exit quality "about 14 %" | 13.6 % | MATCH |
| fuel volume-average "about 700 °C" | 694 °C | MATCH |
| "natural circulation 30 % remains" | 15 % → 3 % | **CORRECTED** |
| Bus load list | **CRD water pump missing** | **CORRECTED** |

Coverage was measured too: 59 help keys, 59 referenced by `!` buttons, matching
in both directions; all 214 operator controls have help reachable from an
ancestor frame. A new `tests/help_test.py` enforces this, plus the presence and
placement of the one-line summary added to every entry, the 70-column display
width, and that every f-string substitution actually resolved.

That last check earned its keep immediately: the natural-circulation constants
were used in the help text without being imported, so `panel/help_text.py`
would not import at all — an f-string module takes the whole panel down with
one missing name.

#### What is still missing

The CRD pump count is modelled as an availability flag, not as individual pump
and header behaviour; drive water pressure, cooling water flow, and the
charging water header are not represented (§5.40's note still stands). The
alternate RPS supply is modelled as a permissive, not as a breaker the operator
can rack in. `NATURAL_CIRC_MIN` has no manual basis.


## 2026-09-13 spatial xenon, flow dynamics and UI revision

CHANGES 5.57 replaces the scalar I/Xe model with 720 active nodal inventories,
ties in-range dryout to the same local critical-quality criterion as CPR, and
corrects RPT coastdown to zero. Normal pump response is a 5 s first-order model;
the response time and post-CHF transition are explicit educational assumptions.
825 psig MSIV isolation is compared at 5.7894998 MPa absolute, in RUN mode.
Manual APRM/RBM bypass does not bypass automatic withdrawal or RPS.

Historical scalar-xenon and pump-minimum results above are not current validation.
Current commands, source provenance, scenario conditions, thermal margins and
remaining limitations are in [SPATIAL_FLOW_REPORT.md](tests/SPATIAL_FLOW_REPORT.md).
No direct scram contact has been invented for calculated thermal-limit ratios.

---

### 13.23 Fourteen reports from operating the simulator

The user operated the panel and reported fourteen items. Eleven were defects,
one was correct behaviour misread as a defect, and two were requests. Two
standing rules came out of the session: **run only the tests the change needs**,
and **do not verify against the manual alone** — the second came from the user
correcting my reading of the onsite power arrangement with outside research.

#### Vessel level used an instrument slope as if it were geometry

`VESSEL_AREA` (30 m²) was calibrated as the **level-instrument sensitivity near
normal level**, then extrapolated uniformly all the way down. The rated
inventory (218 m³) spread over 30 m² is only 7.27 m of column, but BAF sits
8.79 m below normal level — so **an empty vessel could not reach BAF**. It
stopped at −723.9 cm, reporting 59 % uncovery while the model simultaneously
reported the vessel depleted and steam generation stopped.

Below TAF the level now interpolates from TAF to BAF on remaining inventory:
the core region holds less water per metre because fuel, shroud and internals
occupy it. Measured: 1,000 kg → −878.8 cm → **100.0 %** uncovered; 20,000 kg →
72.0 %; 80,000 kg → 0 %. Monotone throughout. Every level setpoint (L1–L8) lies
**above** TAF, so scram, ECCS initiation and shrink/swell are untouched.

Verdict: **CORRECTED**. The sub-TAF interpolation has no manual basis and is
documented as a geometric-consistency device, not a hydraulic model.

#### Steam outflow was not limited by steam inventory

`steam_mass = max(1.0, …)` silently manufactured steam: the floor stopped the
inventory going negative but nothing stopped the outflow being computed. With
the vessel empty and MSIV held open, a main steam line leak kept flowing and
heated the steam tunnel to 96.4 °C — past the 93.3 °C isolation setpoint.

Outflow is now capped by the remaining inventory above its floor plus the
current boil rate and, when the cap binds, every path is scaled by the same
factor. The leak control was also changed from an absolute kg/s to an
**equivalent break fraction of rated steam flow**, matching the drywell
`break_size` — the absolute form was the root of the problem, being independent
of both pressure and inventory.

Control case (rated, steam present): 0.1 % break isolates in 58 s, 0.5 % in
11 s, 2 % in 4 s. With no steam, a 100 % setting produces 0.00 kg/s.

Verdict: **CORRECTED**.

#### RWM: the tolerance was right, the warning was missing

A rod outside the latched group could be withdrawn 8.33 % with no block. That
8.33 % is ±2 notches, which §7.5.2.1.12 explicitly allows as the limit
tolerance. What was missing is §6.1.2.1.6:

> A select error occurs whenever the operator **selects** a rod other than one
> contained in the current rod group. The select error provides the operator
> with **warning** that he has selected a rod, which if moved, will create an
> insert or withdrawal error.

A warning, not a block — and it did not exist, so the screen said nothing while
the operator created an error. `rwm_select_error()` now reports rods selected
from a **later** group (earlier groups stay selectable, since withdrawing them
corrects an insert error), and the panel warns on selection and in the RWM
status field. The post-error blocks are unchanged.

Verdict: **CORRECTED**.

#### Manual relief-mode SRV operation did not exist

SRVs opened only on setpoint or ADS. §2.5.2:

> The solenoids are energized by **switches located in the control room**. This
> type of arrangement provides the control room operator with **a means to
> operate any of the 11 safety/relief valves.**

Without it there is no way to depressurise after a LOOP closes the MSIVs on low
condenser vacuum — exactly the case reported. `srv_manual` now opens valves from
the top of the list (not overlapping the ADS valves), gated on **control power**
because the actuator is solenoid-operated.

Measured from rated + LOOP + MSIV closed: with no manual opening, 7.37 MPa at
300 s and still 6.31 MPa at 1800 s. With three valves opened manually, 0.64 MPa
at 300 s and 0.25 MPa at 1800 s.

Verdict: **CORRECTED**. The ADS/SRV counts (13 total, 6 ADS) were re-examined
and **confirmed against the plant's own licensing basis** — see below.

#### Onsite power: the generator was missing as a source

Reported: a turbine trip leaves the normal source in service. The manual reading
alone was ambiguous; the user supplied the real arrangement and outside sources
confirmed it — the main generator feeds the grid through the main transformer
and **taps off that path to feed the unit's own NSST (main bus)**; when the
generator is down the grid back-feeds the same path; only when the NSST path
itself is lost does the RSST (startup bus) pick up.

§9.1.2.2 is consistent: turbine and generator trips open the generator breakers
"without interrupting the incoming power to the unit station service
transformers". So a turbine trip correctly does **not** transfer to RSST. What
was missing was the distinction on screen. `main_bus_source` now reports
generator-fed versus grid-backfed, shown in the panel and the power diagram.

The existing 5-cycle NSST to RSST fast transfer matches §9.2.3.3 and sits inside
the 4–9 cycle range reported as typical for nuclear plants.

Verdict: **CORRECTED** (display and model structure); the transfer logic was
already right.

#### Manual rod speed was slower than the plant's slowest mode

R-304B §7.1.3.1.1 gives the notch sequence in seconds: about 1 s insert to
unload the collet, **about 1.5 s withdraw** covering one 6-inch notch, 6 s
settle — the "approximately ten seconds ... to move a rod one notch" of the
text. Continuous withdrawal (§7.1.3.1.1.3) holds the timing circuit in the
withdraw state, giving 6 in / 1.5 s = **4 in/s**.

Full stroke is 144 in, so notch stepping is 240 s and continuous is 36 s. The
model's speeds were 2000 / 833 / **400 s** — its fastest setting was slower than
the plant's slowest mode. Now Slow = notch stepping, Fast = continuous, Medium
the midpoint (**NO BASIS**, a simulator convenience). Measured on the real panel:
239.8 s and 36.0 s.

Verdict: **CORRECTED**.

#### The mechanical vacuum pump ran without power

With MSIVs shut, the hogging vacuum pump took over maintaining vacuum — and
nothing checked whether it had power. The vacuum stayed reported as healthy
through 1800 s of station blackout. The condenser pressure did rise to
atmospheric, but only because circulating water was lost; the mechanism was
wrong and off-gas flow was still being computed.

Outside research confirms the pump is a motor-driven axial compressor listed
among power-conversion components, not an emergency load, and that it **trips on
main steam high-high radiation** with its discharge valve closed. Both are now
modelled. Measured: blackout gives no vacuum and off-gas 0.000; clad damage with
power intact trips the pump.

Verdict: **CORRECTED**.

#### Containment heat removal — not a defect

Reported as possibly only working in certain situations. Measured across bus
combinations (all three live; 101 or 102 lost; 103 lost) and every result was
defensible; suppression pool cooling requires the operator to line up loops,
which is correct plant practice. What was missing was the **reason** when it is
not running. `pool_cooling_reason()` now owns that judgement in the core and the
panel asks it, rather than each re-deriving the condition: nothing selected,
service water lost, buses lost, shutdown cooling holding a loop, or the pool
already colder than the service water.

Verdict: **MATCH** (behaviour), with a display addition.

#### Help text

Forty-one detail entries were rewritten to carry setpoints, conditions and
boundaries rather than restating the summary. All numbers are interpolated from
code constants; hand-typed figures go stale silently. Writing them surfaced one
wrong value — the instrument-air scram setpoint was rendering as 1 kPa instead
of 552 kPa because an already-kPa constant was divided by 1000 again.

#### Test-quality notes

The popup Escape check was **flaky**: it passed one run and failed the next,
always on a consecutive stretch of keys, because key events queued before a
window is mapped are discarded. Sending with `when="now"` drives the binding
chain directly; five consecutive runs pass. The production code now also binds
Escape on the window, the text body and the close button, and the body no longer
takes focus.

#### ADS valve count — resolved against the licensing basis

The manual contradicts itself: §2.5.4.11 says the ADS "uses **six of the
thirteen** safety/relief valves", while §2.5.2.1 and chapter 10.2 both say
"**seven of the eleven**". Weight of text favours 11/7; the model uses 13/6.

Browns Ferry's Improved Technical Specifications (NRC ML061140182) settle it:

> LCO 3.5.1 — "Each ECCS injection/spray subsystem and the Automatic
> Depressurization System (ADS) function of **six safety/relief valves** shall
> be OPERABLE."
>
> LCO 3.4.3 — "The safety function of **12 S/RVs** shall be OPERABLE."

Twelve required operable is the usual one-out-of-service allowance on thirteen
installed. Table 1.8-1 identifies this model's 3293 MWt BWR/4 as Browns Ferry,
so **13 installed and 6 ADS is correct** and the manual's 11/7 belongs to a
different, smaller reference plant — the same situation as the ECCS flows in
§5.19.

The general lesson is the one the manual's own preface implies: it is "not
necessarily specific to any particular nuclear power plant". A count that
differs between chapters cannot be resolved inside the manual; it needs the
plant's licensing documents. Verdict: **MATCH**, basis upgraded from inference
to citation.

#### What is still open

`NATURAL_CIRC_MIN` and the sub-TAF level interpolation have no manual basis.
`RECIRC_TAU` and `RPT_COASTDOWN` remain modelling choices at 5 s.

---

### 13.24 A different core at every start — bundle loading scatter

Reported: running the automatic startup always leaves **the same region not
fully withdrawn**. Requested: draw each channel's fuel loading at random when
the simulation starts, within a range that still satisfies the peaking
conditions.

#### What the model had

`FUEL_ZONE` is a single (8,8) array back-solved once from the equilibrium power
distribution. It never changes, so the core is identical on every run, and the
automatic controller — which picks the **coolest channels first** inside the
current rod group — makes the same choices every time.

A real plant is not like that. Every reload mixes fresh, once-burnt and
twice-burnt bundles into a pattern that differs from cycle to cycle. The model
has no loading map, so the scatter stands in for one.

```
fuel_load[x,y] ~ U(-200, +200) pcm     active channels only, drawn from a seed
rho += (FUEL_ZONE + fuel_load)[:, :, None]
```

`Reactor(fuel_seed=None)` — the default — draws **nothing**. Tests and
snapshots must see the same core every time, so the nominal core stays the
default and only the two human entry points (the panel, `run_core.py`) draw a
seed. The panel prints it in the window title so a core can be replayed.

#### The peaking condition is the contract, not the width

200 pcm has **no basis**. What is defensible is the acceptance test the draw
must pass, and the measurements behind it.

*Scale.* `FUEL_ZONE` already carries a large asymmetry of its own: subtracting
its 8-fold symmetrised form leaves **rms 2042 pcm, max 6512 pcm**. The new
scatter is an order of magnitude smaller than the wobble that is already there.

*Gate.* Each draw is measured and redrawn if it falls outside

```
1.00 <= loading_peak(load) / loading_peak(0) <= 1.05
```

`loading_peak` solves the static shape at cold, all rods out, no feedback —
conditions where node reactivity is exactly `EXCESS_REACTIVITY + FUEL_ZONE +
load`, so it can be solved without building a reactor (19 ms; a seeded
Reactor costs 59 ms in total, and an unseeded one costs nothing). Both sides of
the ratio are measured with the **current** physics, so the threshold cannot go
stale. Diffusion lives in one place (`diffuse_field`) that both the gauge and
`Reactor._diffuse` call; a contract check measures the gauge against a live
reactor solved at the same conditions and they agree to **2e-5**.

Mapping the gauge onto the quantity that matters, over 360 samples at
S = 100/200/300 pcm:

```
total peaking / nominal = 1.1415 * (gauge ratio - 1) + 1.0    r = 0.983, |resid| <= 0.29 %
```

So the 1.05 ceiling corresponds to a total peaking of **2.35**, comfortably
inside the 8x8 design band (BWR/6 2.21 – BWR/5 2.51, §4.3.3.1). It never binds
at 200 pcm; the floor always does.

The floor is 1.00 — *no flatter than the nominal core*. The model's equilibrium
total peaking, 2.2193, sits **0.4 % above the band floor of 2.21**, so "stay
inside the band" and "do not flatten the core" are the same condition here.
Ungated, a 200 pcm draw spreads equilibrium total peaking over
**2.1868 – 2.2428**, and roughly half of that falls below the band. Because the
floor is the binding side, accepted cores are always at or slightly **peakier**
than nominal — the limit-facing direction, not the safe one, which is why the
thermal limits were measured rather than assumed.

#### Measured: five automatic startups, cold to 99 %

| | nominal | seed 1 | seed 2 | seed 3 | seed 4 |
|---|---|---|---|---|---|
| rods fully out | 40 | 40 | 40 | 40 | 40 |
| rods partly in | 20 | 20 | 20 | 20 | 20 |
| total peaking | 2.4447 | 2.4471 | 2.4478 | 2.4380 | 2.4564 |
| MFLPD | 1.0108 | 1.0145 | 1.0164 | 1.0108 | 1.0171 |
| MAPRAT | 1.0703 | 1.0742 | 1.0761 | 1.0702 | 1.0769 |
| MFLCPR | 0.8823 | 0.8862 | 0.8850 | 0.8827 | 0.8866 |

Every ratio moves by less than 0.7 % of the nominal value, and seed 3 is
slightly better than nominal. The absolute MFLPD/MAPRAT excursion above 1.0 at
this state is **pre-existing** — this is one hour past the climb, with xenon far
from equilibrium; at equilibrium the nominal core reads MFLPD 0.909 and MAPRAT
0.963. The scatter does not create it and barely moves it.

Verdict: **CORRECTED** (the loading now differs per run), with the peaking
conditions enforced per draw rather than hoped for.

#### The region that stays inserted did not move — and the fuel was never why

Individual rod depths did change, by up to 3 %p, and so did which channel runs
hottest. But the **set** of partly-withdrawn rods was identical in all five
runs. Raising the scatter fivefold, to 1000 pcm, did not move it either: across
three more startups the twelve deeply inserted rods were the same every time,
and only the shallow leftovers near 92-98 % gained or lost a single rod (39, 40
and 41 fully out). Total peaking drifted *down* to 2.4262-2.4466 — a wider
scatter flattens the core rather than varying the pattern, so there is nothing
to gain there either.

The twelve deeply inserted rods are exactly `RWM_GROUPS[4]`. The automatic
controller follows the plant procedure of **finishing the current rod group
before starting the next** (R-304B 7.5.2.1.2), so whatever is unfinished when
the core reaches rated is always the *last group* — and group membership is
fixed geometry, not fuel. Loading scatter can only reorder and re-depth rods
**within** a group.

In a real plant the region does move between cycles, because the **BPWS
sequence itself is changed** (the A and B sequences). That is the actual cause,
and it was given to the model next — see §13.25, where the second sequence
moves the leftover region wholesale.

---

### 13.25 Two withdrawal sequences — and what the old calibration was hiding

§13.24 recorded that the region left inserted at rated is the last RWM group,
fixed geometry, and that loading scatter cannot move it. The follow-up request
was to give the model the second sequence.

#### The manual already had it

R-104B §6.2.2.1, and the same section in the newer NRC Advanced Technology
Manual (§6.6.4.1, Rev 1210):

> The rods are divided into **two rod groups** which are compatible with the
> Rod Worth Minimizer rod groups. From an all rods full in condition, **the
> operator may choose either of the two groups to begin movement.** Once the
> operator begins to withdraw the first rod in that group, the logic will not
> allow selection of any rods but those in the chosen group, until all rods in
> that group are moved to the full out position.

What the two groups *are* comes from the RWM side (§6.1.2.1.1): withdrawing one
of them full out leaves the withdrawn rods in a **checker board (black and
white) pattern**. So the two groups are the two checkerboard colours — and the
model's sort key was already `(x+y) % 2`. The existing order is the **A**
sequence and the other colour first is **B**. Nothing was invented.

```
A last group  41 63 14 36 52 25 32 23 54 45 43 34
B last group  31 13 64 46 22 55 42 53 24 35 33 44     overlap: none
```

Real BPWS lets the operator start with any of groups 1–4 and pairs the second
group to it — the A1/A2/B1/B2 designations. With 60 channels the two
checkerboard colours *are* the two groups, so two sequences is the reduction.

#### Where the sequence lives

`rod_sequence_name` is an attribute of the reactor; `rod_sequence` and
`rwm_groups` derive from it. The module constants `ROD_SEQUENCE` and
`RWM_GROUPS` stay pinned to **A**, and `Reactor()` defaults to A, so tests and
snapshots always see the same startup.

`select_rod_sequence()` refuses once any drive has moved past `RWM_TOLERANCE`,
which is the manual's "Once the operator begins to withdraw the first rod …
will not allow". Switching clears the latched group number, since group numbers
belong to a sequence. The panel exposes it as two buttons and the startup seed
picks one, so a single 8-digit code reproduces both the loading and the
sequence.

Verdict: **CORRECTED** — the leftover region now moves, which is what §13.24
said the loading scatter could not do.

#### B is the harder core, and the reason is in `FUEL_ZONE`

Same loading, cold to rated automatically, then 30 h to equilibrium:

| | A | B |
|---|---|---|
| total peaking | 2.2967 | 2.3031 |
| radial peaking | 1.3659 | **1.5005** |
| MFLPD | 0.9444 | 0.9533 |
| MAPRAT | 0.9999 | 1.0093 |
| MFLCPR | 1.0102 | **1.1033** |

Total peaking is nearly identical; radial peaking and MFLCPR are not. Fitting
`FUEL_ZONE` with a smooth radial function (quadratic in distance²) and looking
at the residuals shows why:

```
(3,4) +7245    (4,5) +7087    (4,3) +6195    (5,4) +5987  pcm
```

Those four are exactly the rods sequence A banks at rated. A's four deepest
average **+6628 pcm** above the radial fit; B's four deepest average
**−3384 pcm**. The array documented as "radial fuel zoning" also carries a
**rod-shadow correction for the A pattern** — an unavoidable consequence of
back-solving it from the A equilibrium, and invisible while only one sequence
existed.

#### The hour after reaching rated is the hottest — measured

Taking arrival at 99.5 % as t = 0 and sampling hourly for 30 h:

```
   h |   total peak    |      MFLPD      |     MAPRAT      |     MFLCPR
     |    A       B    |    A       B    |    A       B    |    A       B
   0 |  2.490   2.815  |  1.038   1.174  |  1.099   1.243  |  0.899   0.916
   2 |  2.377   2.808  |  0.987   1.149  |  1.045   1.216  |  0.872   0.910
   4 |  2.313   2.672  |  0.948   1.091  |  1.004   1.155  |  0.849   0.931
   6 |  2.195   2.522  |  0.904   1.037  |  0.957   1.097  |  0.841   0.952
  12 |  2.014   2.240  |  0.823   0.927  |  0.871   0.982  |  0.842   0.982
  20 |  2.243   2.271  |  0.926   0.931  |  0.980   0.986  |  0.958   1.033
  28 |  2.314   2.343  |  0.960   0.967  |  1.016   1.024  |  1.019   1.098
```

**Both** sequences exceed MAPRAT 1.0 immediately after reaching rated — A at
1.099, B at 1.243. Xenon builds, the shape flattens, and A drops below 1.0 in
about **4 h** while B takes about **12 h**. The flattest point is near 12 h;
after that xenon redistribution and further withdrawal push both back up.
Neither scrams.

So B is distinctly hotter for the first half day at rated and converges to
nearly the same total peaking at equilibrium, while MFLCPR stays apart
(1.010 vs 1.103). The excursion right after the climb is pre-existing — A
does it too.

#### A B-specific zone was attempted and rejected

Eight proportional iterations (cold-to-rated plus 30 h each, ~7 min per
iteration) targeting "the same equilibrium channel power as A". Thermal ratios
improved (total 2.03, MAPRAT 0.89) but the error against the target **diverged**
from 0.035 to 0.20 rms. A solution that helps but does not converge is not a
calibration constant; and physically the fuel does not know which sequence the
operator will pick at startup. Left at **one loading, two sequences**.

The residual measurement above is the durable result: `FUEL_ZONE` is not a pure
radial zone, and now that is written down with numbers instead of being implied
by "back-calculated from the real power distribution".

Verdict: **MATCH** for the sequence mechanism; the peaking difference is a
**documented limitation**, not a defect. B's MCPR is 1.305 against A's 1.425,
both above the 1.07 safety limit.

#### What is still open

A `FUEL_ZONE` that is a radial zone and nothing else, with the rod-shadow term
either removed or carried separately. Removing it would move A's calibration
(equilibrium total peaking 2.2193, the `REFLECT_Z` anchor in §4.3.3.1, every
fixture), so it is not a small change and was not attempted here.

---

### 13.26 Electrical audit — the motors that drew nothing

Three reports: the steam tunnel readout, the 125 VDC batteries, and a request
to re-verify the electrical system system by system.

#### The tunnel said "normal" because flow, not temperature, chose the word

```python
text = "%.0f℃ %+.1f kg/s" % (t, flow) if flow > 0.005 else "%.0f℃ 정상" % t
```

A high-temperature isolation closes the MSIVs, and the leak is multiplied by
`msiv`, so flow goes to zero — and the word flipped to "normal" with the tunnel
at 138 °C. Only the colour still said otherwise. The word now tracks
temperature (normal / high alarm / isolation setpoint). Reproduced and
re-measured: a 2 % equivalent break isolates at 107 °C and the cell reads
`138℃ 격리설정` in red, returning to `42℃ 정상` ten minutes later.

Verdict: **CORRECTED**.

#### There are two control powers, not one

Both battery reports share a root cause. §9.4.1 names the fourth division:

> "The fourth division consists of four independent systems to provide power
> for non safety related systems. **Two of the systems supply control and
> protective equipment**, and auxiliaries required to achieve safe shutdown
> when offsite power is lost."

The model had one `control_power` flag covering MSIV isolation solenoids, ADS,
the SRV relief mode and the EHC. So division D alone read as "control power
lost", and — in the other direction — the secondary side never looked at DC at
all (`grep dc_` in `bwr4_bop.py` matched only the drain-cooler temperature), so
flat batteries left the turbine running at 1131 MWe.

| | feeds | source |
|---|---|---|
| `control_power` (safety) | MSIV isolation solenoids, ADS, SRV relief | emergency AC, or ESF batteries A1/B1/C1 |
| `turbine_dc_ok` (non-safety) | main turbine control and protection | normal AC, or **division D** |

The main turbine trip solenoids are normally energised and de-energise to trip,
the same fail-safe arrangement as the RPS scram solenoids (§7.3), so losing
that division trips the turbine. Measured:

```
all DC 0 %     turbine False, reason "직류 제어전원 상실", gross 0
division D only  control_power False, turbine_dc_ok True
healthy          turbine True, gross 1131          (unchanged)
```

Bus health also follows the manual now: §9.4.1's "each distribution bus has two
sources ... a solid state battery charger ... [and] a 125 VDC battery" means a
fed, healthy charger holds the bus even with a flat battery. `dc_bus_ok(d)`
owns that judgement.

Verdict: **CORRECTED** (both reports).

#### Two ledgers were counting two different plants

Stopping each machine and measuring `aux_mw` gave zero for the air
compressors, the mechanical vacuum pump, RWCU, the CRD pumps, core spray and
RHR. The emergency-bus ledger was no better: CRD pumps, SLC pumps and RHR
running in shutdown-cooling or pool-cooling mode all moved `bus_load_kw` by 0.

The cause was structural: `bop.aux_mw` counted only turbine-building motors,
`bus_load_kw` only emergency-bus motors, `net_mwe` subtracted only the first,
and the two sets never intersected. Running the whole ECCS changed the station
load by nothing.

Every missing motor is now counted, and the ratings are **derived from the
capacities the manual states** rather than typed in — a hand-typed kW does not
follow when the capacity is corrected (the `AUX_BOOSTER_EACH` convention):

| machine | manual | value |
|---|---|---|
| CRD pump | 2.3.2.1 "200 gpm at 1600 psig ... **motor driven**" | 198.9 kW |
| SLC pump | model 39 gpm against rated pressure | 23.0 kW |
| RWCU pumps | 2.8.2.2 "**motor driven** ... 1 % of rated feedwater flow" | 57.0 kW (pair) |
| air compressor | 11.6.2.1 "**electric motor** ... 1000 scfm at 125 psig" | 0.160 MW |
| mechanical vacuum pump | 8.1.2.6 "two **motor driven**" — no rating | 0.200 MW (**NO BASIS**) |

RHR pumps are the same motors in shutdown-cooling, pool-cooling and spray modes
and are now counted there too. RWCU was wrong twice over — no load and **no
power dependency at all**, so letdown continued through a station blackout;
§2.8.2.2's pumps are motor driven.

#### The audit also found consumption that did not exist

- `bus_load_kw` answers *what is connected to this bus*, because a diesel has to
  know the load it will pick up before it energises the bus. Summing it into the
  consumption ledger drew 1.756 MW during a station blackout. Consumption now
  sums only energised buses.
- Recirculation was charged on speed cubed alone, so a pump coasting down after
  its breaker opened still drew power. Gated on the power fraction.

```
rated  aux 34.470 MW   net 1096.7 MWe   (3.05 % of gross; was 2.8 %)
LOOP   aux  2.056 MW   exactly the emergency-bus load the diesels carry
SBO    aux  0.000 MW
```

Worst-case bus load is now 3526 kW against the 3850 kW overload threshold —
324 kW of margin, and the integration profile (including the diesel-overload
paths in `power_test` and `atws_test`) passes unchanged.

Verdict: **CORRECTED**.

#### The feedwater pumps were not a defect

They were named in the report but drawing no electricity is correct: a BWR/4's
reactor feed pumps are main-steam turbine driven (§2.6.2.7), which the model
already represents with `rfp_steam_ok`, and their shaft power `pump_mw` is
subtracted from main-turbine shaft work. That is why they coast down when the
MSIVs shut.

Setting the pump count to zero *appears* to drop station load by 3.24 MW, but
that is the recirculation runback it causes (1.000 → 0.900 of 12 MW, cubed).
An indirect probe would have confirmed a defect that is not there, so the
control is written into the contract checks.

Verdict: **MATCH**.

#### What is still missing

**RBCLCW** — the reactor building closed cooling water system (§11.3) does not
exist in the model at all; only the turbine-building loop (§11.5) does. The
manual gives its circulating pumps as 100 Hp and the recirculation pump M/G
fluid-coupling cooler pumps as 75 Hp. This is an absent system rather than an
unbilled load, and was left alone.

---

### 13.27 A five-day rated plant from the panel

Requested: a plant already at rated after five days of operation, because
climbing from cold and settling takes too long to reach by hand — as a button
if possible, so it can be kept correct as the model changes.

A button, not a saved file. A stored state goes stale the moment the physics
changes, and the staleness is invisible on screen; a button rebuilds with the
physics that is there now.

#### One recipe, two callers

The test suite already had this state as the `aged` fixture. The panel had no
path to it. Rather than write a second recipe, `core.RatedRun` owns it and both
call it:

```
climb   automatic withdrawal to 99.5 % power, 1 s steps,
        peak limit bypassed below 10 %
hold    5 days at 20 s steps
```

Verified **bit-for-bit** against the fixture: power, rod average, pressure,
level, xenon, exposure and net MWe all identical, and the maximum difference in
the rod array and the power shape is exactly zero.

#### Keeping the panel responsive

tkinter is single threaded, so `RatedRun.advance(budget)` runs only until a
wall-clock budget (0.06 s) is spent and returns; the panel repeats it through
`root.after` and draws the phase and progress on the button. Fixtures give the
same function an infinite budget. A contract check measures that the chunking
does not change the answer.

#### Caching, and why the key contains the source

First press **164 s**, subsequent presses **0.04 s** — the round trip matters
because the point of the feature is to scram the plant, look, and come back.

The cache key hashes `core/*.py`, so one changed line invalidates it. Stored
rod positions and power shapes are the equilibrium of the physics that produced
them; serving them under different physics is the exact failure `.gitignore`
already guards against by excluding `*.pkl`. Entries are per core (fuel seed and
rod sequence).

Verdict: **CORRECTED** (a capability the panel lacked), with the recipe shared
rather than duplicated.

#### A latched phase, not a derived one

The first implementation derived the phase from power (`climbing = total_power
<= climb_to`). Twenty minutes of running stayed at "climb 30 %": once the hold
began, any dip below 0.995 flipped it back to climbing, so hold time never
accumulated and 1 s steps interleaved with 20 s steps. Latched now, for the same
reason `rwm_latched_group` is latched.

#### Three checks that were measuring nothing

Mutation testing caught all three.

- The equivalence check drove **both** sides from `c.RATED_RUN_CLIMB_DT`, so
  changing that constant moved both and nothing diverged. The reference side now
  uses the literal values `mk_aged` writes (1.0, 0.10).
- The progress check never entered the hold phase, so the hold formula was never
  evaluated.
- The snapshot check only corrupted a file, which pickle rejects on its own —
  so removing the integrity test changed nothing. What that comparison actually
  guards is a snapshot filed under **someone else's key**, and checking that
  kills the mutant.

---

### 13.28 The panel was dead — two defects behind one report

Reported: too many bugs, the graph does not move and no button does anything.

Reproduced first. A freshly launched panel was healthy — simulated time
advanced 2.10 s in 3 s of wall clock, and invoking **all 221 buttons and
checkbuttons** in the widget tree produced zero exceptions and left the tick
running every time. So the question became *which action kills it*, and the
answer was the §13.27 five-day button, which carried two defects.

#### The computation ran on the main thread

Chaining `root.after(1)` lets tkinter run **many queued callbacks inside a
single `update()`**. Measured:

```
update() latency   mean 0.100 s, worst 7.688 s
panel sim time     +0.00 over 40 s   (physics never stepped)
chart samples      4 -> 4            (graph never grew)
```

for the three minutes the run takes. Buttons register but nothing calls
`refresh()`, so the screen never changes — exactly "no button does anything".

#### Worse: one press killed the panel permanently

`tick()` returned early when `tick_enabled` was false **without rescheduling
itself**, so the chain ended there. The five-day button disabled the tick for
the duration and restored the flag afterwards — but nothing re-arms `tick`, so
the panel never simulated again even after the run finished. Restarting the
application was the only recovery. That is why a single press made the whole
panel look broken.

#### Fixes

The computation moved to a **worker thread**. The reactor being built is a
different object from the one the panel displays, so there is no shared state;
widgets are touched only from the main thread (200 ms polling).

| | before | after |
|---|---|---|
| `update()` mean | 0.100 s | **0.002 s** |
| `update()` worst | 7.688 s | **0.234 s** |
| sim time over 40 s | +0.00 | **+30.00** |
| chart samples | 4 -> 4 | **4 -> 18** |
| events processed | 389 | **8350** |

`tick()` now reschedules itself even while disabled. This is not specific to
the five-day button: it removes the trap for any future feature that toggles
the flag. Physics still does not advance while disabled, so test determinism is
unchanged. Cancellation was added: the button becomes `취소 (21%)` and stops
within 0.1 s.

Verdict: **CORRECTED**, both.

#### Why no test caught it, and the gap that was closed

Every other check sets `tick_enabled = False` and steps the physics directly —
deliberately, to decouple results from the wall clock — which means **the real
event loop is never exercised**. Neither a seven-second freeze nor a severed
timer chain is visible to any of them.

`gui_test` now drives the real loop for two seconds after pressing the button
and measures `update()` latency, panel simulated-time progress and the number
of events processed, then cancels. Reverting each defect makes it fail with the
matching message ("계산 중 제어반의 물리가 멈춰 있다").

Closing that gap also exposed a defect **in the verification tooling**:
`gui_test.check()` printed a traceback for every exception, and `mutate.py`
classifies output containing a traceback as a crash (ERROR) rather than a kill.
Checks that correctly caught a reverted defect were therefore not counted.
Assertions now record only their message; genuine crashes still print a
traceback. "Caught it" and "crashed" are different signals and must look
different.

The tally could not be automated, though: `mutate.py` builds a fresh isolated
tree per case, and that tree has no `.test-cache`, so every case rebuilds the
`aged` fixture — the baseline alone takes 349 s and a full pass exceeds 25
minutes. The evidence is the per-case log text instead: reverting defect ①
produces "계산 중 제어반의 물리가 멈춰 있다 (그래프가 안 움직인다)", and
defect ② and the ignored-cancel mutant produce "취소가 안 먹는다". All three
are caught. `cases_loop.py` records the cost at the top — run it only when the
panel's event loop changes.

#### What this says about the test strategy

Decoupling tests from the wall clock was and remains right. The cost was
invisible until now: **defects whose cause is the wall clock have no observer.**
One check that drives the real loop is the minimum, and it belongs in the GUI
layer where a display already exists.

---

### 13.29 Electrical re-verification — a diesel cannot start on a dead battery

The question was simply "if DC A1 is flat, can the bus 101 diesel still be
started?" Measuring the model said **yes, it starts** — `_diesel()` contained no
reference to DC anywhere, while its own docstring claimed "breaker control power
is DC, so it starts even with all AC dead". The claim was in prose, not in code,
and the case where that DC is *gone* was never considered.

#### Evidence from the newer revision

The NRC HRTD R-304B revisions (Rev 09/11) are far cleaner than this repository's
OCR of R-104B. From 9.2 ([ML11258A358]) and 9.4 ([ML11258A361]):

> **9.2.4.7 System Interfaces** — "Control power for EP system circuit breakers,
> **diesel generator field flash** and other components is provided by the DC
> Power system."

And **Tables 9.4-1/2/3** give the per-division load lists in full — the very
tables that were figures (and therefore absent from the extracted text) in the
older revision, which is why the model previously said "the manual does not say
which division" and spread the loads evenly.

Division I (A1) carries, among others: RCIC motor-operated valves; RCIC
condenser vacuum and condensate pump motors; **diesel generator 101 fuel oil
pump**; control power for ADS I, RHR, CS I, backup scram A, **4160V and 480V
emergency switchgear division I**, and **diesel generator 101**; plus **field
flashing for diesel generator 101**. Divisions II and III repeat the pattern for
DG102/DG103, with HPCI on II.

The engine itself is not DC-cranked — R-304B 9.2.3.2: "Each EDG is provided with **two
full capacity air starting systems** … each capable of storing air for five
normal starts." So the engine will turn on air, but without DC it cannot flash
its field or close its output breaker, and its fuel oil pump is dead. The result
is the same: no power to the bus.

#### Corrections

1. **Diesels now require their own division's DC.** `dg_dc_ok(i)` =
   `dc_bus_ok(DC_DIV_DG[i])`. Measured: depleting A1 alone stops DG1 and kills
   bus 101 while 102 and 103 run normally — correct, since the divisions are not
   cross-tied. This creates a **one-way door**: a flat battery prevents the
   diesel, and no diesel means the charger is never fed, so the battery never
   recovers. That is the correct physics of a station blackout that outlasts its
   batteries.
2. **HPCI and RCIC now hang on their own divisions** (II and I respectively)
   for both the battery drain and the control-power gate. Previously either one
   ran as long as *any* ESF division survived.
3. **The diagram's DC-load box was misleading.** It read "제어·계측 · 차단기 ·
   HPCI/RCIC 제어" while being drawn fed from the **non-safety D division**. The
   tables place those loads on the ESF divisions. The box is now the non-safety
   D load (main turbine control and protection, plant computer) and each DC box
   states what its own division feeds.
4. **Diesel fuel tank derived from the cited seven days** (R-304B 9.2.3.2) instead of a
   hand-typed 30,000 kg, which was 1.45 days.

#### Table 9.2-1 verified unchanged

All five bus assignments match the table exactly: RHR A/B/C+D, CS A/B, RBSW
A/B/C+D, CRD A/B on buses 101/102/103, and the DC divisions taking their
chargers from the corresponding emergency bus.

#### A duplicated ledger, again

The new check for per-division pump drain **survived** its mutation. The extra
load was computed twice — once in `_battery` (discharge) and once in
`dc_div_hours` (remaining-time display) — so changing one left the other
answering by the old rule. Unified into `dc_extra_load()`, after which the
mutant dies. This is the same disease as §13.26's two electrical ledgers.

#### Still open

- **Battery duty.** 9.4.3.3: "The 125 VDC batteries can meet worst case loads
  for **two hours** in the event of a blackout" (24 VDC: four hours, which
  matches `NMS_DC_ENDURANCE_H`). The model's `DC_ENDURANCE_H` is **8 hours**,
  sourced from Fukushima Unit 1. The two measure different things — two hours is
  the design worst case with ESF initiation loads in the first minute; eight
  hours is an observed blackout without them. The model's multipliers (HPCI 1.6,
  RCIC 1.3) cannot reach two hours from an eight-hour idle basis. Recorded rather
  than patched with an invented ESF-initiation term.
- **LOCA load sequencing.** Table 9.2-2 staggers loads at 0/2/7/12/15 s after
  the output breaker closes; the model applies them all at once, which is the
  conservative direction for the diesel overload check.

[ML11258A358]: https://www.nrc.gov/docs/ML1125/ML11258A358.pdf
[ML11258A361]: https://www.nrc.gov/docs/ML1125/ML11258A361.pdf

---

### 13.30 The bubbles that held up an empty vessel

Report: **"the 'vessel depleted, no steam generation' alarm appears at 61%
core uncovery — is that intended?"**

It was not. The two lines are driven by different quantities:

```
alarm     : water_mass <= MIN_WATER_MASS x 1.01   -> inventory
uncovery  : how far water_level sits below TAF    -> level
```

If the inventory can bottom out while the level does not, the panel
contradicts itself. Measured, by actually boiling the vessel dry:

```
at the alarm: water 1008.6 kg, void 62.4%, level -761.2 cm, uncovery 69.1%
```

#### The swell term was added on one side only

Below TAF the level interpolates TAF..BAF on remaining inventory (§5.58), but
the "inventory" being interpolated included the **whole core void volume**,
while the floor reference did not:

```
v(liquid) =  1.374 m^3
v(void)   = 20.803 m^3   <- added although there is no water
v_min     =  1.362 m^3   <- no void term at all
frac = (1.374 + 20.803 - 1.362) / (68.825 - 1.362) = 0.3085 -> 69.1% uncovered
without the void term                                 0.0002 -> 100.0%
```

The term *grows as water leaves* because `_channel_hydraulics` forces the void
of uncovered nodes to VOID_MAX. That forcing is correct where it lives — no
water means no moderation, and the reactivity must fall. The defect is that a
second consumer read the same number as "how much has the boiling water
swollen". **Bubbles in a dry channel lift nothing.**

This is why §5.58's fix of the same property survived: its verification table
(`1000 kg -> -878.8 cm -> 100.0%`) was taken at **zero void**, so the input
never exercised the defect — and the table was a one-off print, never a test.

#### The fix: only submerged slices swell

```
frac = (v_liq + swell(frac) - v_min) / (v_taf - v_min)
swell(frac) = core channel volume x mean(axial void x submergence) / 100
```

Submergence follows from the level and the level follows from the swell, so
this is solved as a fixed point. The map is a contraction (slope <= 0.44), so
the solution is **unique**: at the inventory floor, every void distribution
gives frac = 0, i.e. BAF and 100% uncovery. The previous step's `submerged`
array is deliberately *not* used — that would make the level depend on
history, and asking for the level at a given inventory must give one answer.

Continuity at TAF is exact (measured −497.800 cm against TAF −497.8), because
frac = 1 corresponds to precisely the inventory at which the upper branch
reads TAF.

#### The first fix was wrong, and measurement — not mutation — caught it

The first attempt multiplied the *core-average* void by the submerged
*fraction* and solved in closed form. But that average is already contaminated
by the forced VOID_MAX of the dry nodes.

```
from 30 t at operating temperature, decay heat only (5 s x 300 steps)
variant        step 0    100      200      300      water left
original bug    41.5%  **38.0%**  42.0%    50.1%    30.0 -> 14.7 t
first attempt   42.2%    43.9%    48.5%    56.8%    30.0 -> 16.9 t
fixed           42.6%    53.1%    57.2%    63.5%    30.0 -> 18.8 t
```

The original defect reads uncovery **falling** from 41.5% to 38.0% while 5 t
of water boils away. The first attempt does not reverse, but lags badly: it
has lost more water (16.9 t left against 18.8 t) and reports less uncovery.

> This evidence was itself mis-taken once. The first measurement started from
> a **20 C cold** vessel and reported "the first attempt goes backwards,
> 62% -> 57%". That reversal was thermal expansion — heating 20 -> 260 C drops
> the density 1001 -> 783, so the volume can grow while the mass shrinks, which
> is correct physics. Only on repeating it hot did the reversal turn out to
> belong to the original defect.

That wrong answer passed all six contracts written up to then — the floor, the
monotonicity, the TAF continuity are all satisfied by it. Separating it
requires a case where **the average is identical and only the distribution
differs**:

```
A uniform 45%        — bubbles are also in the submerged region
B bottom 0% / top 90% — bubbles are entirely in the dry region
Same core average. A's level must be higher than B's.
```

#### The same contamination drove natural circulation

`natural_circ` uses the core-average void as a buoyancy proxy:

```
uncovery 60.9% : core average 55.5% -> natural circulation 0.300 (rated cap)
                 submerged only 5.6% -> 0.110
```

A dry core circulated better than a healthy one. `wet_void()` now owns that
question, and natural circulation asks it. When the core is fully covered the
two are **identical**, so no normal-operation, startup or rated behaviour
moves at all; they diverge only once the core uncovers.

The submergence profile itself had two copies (`_fuel_temperature` and the
level solver); `submerged_profile()` is now the single owner.

#### Also: the panel announced uncovery twice

Once keyed on fuel temperature (>0.5%) and once on clad temperature (>0%), so
two nearly identical lines appeared side by side and read as two separate
events. Merged into one line carrying both temperatures.

#### Effect on accident progression

```
isolation + total loss of makeup, 24 h after scram
             water     uncovery   peak clad   clad damage   fuel melt
before       1.00 t     69.18%      2519 C       2.87 h       3.37 h
after        2.77 t     96.09%      3040 C       2.82 h       3.31 h
```

The timing barely moves; what changes is what the plant *says about itself*
and where it ends up. Previously it insisted a third of the core was still
submerged while the fuel was molten.

#### Cost — nothing in normal operation, real only in the accident branch

`water_level` is evaluated **135 times per step**, so the fixed point
multiplies straight through. Two things keep that contained.

First, the **upper branch is untouched**: the per-height decomposition is only
needed below TAF, so above it the function still takes a single `avg(void)`,
exactly as before. Normal operation almost always ends there, and costs
nothing extra.

Second, the iteration body uses **plain Python floats, not numpy** — with
NZ = 12 a numpy call costs more than the arithmetic inside it (the numpy
version measured 0.33 ms per call).

Measured with `timeit` on identical copies of one state, minimum of repeats,
nothing else running:

```
normal level, +122 cm    water_level per call  0.0048 -> 0.0047 ms   x1.0
below TAF, 62% uncovered                       0.0059 -> 0.0538 ms   x9.2
                                               (135 calls/step -> +6.5 ms/step)
```

Below TAF the call is **9x more expensive**. That is a price paid, not a
saving: what it buys is the right answer in that state, and that state is
already the slowest part of the run.

> The numbers first written here ("faster than before the fix") were wrong.
> The integration suite was running in the background, and worse, the two
> variants were timed on **different states** — stepping the first variant
> advanced the reactor, so the second measured a different point in the
> transient. Step-level timing cannot isolate this change in any case, because
> it also changes how often `max_dt` subdivides. Only the per-call cost is
> attributable.

The iteration cap is 40. The worst case is TAF to BAF,
log(381/1e-4)/log(1/0.44) ~ 19, and 19 was measured. Whether it converged is
checked **from the answer, not the code**: a contract feeds the returned level
back through the equation and requires it to be a fixed point. Lowering the
cap to 8 breaks that contract (confirmed by mutation).

#### Open

- **Boil-off alone can no longer empty the vessel.** Heat into the water is
  scaled by `(1 - uncovery/100)`, and near 96-97% that share matches the
  ambient loss, so boiling stops (2.2 t left after 72 h). The remaining water
  being in the lower plenum makes a slowdown plausible, but multiplying core
  power by the covered fraction is a crude coupling and the stopping point has
  weak basis. The "vessel depleted" alarm now requires an actual drain path.
- Natural circulation now reads the submerged void, but the correlation still
  depends on **void alone**; real buoyancy also scales with the height of the
  two-phase column, so 0.110 at 61% uncovery is likely still high. Recorded
  rather than replaced with an invented model.

---

### 13.31 Hydrogen — the pressure that cooling cannot remove

Request: model the effect of accident hydrogen on the wetwell and drywell —
but core oxidation itself is not the point, so keep it simple.

#### What existed

Of `Zr + 2H2O -> ZrO2 + 2H2 + heat`, only the **heat** was modelled. No zirconium
inventory, no steam consumption, no hydrogen. Temperature alone drove it, so it
burned at full rate forever. Measuring what that heat implies over 24 h of
isolation with total loss of makeup:

```
2256 GJ of reaction heat -> 351 t of zirconium (over 3x the whole core inventory)
                         -> 15.5 t of hydrogen
                         -> 139 t of steam consumed (78% of the 179 t actually boiled)
all of it in the drywell -> 5,308 kPa partial pressure (design 529 kPa abs)
```

So hydrogen could not simply be back-calculated from the existing heat: it gives
ten times the design pressure. Adding hydrogen means making the reaction
conserve mass.

#### One cap was enough

No oxidation kinetics and no per-node zirconium inventory were needed. A single
**steam limit** suffices — the reaction eats steam, so it cannot outrun boiling.

```
  Zr stock  steam cap   Zr used    H2       stock runs out   steam binds
   60 t      yes        21.4 t     948 kg        never          4.18 h
  100 t      yes        21.4 t     948 kg        never          4.18 h
   none      no        351.2 t   15522 kg
```

Three things fall out at once. The zirconium inventory **need not be researched**
— the steam binds first, so 60 t and 100 t give the same answer, and a number
that does not change the result is not worth sourcing. The **calibrated core
behaviour is untouched**, because the cap first binds at 4.18 h while fuel melt
is at 3.32 h (§5.66). And the resulting partial pressures land where venting
actually becomes a decision.

Stoichiometry is derived from molar masses, never typed: 1 kg Zr yields 0.0442 kg
H2, consumes 0.395 kg H2O and releases 6.42 MJ. Heat, hydrogen and steam
consumption all come from **one rate** — the third instance of the "two ledgers"
disease would otherwise have been born here (§13.26, §13.29).

#### Transport reuses the existing piping

`steam_out` already splits into valve, SRV, break, turbine-drive and tunnel
paths, and already scales them together when inventory runs short. Hydrogen
rides those flows. SRV and turbine-drive steam pass through the suppression
pool, where the steam condenses and **the hydrogen does not** — it collects in
the wetwell gas space. A break delivers to the drywell. Main-steam and tunnel
flow leave the containment entirely.

In the containment, hydrogen is simply a third species that behaves exactly like
nitrogen: non-condensable, ideal. The downcomer and vacuum breaker already moved
a proportioned mixture; hydrogen needed one more fraction.

#### The behaviour this buys

```
cooling the pool no longer recovers the pressure
  pool 48.8 -> 21.9 C
  wetwell 168.0 -> 149.6 kPa
    saturation  2.7 kPa   <- collapses when cooled
    nitrogen   74.2 kPa
    hydrogen   72.7 kPa   <- does not
  hydrogen 202.3 -> 203.2 kg
```

Before this, RHR pool cooling always brought containment pressure down, because
the pool's saturation pressure was the dominant term. It no longer does.

Purging is the only way out, and it already existed: the purge empties the
drywell gas space rather than selecting nitrogen, exactly as 4.1.3.7 describes
("Gas releases are also performed periodically to ensure containment pressure
remains below 30 psig"). The wetwell has no direct vent, so its hydrogen must
first cross the vacuum breaker into the drywell — which the measurement shows
happening (wetwell 236.1 -> 84.7 kg while purging).

#### Five defects found while building it

1. **The steam cap counted inventory.** Vessel steam sits mostly in the dome and
   never passes the core, so counting it made the cap meaningless (21.4 -> 35.7 t).
2. **Hydrogen was trapped in the vessel.** Outflow was tied to *realised* steam
   flow, so when the core dried and boiling stopped, hydrogen stopped leaving
   too — 1081 kg stuck behind an open relief valve. Outflow now follows **valve
   capacity** (the demand before the inventory cap) times the hydrogen mass
   share, with a sqrt(M) choked-flow factor: the same valve passes about a third
   as much hydrogen by mass as steam.
3. **The pending hydrogen was handed over as a rate.** `_containment` runs before
   `_vessel`, so delivery lags a sub-step — and `step()` changes its subdivision
   with plant conditions, so the two sides could multiply by different dt.
   **91 kg vanished in 400 steps.** Carrying a mass instead removes the gap.
4. **`_vessel` subdivides ten times internally** (`_vessel(dt, sub=10)`). Applying
   the steam cap there compounded it ten-fold per step and corrected the
   accumulators ten times (0.17 kg in one event). The cap now lives in
   `_fuel_temperature` alone.
5. **The downcomer equilibrium assumed the gas being moved was nitrogen.**
   Hydrogen's gas constant is 14x nitrogen's, so the same mass makes 14x the
   partial pressure; treating it as nitrogen overestimates the mass to move by
   the same factor, overshooting an equilibrium that the water leg makes
   irreversible. The mixture's mass-weighted gas constant is now used, which also
   corrects steam (461.5 against 287).

#### What mutation caught: measuring the part, not the result

A mutant that removed only the hydrogen term from `ww_pressure` **survived**. The
contracts examined `ww_p_h2` — the partial-pressure *function* — but never
checked that it entered the total. The drywell had that check; the wetwell, the
actual subject of this work, did not. Adding it gives 15/15.

#### Cost

Step cost is essentially unchanged, measured back-to-back on one machine:
normal operation 15.61 -> 16.08 ms/step (+3%), accident 65.38 -> 64.37 ms/step.
The contract layer did grow; while trimming it, two classes were found running
**the same preheat separately**, now shared through `preheated()` (11 s saved).

#### Deliberately out of scope

Combustion inside containment: Mark I is nitrogen-inerted (4.1.3.3), so there is
no oxygen and nothing burns there — that is why no oxygen inventory is tracked.
The reactor building, where Fukushima's explosions actually occurred after
leakage, has no atmosphere in this model. Radiolytic hydrogen, listed by the
manual as the second source, is negligible next to the metal-water reaction in
an accident. The CAD nitrogen-dilution system is not modelled; purge is the only
removal path.

#### Open

- **Hydrogen trapped in the vessel.** With every valve shut there is no way out —
  689 kg of 933 stayed in the RPV in the isolated case. A real plant would breach
  the vessel and release it all at once; that path does not exist here. With a
  break, everything leaves (vessel 0 kg).
- The molecular-weight correction for choked flow is a single sqrt(M) factor,
  assuming both gases move at the same velocity.

---

### 13.32 No way back to the normal source

Report: cutting the normal source transfers to the backup, but restoring the
normal source does not transfer back — so turning off the backup produces a loss
of offsite power.

#### The missing automatic transfer is correct

9.2.3.3 is explicit: "a fast transfer to that source is made approximately 5
cycles after bus undervoltage is detected. The function is only available from
the normal to the backup source of preferred power **and not in reverse**."

So not returning automatically is the model obeying the manual. Adding an
automatic return would be the error.

#### What was missing was the operator's action

`restore_offsite()` returned `False` immediately whenever a source was already
feeding the buses. Once transferred to the RSST, the plant stayed there forever
even with the NSST healthy, and the only lever left was to drop the backup —
which, with no reverse transfer, is a loss of offsite power while the normal
source sits available. That dead end is exactly what was reported.

The panel button's own docstring already claimed the operation existed
("the fast transfer is one-way, so returning from RSST to NSST needs this
action"), and the help text stopped at restoring from a dead bus. The intent was
written down; nothing owned it.

#### The fix

One switch, two operations: reconnect when nothing is feeding (normal source
first), and transfer back to the normal source when running on the backup. The
return is **uninterrupted** — unlike the fault-driven fast transfer this is a
planned action, done on real plants by paralleling the two sources briefly
before opening the old breaker. No diesel start.

```
start              NSST
normal source cut  RSST    fast transfer NSST -> RSST (5 cycles)
normal restored    RSST    (no automatic return — per the manual)
restore button     NSST    manual transfer RSST -> NSST
backup then cut    NSST    nothing happens
```

Losing the backup while still on it remains a loss of offsite power — that is
correct — but the event now says why, instead of leaving the operator staring at
a healthy normal-source lamp and concluding the model is broken.

#### Tests

Seven contracts, mutation `cases_xfer` 7/7. One case **adds** the reverse
automatic transfer, so the contract set catches an over-correction into a manual
violation as readily as it catches the original gap.

---

### 13.33 Turbine building cooling water and station air — triple check

Re-verified against the old manual (R-104B), the revision (R-304B) and the web,
looking at **inter-system coupling and operating logic** rather than numbers.

#### Confirmed as modelled

TBCLCW at 17,800 gpm driven by a 1,000 hp motor is identical in both revisions.
The **10-second standby discharge valve delay** is stated only in the *old*
manual (§11.5.4.1) — the revision gives the interlock without the number, one of
the rare places where R-104B is the more specific source.

#### A contradiction in the old manual, settled by the revision

On what cools the compressor aftercoolers, R-104B says two different things:
§11.6.2.2 calls them "air to water heat exchangers cooled by **raw water**"
(and the chapter summary lists Raw Cooling Water as the interface), while
§11.6.4.1 says "The **Turbine Building Cooling Water System** supplies normal
cooling for the compressors and after coolers." The revision (§11.6.4.2) settles
on TBCLCW. The model had already followed the correct one.

#### The load list was one item long

The revision carries **Table 11.5-1**, absent from the old manual, listing every
TBCLCW load: station air compressors, condensate pump motor bearings, condensate
booster and feedwater pump turbine lube oil coolers, condenser air removal pump
lube oil and sealing water coolers, **generator stator cooling unit**, exciter
and generator leads coolers, **main turbine lube oil coolers**, hydrogen
coolers, **EHC coolers**, offgas equipment, plus radwaste and office building
loads. The model connected only the compressors, so losing the loop did nothing
until the instrument air header had bled down.

The stator cooling path is corroborated three ways inside the revision — Table
11.5-1 lists it as a TBCLCW load; §9.1.3.1.3 states "the stator cooling system
heat exchangers are cooled by the turbine building closed loop cooling water
system"; and §3.2 with Table 3.2-1 gives the consequence: low stator inlet
pressure (<13 psig) or high outlet temperature (>95 °C) runs the generator back
to below 25% load, and trips the turbine **after a 70 second delay** if generator
current is still above 5811 A. The web returns the same figures.

```
TBCLCW pumps stopped at rated power
before   compressors only -> instrument air decays -> scram at ~8.5 min
after    stator cooling lost -> turbine trip at 70 s -> scram follows
```

The load condition is modelled with the 25% runback target, since the model has
no generator current; 5811 A is roughly 20% of rated, sitting just under that
target, which supports the reading. The **runback itself is not modelled** — this
turbine is pressure-following and has no load-control path — so only the trip is
implemented, with an alarm for the stage that precedes it.

#### A two-unit arrangement had been imported into a single-unit plant

Read closely, the old manual explains its own compressor count: "Two compressors
are operated simultaneously to meet normal air requirements **for both units**."
The revision describes a single unit — "one continuously operating with two
normally in standby" — and supplies the pressure ladder the old manual states
only as an ordered list: 110 psig starts the primary backup, 105 psig the
secondary, 95 psig isolates the service air header (AOV-010). The structure came
from the old manual, the numbers from the revision, and only the scram-header
threshold remains without a basis in either.

The revision adds one behaviour the model had backwards: "Once started ... an air
compressor remains running until manually shut off or tripped." Compressors do
not stop when pressure recovers. One machine at 1,353 scfm covers the modelled
1,200 scfm demand with 13% margin, so a backup start is now a real event.

#### Nitrogen, not air, inside the drywell

The revision explains an arrangement the model had stumbled into correctly: the
drywell pneumatics — safety relief valves and **inboard** MSIVs — run on nitrogen
from the containment inerting system, specifically so that a leak inside the
drywell cannot raise the oxygen concentration of the inerted atmosphere and
defeat the protection against hydrogen ignition. That connects directly to
§13.31. The web confirms the same arrangement at Browns Ferry, with the inboard
MSIVs fed from the instrument nitrogen system at a nominal 105 psig and local
accumulators as backup.

The model never tied SRVs to the air header, which turns out to be right. Its
single lumped MSIV is air-driven, which matches the **outboard** valve and gives
the same isolation outcome either way; the comment now says so.

#### Open

Every other Table 11.5-1 load is still unconnected — main turbine and feedwater
pump turbine lube oil, EHC, hydrogen coolers, condensate pumps, condenser air
removal. The stator path is the fastest (70 s) so the outcome does not change,
but a stator-only failure has nothing behind it. Also unmodelled: the EHC
runback; the 15 psi design-pressure margin over offgas and TBSW that makes leaks
travel outward, and the pump-discharge radiation monitor that catches in-leakage
from a failed cooler; the six compressor trips of §11.6.3.1 (intercooler outlet
125 °F, lube oil 20 psig and 140 °F, vibration, 70% undervoltage), which the
model lumps into one loop-temperature trip; the 450 scfm dryer rating against a
modelled 800 scfm instrument air demand; the second 100% heat exchanger; and
pressure control valve PCV-092.

### 13.34 Drywell and suppression chamber — triple check

**Sources.** R-104B §4.1, §4.4.3.3, Table 4.1-1 (`manual/part2.txt`);
R-304B §4.1, §4.4, §10.4 (`manual/r304b/`); the web for operating history
and technical specifications.

#### 13.34.1 The two revisions describe different containments

The numbers disagreed so widely that the first task was to find out why.
They are not two accounts of one plant.

| | R-104B (Rev 0300) | R-304B (Rev 09/11) |
|---|---|---|
| Containment | **Mark I** | **Mark II** |
| Drywell shape | bulb (sphere + cylinder) | truncated cone |
| Suppression chamber | torus alongside | "directly beneath the drywell" |
| Drywell free volume | 159,000 ft³ | 192,500 ft³ |
| Gas space | 119,000 ft³ | 134,000 / 138,500 ft³ |
| Pool water | 135,000 ft³ | 76,870 / 81,385 ft³ |
| Internal design pressure | 62 psig | 48 psig |
| Vents | 8 × 81 in | 88 × 23.25 in through the floor |
| Vacuum breaker ΔP | 0.5 psi | 0.25 psid |
| Drywell coolers | 10 | one to eight |
| ΔP control | §4.1.3.5 (1.3–1.5 psid) | **section removed** |

The figure titles settle it: Figures 4.0-1, 4.0-2 and 4.2-1 are all
captioned **"Mark II Containment"**. Some BWR/4 units do use Mark II
(Limerick, Susquehanna).

**Consequence for this verification.** R-304B Chapter 4 *geometry,
pressures and counts do not apply to this model*; its *system logic,
isolation groups and interfaces do*, because those are common BWR/4
design. Mark I geometry stays sourced to R-104B Table 4.1-1.

Without that distinction the pool water volume would have been "corrected"
from 135,000 ft³ to 81,385 ft³ — replacing a right number with one from
another plant. **A later revision is not automatically a better source for
the same question; this one was answering a different question.**

This also revises how §4.7 of this document should be read: its Table 4.1-1
citations are Mark I and remain valid, but any future comparison against
R-304B §4.1 must name the containment type as well as the revision.

#### 13.34.2 Section numbers shifted again

| Section | R-104B | R-304B |
|---|---|---|
| 4.1.3.3 | Nitrogen Inerting | Containment Heat Removal |
| 4.1.3.5 | Drywell/Suppression Chamber ΔP Control | Suppression Pool Temperature Monitoring |
| 4.1.3.7 | **Containment Atmosphere Dilution** | **10CFR50 Appendix J Testing** |
| 4.1.4 | System Interfaces | Summary |

Eight citations of "교범 4.1.3.7" in the code and panel were R-104B numbers
carrying no revision label. In R-304B that section is leak-rate testing;
the combustible-gas material moved to **4.1.2.4.4 Containment Combustible
Gas Control**. All eight now name the revision. Same trap the R-304B index
caught last round (§13.33).

#### 13.34.3 Four couplings that were not there

The brief was relationships and operating logic rather than constants, and
four gaps turned up by reading the code against §4.1.3.8's interface list.

**A. Pool level fed nothing.** `pool_level` was display-only.
`DOWNCOMER_SUBMERGENCE` was a constant, so hundreds of tonnes of SRV
discharge left the vent hydrostatic leg unchanged.

The web supplied the basis that the manual did not: a Vermont Yankee
technical specification change requires a suppression chamber level
"corresponding to a downcomer submergence range of 4.29 to 4.54 ft", and
other plant documents note that at the minimum level "submergence is
approximately four inches less". Submergence tracks level inch for inch —
which is *why* the level is a specification item.

`Containment.submergence()` and `gas_volume()` now derive both from pool
inventory. `POOL_SURFACE_AREA = 1003 m²` is derived from R-104B Table 4.1-1
torus dimensions (111 ft centreline diameter, 31 ft cross-section →
348.7 ft circumference × 30.96 ft surface width = 10,796 ft²); the check is
that 2π²(55.5)(15.5²) = 263,200 ft³ matches the table's 254,000 ft³ within
3.6%. Submergence clamps at zero — below that the vents are uncovered and
there is no pressure suppression, not a negative water leg.

Measured, four SRVs for one hour:

```
        pool[t]  subm[m]  leg[kPa]  gas[m³]  WW[kPa]
  0 min    3804    1.200     11.71     3400    110.0
 60 min    3911    1.307     12.76     3292    117.5
```

Submergence moved 4.2 inches — the width of the specification band.

**B. Drywell coolers ignored accident signals.** The model required only
`buses_live > 0`. R-304B §4.4 Group 8 closes "all the RBCLCW primary
containment isolation valves **and drywell unit coolers isolation valves**"
on high drywell pressure, Level 1, or RBCLCW head tank level low-low.

The feedback is perverse by design: the signal that stops the coolers is
the signal that the drywell is hot. NRC Information Notice **84-35, "BWR
Post-scram Drywell Pressurization"**, records it happening — at Hatch 2
(Aug 1982) "the 2.0 psig drywell pressure scram signal was still present"
and staff "had to restore the drywell coolers to operation in order to
rapidly reduce the drywell pressure"; at Quad Cities 2 (Jun 1982) the
coolers tripped at 2 psig along with the second scram and ECCS initiation,
and the logic was later modified.

The model's `DW_HIGH_PRESSURE` was already 2 psig. The isolation latches
(the notice's "could not be readily reset"), and `reset_dw_cooling()`
refuses while a signal stands.

Measured, a 0.06 % steam leak over 40 minutes:

```
            coolers keep running     Group 8 isolates
  0 min     109.7 kPa   56.3 °C     109.7 kPa   56.3 °C
 20 min     132.4 kPa   67.4 °C     143.6 kPa   92.3 °C
 40 min     138.2 kPa   73.6 °C     166.4 kPa  111.2 °C
```

The old behaviour nearly plateaus; the new one does not.

RBCLCW itself (§11.3) is still absent, so the head-tank signal is omitted.

**C. Purge had no isolation interlock.** Both revisions agree here, which
is rare: R-304B §4.4 Group 9 (high drywell pressure, Level 2, refuelling
floor exhaust radiation high, reactor building low differential pressure)
and R-104B §4.4.3.3 ("primary containment vent and purge system" isolates
on low water level or high drywell pressure).

Blocking it outright would have produced a deadlock, because purge is the
only way the model removes drywell pressure and that pressure *is* the
isolation signal. The plant resolves this with a deliberate bypass —
R-304B 4.1.2.4.4: "If containment pressure approaches 24 psig a purge will
be initiated to the reactor building using the containment purge filter
system. Venting continues until containment pressure has been reduced to
atmospheric." That is far above the 2 psig isolation setpoint. Post-accident
venting is done *with the signal present*. The latch stands; an operator
override opens it.

`PURGE_RATE` also gained a basis. The old 6.0 kg/s had none and exceeded
even the unfiltered 10,000 scfm purge fan. R-304B 4.1.2.4.2 gives 1,000 scfm
through the purge filter when radiation is detected and states the filter
and fan "may also be used to vent excess pressure from the primary
containment" — which is exactly what this model's purge does. 1,000 ft³/min
= 0.4719 m³/s × 1.165 kg/m³ = **0.55 kg/s**, emptying one drywell volume in
about 2.5 hours rather than 14 minutes.

**D. Containment spray had no permissive.** R-304B §10.4.3.4: the test line
valves and "containment spray valves (MOV-38A, B 39A, B, MOV-40A, B, and
MOV41A, B) are interlocked closed on a LPCI initiation. These valves can be
opened if adequate core cooling exists and the LPCI initiation signal is
bypassed with the containment spray valve override switches."

The purpose is to stop the operator spraying into containment the water the
core still needs. `spray_blocked()` is the single owner; the displayed
reason (`pool_cooling_reason`) and the calculation (`_rhr`) read it, so the
screen cannot disagree with the physics. `set_spray_override()` refuses
below the top of active fuel. The latch sits outside the `eccs_auto` guard
because the interlock is driven by the *signal*, not by whether pumps ran.

#### 13.34.4 A surviving mutation found a hole in the contracts

23 contracts, fixture-free, 0.95 s; 22 mutations plus a control in 35.8 s,
all 23 behaving as intended. The split into `cases_cont.py` was made from the start on the
§13.33 lesson.

The first run left one survivor: reverting the vent leg to a constant.
`test_a_swollen_pool_holds_the_vent_shut_longer` passed anyway, because a
rising pool shrinks the gas space, which raises wetwell pressure, which
reduces the differential — the contract was passing through the *volume*
path while the *leg* path was dead. `test_the_vent_opens_at_the_water_leg`
now pins the threshold itself. **When two paths push the same direction, a
contract that watches only the outcome cannot tell which one is carrying
it.**

#### 13.34.5 Three measurement mistakes worth recording

- Setting `cc.pool_m` directly changed nothing: `step()` overwrites it from
  `r.pool_mass` every call. One owner — and the test has to address that
  owner. This is why the first A/B run matched to the decimal.
- The cooler latch test could not clear its own signal. Raising nitrogen
  1.3× pushes gas through the downcomers into the wetwell, from which the
  vacuum breakers return it, so restoring the nitrogen leaves the drywell
  high. That is the very phenomenon purge exists for. The stimulus was
  lowered below the vent threshold (1.08×) instead.
- The reason strings overflowed their readout. The panel value cell is 17
  columns; the first wording ran to 40. This repository built `fit_cols`
  for exactly this (Hangul is double-width) and the trap was still there to
  fall into. The strings were shortened — the pressure value is already on
  the cell above — and `fit_cols` now guards them. The mutation anchors were
  realigned to the new strings, which
  `test_registered_mutation_anchors_are_still_applicable` enforces.
- The fast layer can livelock. The fixture cache keys on `core/*.py`, but the
  build runs inside the layer's 120 s budget: 27 s of contracts plus a 96 s
  `rated` build exceeds it, the runner kills the build, nothing is cached,
  and the next run repeats. Adding these 23 contracts tipped a margin that
  had been holding. Clearing the locks and building once outside the runner
  breaks the cycle; removing the build from the budget is left open.
- The first Group 8 measurement showed no difference at all. Resetting the
  latch inside the loop is void — `_containment` re-latches within the same
  step — and the break was large enough to swamp a 2.5 MW cooler.
  Overriding `group8_signal` and using a 0.06 % leak produced the contrast
  above.

#### 13.34.6 Investigated, recorded, not built

Differential pressure control (R-104B §4.1.3.5; the revision dropped the
section, so whether to model it at all is itself the open question);
CAD/CCGC recombiners and nitrogen dilution (R-304B 4.1.2.4.4); the external
design pressure, which R-104B §4.1.3.8 warns about directly — steam
condensing in the drywell "is possible to create a negative pressure in the
drywell exceeding the design external pressure" (2 psig old, 10 psia new);
reactor building to suppression chamber vacuum breakers; suppression pool
temperature alarms (90 °F at two feet, 110 °F at one foot, R-304B §4.1.3.5)
against the panel's unsourced 70 °C / 95 °C; suppression chamber spray
(about 5 % of flow); the two Group 9 signals that need a secondary
containment the model does not have; RBCLCW (§11.3) itself; vacuum breaker
counts and arrangement, which differ by containment type.

### 13.35 Main turbine and condenser — triple check

**Sources.** R-104B §7.3.3.2.8, §11.1 (`manual/part3-4.txt`); R-304B §2.6.3.x,
§3.2 and **Table 3.2-1**, §11.1 (`manual/r304b/`); the web for the physics of
low-load hood heating, extraction backflow overspeed, and an independent
statement of the bypass interlock.

#### 13.35.1 This system has no chapter of its own

The main turbine and condenser are not a chapter; they are an intersection of
five. Reading them meant crossing between:

| Topic | Section |
|---|---|
| Main condenser, hotwell, SJAE, exhaust hood spray | R-304B **2.6.3.x** (inside Condensate and Feedwater) |
| Turbine trips, EHC, interfaces | R-304B **3.2** and **Table 3.2-1** |
| Circulating water (the heat sink) | **11.1** |
| Offgas (where the noncondensibles end up) | **8.1** |
| TBSW / TBCLCW | 11.4 / 11.5, done in §13.33 |

#### 13.35.2 Two candidates from the plan did not survive contact

A verification pass has to verify its own plan. Two items on the pre-work
candidate list turned out to be wrong.

**The cooling tower was already adjudicated.** Both revisions do describe
once-through circulating water drawn from and returned to Long Island Sound,
with identical pumps (143,400 gpm at 34.2 ft). But this is not a new finding:
§4.9.3 of this document already records "Cooling tower — DELIBERATE, out of
scope for comparison", the comparison table already carries the row, and the
`Tower` help text opens by saying the reference plant is once-through seawater
while the model assumes an inland site. Nothing to change. The triple check
re-confirmed a standing decision.

**The extraction non-return valves were already covered on the forward path.**
When the turbine trips, `m_turb` goes to zero, so heater shell temperatures
collapse to the condenser temperature and `cap = EXTRACT_LIMIT * m_turb`
forces extraction to zero. The heaters are already isolated from the turbine.

What the NRVs actually guard is the *reverse* path — R-304B 2.6.3.13: they
"prevent backflow of water or steam from entering the turbine and causing
damage either by water induction or over speed resulting from reverse steam
flow." The web agrees and adds the timescale: on loss of load a large turbine
"can rapidly accelerate to Overspeed in order to reverse steam flow caused by
the energy contained in the extraction and feedwater heating system," and
"flow reversal can occur in less than 0.2 seconds." This model has no
overspeed state at all — synchronized means 1800 rpm, tripped means coastdown.
An NRV without an overspeed model protects against nothing, so it is recorded
rather than built.

#### 13.35.3 The exhaust hood was missing entirely

R-304B **2.6.3.7** supplies a complete ladder: at low load the friction of
slow steam on the last-stage blading raises temperature and can damage the
blades; **130 °F** opens the spray valve, **175 °F** alerts the operator,
**225 °F** trips the main turbine. Table 3.2-1 carries the 225 °F trip again
with its reason ("thermal stress, damage to exhaust hood or potential
misalignment"). The model had no hood temperature at all, so low-load
operation had no consequence of any kind.

The physics needs nothing invented: windage power is set by speed, and the
exhaust flow is what carries it away, so the rise is `Q / (ṁ·cp)`. At rated
flow the rise is about 2 °C; it grows sharply as flow falls; and it vanishes
when the machine slows, by the cube law.

What makes this a *coupling* rather than a component is where the spray comes
from. R-304B 2.6.3.8 tees the hood spray line off the condensate system, so
losing condensate loses the spray. Measured:

```
  load %   spray available   no spray
   100.0            37.0        37.0
    15.0            48.3        48.3
     5.0            62.8        75.0
     3.0            73.5       101.6
     2.0            82.7       134.9   <- trips without spray
     0.0           126.2         --    <- trips even with spray
```

With spray, only a near-zero exhaust flow trips. Without it, anything below
about 2.5 % load does. **Loss of condensate now reaches the turbine through
the hood**, which it could not do before.

`HOOD_WINDAGE` and `HOOD_SPRAY_FLOW` are **NO BASIS** — the manual gives the
setpoints but not the windage power, so they are back-derived to make the
stated ladder behave, and labelled as such at the constants. The three
setpoints are sourced.

#### 13.35.3a The trap was checked for, and walked into anyway

Startup was checked first, because §13.33 had produced a latent trap of
exactly this shape. A twelve-hour run peaked at 41.5 °C against a 107.2 °C
trip, and that was written down as clearance.

**The measurement was wrong.** That run had tripped the turbine on Level 8
early on, so it never entered the state that matters — *turbine synchronized
at very low load*. The limitation was noted at the time and then treated as
acceptable, which it was not.

The fixture build found it instead. A full `contracts` run passed 395 CPU
seconds without producing a `.pkl`, because `climb()` runs its full 400,000
steps before reporting that the target was never reached — the failure mode
looks like a hang. Run directly:

```
n= 19640  power=0.0083  TRIP=exhaust hood 107 C  hood=107.3  exh=0
END       power=0.1594  online=False
```

Immediately after synchronization, with zero exhaust flow, the hood reached
the trip in about fifty seconds and the plant could never climb.

**The cause was a physics error: the spray was modelled as sensible heat.**
Treating spray water like extra steam (`ṁ·cp`) understates what a kilogram
absorbs by more than a hundredfold — it evaporates, so the figure is 2257 kJ,
not 2.0 kJ/K. With no exhaust flow, no amount of spray could hold the hood.

Correcting it also restored the valve's character. The manual says a
temperature element controls the spray valve; it is a *modulating* valve that
passes only what is needed, and nothing at all when the exhaust already
carries the windage:

```
  load %   condensate available            no condensate
   100.0   hood 41.0 C, spray 0.00 kg/s    same (none needed)
     5.0   hood 54.4 C, spray 1.21         rises to trip
     2.0   hood 54.4 C, spray 2.42         **trips**
     0.0   held,        spray 3.23         **trips**
```

Spray holds the hood at setpoint at any load; losing condensate trips it.
That is both the real behaviour and the coupling this section set out to
add. After the fix, startup reaches 99 % in 9.6 h with a peak hood
temperature of exactly 54.4 °C — the spray setpoint, which is the evidence
that the valve was modulating through the low-load stretch.

Two lessons. A measurement that never enters the dangerous state provides
false assurance, and "peak 41.5 °C" was true without being an answer. And
when changing a coolant, establish first whether it changes phase; water and
steam do not carry heat by the same mechanism. A mutation case now guards
the specific error.
#### 13.35.4 The bypass valves did not look at the condenser

Both revisions agree, and the web corroborates:

- R-304B **3.2.4**: "Whenever condenser vacuum is below 7" Hg, the BPVs are
  interlocked closed." The lecture copy (ML11258A318) fixes the unit —
  "BPV's are interlocked closed at less than **7 inches mercury vacuum**."
- R-104B **7.3.3.2.8** says the same thing and gives the reason: "To protect
  the main condenser against overpressure, a loss of condenser vacuum
  initiates automatic closure of the turbine stop valves and turbine bypass
  valves."
- An NRC-hosted design document states it independently: "The turbine bypass
  valves close and are prevented from opening on high condenser backpressure
  or high hot well level."

**The unit nearly produced an inverted ladder.** Read as absolute, 7" Hg is
23.7 kPa, *below* the turbine trip at 25.1 kPa, which would put the bypass
closure first. Read as a vacuum gauge — which the lecture copy settles — it
is **77.6 kPa**, far above the trip:

```
vacuum degrading (pressure rising) →
  20.0 kPa   low vacuum alarm
  25.1 kPa   turbine trip      (Table 3.2-1, 22.5" Hg vacuum)
  30.0 kPa   MSIV isolation    (this model's DELIBERATE signal, §13.20)
  77.6 kPa   bypass closes     (3.2.4, 7" Hg vacuum)
```

The turbine goes first and the bypass stays available long after — the design
keeps somewhere to dump steam for as long as it possibly can. Collapsing the
two setpoints together would silently remove that margin, and a mutation case
exists for each mistake.

`COND_TRIP_PRESSURE` also moved, 27.0 → **25.13 kPa**, from Table 3.2-1's
22.5" Hg vacuum. The old value had no source and tripped slightly late.

#### 13.35.5 Three mutations survived because the contracts copied the formula

The first mutation run killed 13 of 16. The three survivors — "the hood
ignores load", "exhaust flow does not cool the hood", "spray carries no heat"
— were all about the hood equation, and they survived for one reason: the
contracts recomputed the equation inside the test instead of stepping the
model. A test that re-implements the logic passes no matter what the logic
does. Rewritten to drive `bop.step` with real steam flow, all three die.

This is the same family as §13.34's surviving mutation, and worth stating as a
rule: **a contract that contains the formula is testing itself.**

#### 13.35.6 One contract had to move to the scenario layer

The bypass interlock was written fixture-free first, and its control would not
stand up: with only decay heat, the pressure regulator holds the bypass shut
*legitimately*, so the valve reads zero whether the interlock is present or
not. After several attempts to force a discriminating state, the right
conclusion was that the state cannot show it — the interlock only has an
effect when there is steam to dump. It moved to
`TurbineHoodScenarioContracts` with its own mutation file, keeping the
§13.33 split intact from the start.

#### 13.35.7 Investigated, recorded, not built

Table 3.2-1 lists eleven turbine trips; the model now has four (Level 8,
stator cooling, low vacuum, exhaust hood). Missing: mechanical and backup
electrical overspeed at 110/112 %, thrust bearing wear 35 mils, main shaft
oil pump < 105 psig above 1300 rpm, bearing oil 8 psig, EHC fluid header
< 1100 psig, vibration 10 mils, and loss of both speed feedback channels.
Turbine overspeed itself is absent, which is what blocks the non-return
valves. Also open: traveling screens and the 30-inch differential that trips
a circulating water pump (11.1.3.2); the bypass interlock's second signal,
high hotwell level, which the NRC document gives but R-304B does not state
for this unit; the condensate demineralizers, including the 130 °F resin
limit; the steam packing exhauster and its constant 10,000 gpm requirement,
which is the real path by which gland sealing failures admit air; booster
pump low suction pressure trips at 35 psig staggered 20 s / 45 s; two
condenser shells with cross-connects against the model's one; bypass capacity
25 % in R-304B against the model's 30 %; and the hotwell's two-minute
retention, which R-304B attributes to N16 decay (t½ = 7.11 s) rather than the
reason currently in the constant's comment.

---

### 13.36 RBCLCW implementation (2026-09-20)

The corrected plan's RBCLCW scope is implemented in `core/building_systems.py`
and connected through `Reactor`. Secondary containment was held for the next
instruction, then implemented separately in §13.37 below.

Evidence: R-104B/R-304B §11.3, R-304B Table 11.3-1, §4.4 Group 8,
and §2.8.4.4–5. The source comparison and the limits of NRC IN 84-35 are recorded
in `PLAN_RBCLCW_SECONDARY.md` §6. Source-backed settings and NO BASIS parameters
are separated in the RBCLCW section of `physics.md`.

Implemented: three independently powered pumps; standby and C restoration delay;
two loop inventories/temperatures; makeup/leak enthalpy; normal cross-connection;
powered LOCA separation and non-safety isolation; air-loss heat-exchanger alignment;
two heat exchangers; separate remaining service-water budget; actual motor loads;
DW heat transfer; recirculation/MG thermal states; RWCU NRHX isolation and separate
pump cooling-water trip. Head-tank low-low participates in Group 8 only, not in
the whole-loop LOCA separation signal. Fan reset cannot bypass water-side isolation.

Service-water loss leaves closed-loop circulation and thermal storage intact.
RWCU isolates only after its thermal detector reaches the source threshold.
SDC, pool cooling and spray share the RHR budget left after DG/RBCLCW allocations.
This closes an older path where SDC independently took its full rating.

Deliberate approximations: lumped valve groups, no MOV travel time or pressure
network; no PCV characteristic, separate booster inventory, tank overflow plumbing,
RHR/CRD seal failure, fuel-pool or radiation model. Emergency bus mapping,
hydraulic/thermal coefficients and reset policy are educational assumptions.
There is no unsupported main recirculation-pump timer trip.

Local contracts join the existing fast suite; two rated scenarios run separately.
Mutations select only fixture-free contracts. GUI tests cover command/actual state,
invalid input, snapshot replacement, help and label width. Exact commands, result
files, exit codes and manual checks are in `tests/RBCLCW_REPORT.md`; this entry
does not imply that the full 30-day/report profile has been run.

Two older contracts needed updated boundaries: DW fan reset now also verifies
that water-side isolation still prevents cooling; the zirconium stoichiometry
unit test explicitly supplies steam, as it already supplies hot cladding. The
preheated fixture can now be subcooled by the added auxiliary heat removal.
The electrical mutation anchor moved without removing its power-loss test.

### 13.37 Minimal secondary containment / SGTS (2026-09-20)

The user's next instruction authorized only the essential secondary-containment
elements. Selected R-104B §4.2/4.3's three-train SGTS, after comparison with the
local R-304B §4.2/4.3 and NRC Issue 192. This is not the two-train RBSVS design.
Source-backed facts, single-building reduction and NO BASIS coefficients are in
the new secondary-containment section of `physics.md`.

Implemented in `core/secondary_containment.py`: building air inventory/pressure;
normal supply/exhaust and isolation; four independent automatic start inputs;
latched demand distinct from fan power/failure/actual operation; manual start and
reset; failed double-door boundary; actual blower AC load; direct DW-to-SGTS-to-stack
purge sharing fan capacity and preserving the existing hydrogen ledger.
Independent radiation test contacts do not claim actual radiological measurements.
The R-304B low-vacuum start timer and its complete four-input Group 9 are excluded.
Thus §13.34's R-304B-specific gaps are **not** all closed by this implementation.

The normal ventilation trip/SGTS start follows R-104B §4.3.3.2. Existing Level 2
and DW pressure settings remain, since that old section gives no numeric setpoints.
Normal pressure is -0.25 inH2O. 9000 scfm is a per-blower rating, not the unrelated
<12000 scfm integrity-test setting. Door leakage and available flow determine the
achieved vacuum. No room-by-room, source-term/dose, filter-efficiency, HPCI gland
flow or primary-containment leak-to-building model was added.

Verification commands (Windows, project venv; `PYTHONIOENCODING=utf-8`):

```powershell
.venv/Scripts/python.exe -m unittest tests.secondary_contracts -v
.venv/Scripts/python.exe tests/mk_aged.py rated aged
.venv/Scripts/python.exe tests/mutate.py tests/secondary_contracts.py tests/cases_secondary.py --timeout 30
.venv/Scripts/python.exe tests/run_all.py diagram_audit help_test
.venv/Scripts/python.exe tests/run_all.py contracts harness --timeout 300
.venv/Scripts/python.exe tests/run_all.py power_test air_test button_test display_test flow_test gui_test
```

The first command's 20 fixture-free contracts passed (exit 0, 0.41 s test time).
Initial test setup
errors attempted to set derived pressure/voltage properties; fixtures were corrected
to change gas inventory and `power_frac`, without bypassing assertions. The tests
are imported by the existing fast suite; no additional default test program is added.
Final results; recorded source hashes matched before and after each run:

| Check | Result and actual duration | Local evidence |
|---|---|---|
| Final rated / aged fixture generation | exit 0; 424.02 / 146.15 s | source key `c79220cd9264` |
| Local causal mutations | exit 0; 22.43 s; 10 assertion kills + surviving comment control | `.test-results/mutation-1789894123006777800/summary.json` |
| Fast contracts + harness | exit 0; 265 + 37 PASS; 35.98 s total | `.test-results/20260920T085546Z-00d1061b/summary.json` |
| Diagram + help | exit 0; both PASS; 47.92 s | `.test-results/20260920T085340Z-24d96b40/summary.json` |
| Power + air integration and remaining four GUI programs | exit 0; 6/6 PASS; 212.04 s | `.test-results/20260920T085840Z-71035bbb/summary.json` |

The initial combined run (`20260920T084104Z-f1656c0d`) exited 2: an old purge
mutation anchor still named the previous comment, and contracts exceeded 120 s.
The anchor was updated without removing the test. Unnecessary pressure substeps
were replaced by a piecewise analytic mass-balance integration; a new coarse/fine
boundary-crossing contract checks conservation and equal results. The diagnostic
fast retry allowed 300 s locally but actually finished in 35.98 s, within the
unchanged catalog's 120 s budget. No physics assertion, GUI response threshold or
default timeout was relaxed. Runtime variation is not proof that the new pressure
solver caused the original timeout: a 300-step profile measured 0.011 s inside that
solver out of 5.545 s total. Final fixtures were regenerated after the code change.

GUI evidence includes actual radiation/fault/door checkbox and reset-button
invocation, snapshot replacement, high-speed command rejection, text width and
unchanged-value refresh checks. During background rated-run calculation, maximum
Tk update was 0.060 s with 383 event loops. All 63 help keys have buttons/content.
Normal and door-fault panel screenshots were visually inspected locally in
`.test-results/secondary-normal.png` and `secondary-door-fault.png`.

Not run: the other six legacy integration programs, 30-day stress, the five numeric
report programs and NMS's separate full-startup case. The existing fast hydrogen,
containment, protection, snapshot and RBCLCW contracts did run. This is targeted
regression, not a claim of complete plant validation or real drawdown-time accuracy.

### 13.38 Windows v0.1.0-beta portable release (2026-09-20)

This change adds distribution plumbing, not a new plant model. The frozen entry point
opens the same `Panel` and `Reactor`, displays the release version, and records startup
or Tk callback failures under `%LOCALAPPDATA%\BWR4Simulator\logs`. Frozen rated-state
snapshots use the release version as their invalidation key because PyInstaller modules
are not available as ordinary `core/*.py` files. Source checkouts retain the existing
source-content hash and `.state/rated` location.

The release contract run was:

```text
.venv\Scripts\python.exe -m unittest tests.release_contracts -v
5 tests, exit 0

.venv\Scripts\python.exe bwr4_simulator.py --smoke-test
exit 0
```

The smoke path creates a hidden Tcl/Tk root and complete control panel, completes idle
layout work, verifies the versioned window title, destroys the window, and advances a
fresh reactor by one short step. It therefore checks packaged imports, Tcl/Tk resources,
panel construction and basic physics startup; it is not an operator walkthrough.

The artifact was built and checked with:

```text
powershell -NoProfile -ExecutionPolicy Bypass \
  -File .\packaging\build_windows_beta.ps1
exit 0

Expand-Archive BWR4-Simulator-v0.1.0-beta-Windows-x64.zip <empty directory>
<extracted>\BWR4-Simulator.exe --smoke-test
exit 0
```

Result: a 24.80 MiB Windows x64 ZIP, matching SHA-256 sidecar, `0.1.0-beta`
file/product metadata, beta instructions and five third-party notice files. The EXE
passed both the build-directory and clean-extraction smoke checks.

The first fast run correctly failed in the harness because the source-hash mutation
anchor still described the pre-release single-branch snapshot code. The mutation was
rewritten to remove source versioning while leaving valid Python, then the exact final
GitHub Actions gate was rerun:

```text
PYTHONIOENCODING=utf-8 .venv\Scripts\python.exe \
  tests\run_all.py contracts harness --timeout 300
270 contracts + 37 harness tests, 32.96 s, exit 0
evidence: .test-results/20260920T111720Z-fec48be8/summary.json
```

Not run: integration, full GUI profile, numeric reports, NMS full startup or 30-day
stress. Physics equations and control layout did not change. The hidden packaged-panel
check is narrower than the six-case GUI profile and does not validate other Windows
machines, DPI scales, SmartScreen behaviour or graphics drivers; those are beta tester
tasks documented in `BETA_README.md`.

### 13.39 Restricted-license multiplatform v0.2.0-beta.1 release (2026-09-20)

This release-engineering change does not alter plant equations or control-panel layout.
It replaces the Windows-only packaging path with native Windows x64, Linux x64/ARM64,
and macOS Intel/Apple Silicon builds. Each matrix job builds with pinned NumPy and
PyInstaller versions, collects required third-party notices, constructs the complete
Tk panel, advances a fresh reactor one step, and only then uploads its archive and
SHA-256 sidecar. macOS uses an ad-hoc signature and is not Apple-notarized.

The end-user archive copies documentation from an explicit allowlist: the beginner
guide, physics reference, this verification document, and searchable R-104B/R-304B
official-manual material. The project license and required third-party notices are also
present. Change histories, implementation plans, internal test reports, and manual
download/index scripts are absent. The source repository is private. The separate public
distribution repository contains only its README and distribution license as tracked
files, so its GitHub-generated source archives contain no Simulator source. `export-ignore`
also reduces Git archives made from the private source repository, but it is not an
access-control mechanism.

The custom beta license permits personal, non-commercial evaluation and prohibits
unauthorized redistribution, mirroring, sale, modification distribution, and
repackaging. This is a rights notice and contractual deterrent, not technical copy
protection. The verification suite checks that the intended clauses and third-party
carve-outs exist; it cannot determine legal enforceability in a particular jurisdiction.

Local Windows evidence for the preceding `v0.2.0-beta` candidate before the native CI
matrix was:

```text
.venv\Scripts\python.exe -m unittest tests.release_contracts -v
10 tests, exit 0

.venv\Scripts\python.exe packaging\build_beta.py --label Windows-x64
exit 0

Expand-Archive <archive> <new empty directory>
<extracted>\BWR4-Simulator.exe --smoke-test
exit 0
```

The local archive was 26.64 MiB. Its computed SHA-256 matched the sidecar; Windows
file/product version was `0.2.0-beta`; the extracted bundle held `START_HERE.md` plus
109 files under `DOCUMENTATION`, including 107 manual files, and none of the prohibited
development documents. A test `git archive --worktree-attributes` also held zero entries
from the internal-document exclusion set. Python syntax compilation passed.

The final local fast gate was:

```text
PYTHONIOENCODING=utf-8 .venv\Scripts\python.exe \
  tests\run_all.py contracts harness --timeout 300
275 contracts + 37 harness tests, 261.92 s, exit 0
evidence: .test-results/20260920T115805Z-f00fd901/summary.json
```

The first native run, GitHub Actions `35509822523`, passed the fast gate and Windows
x64 build. Both macOS and both Linux jobs built their native executable trees, then
failed before smoke testing because setup-python's Unix layouts did not contain the
`LICENSE`/`LICENSE.txt` path available in the local Windows installation. The build was
kept fail-closed: no incomplete Unix artifact was uploaded. The workflow now fetches
the official CPython `LICENSE` for the exact interpreter patch version, verifies that
it contains the PSF License Version 2 marker, and supplies it as the explicit fallback.

The corrected pre-tag run was:

```text
GitHub Actions 35510205097, commit 89afc1e
fast verification                         PASS, 4m24s
Windows-x64 native build/smoke/upload     PASS, 1m08s
Linux-x64 native build/smoke/upload       PASS, 1m16s
Linux-arm64 native build/smoke/upload     PASS, 0m58s
macOS-x64 native build/smoke/upload       PASS, 1m03s
macOS-arm64 native build/smoke/upload     PASS, 0m39s
workflow exit 0; five non-expired artifacts returned by the GitHub API
```

This proves construction and the automated hidden-panel smoke path on the named hosted
runners. It is still not an interactive operator test on independent user machines.

The final public-distribution run and publication were:

```text
GitHub Actions 35511324305, commit 2f62abb
fast verification                         PASS, 1m59s
Windows-x64 native build/smoke/upload     PASS, 1m08s
Linux-x64 native build/smoke/upload       PASS, 1m08s
Linux-arm64 native build/smoke/upload     PASS, 0m55s
macOS-x64 native build/smoke/upload       PASS, 2m15s
macOS-arm64 native build/smoke/upload     PASS, 0m38s
workflow exit 0; release job skipped as expected for workflow_dispatch
```

The five archives and five SHA-256 sidecars from that run were published as the public
`v0.2.0-beta.1` pre-release in `Loun0521/bwr4-simulator-releases`. GitHub reports the
repository as PUBLIC, the release as non-draft and prerelease, and all ten assets as
uploaded. Every uploaded archive digest matched the corresponding downloaded CI
artifact. Each archive contained 109 allowlisted documentation files plus the Python
license and zero forbidden development documents. An unauthenticated download of the
tag's GitHub-generated source archive contained only `README.md` and `LICENSE.md`; the
private simulator source repository remains separate.

Not run for this packaging-only change: integration, the full six-case GUI profile,
numeric reports, NMS full startup, or 30-day stress. The packaged-panel smoke test is
narrower than those suites and does not prove operation on every graphics driver,
desktop environment, DPI scale, or OS security policy.

## 14. Completeness audit — what the model does not contain at all

Dated 2026-09-18. §9 lists gaps *within* systems the model has; this section
asks the other question — **which systems are absent entirely.**

Update 2026-09-20: RBCLCW and minimal SGTS secondary containment are now present,
with the limitations in §13.36–13.37.
The ranking below is retained as the historical 2026-09-18 selection record.

### 14.1 Method

Two legs are the two revisions' chapter lists. The third was meant to be a
reference simulator's system list, but the IAEA manuals would not fetch (402
and a refused connection) and search returned no enumeration. It was replaced
with something stronger and reproducible: **the manuals' own cross-references.**
Every `(Section X.Y)` in R-304B was counted, which ranks each system by how many
*other* systems name it as an interface partner.

```bash
grep -rhoE "\(Section [0-9]+\.[0-9]+\)" manual/r304b/*.txt | sort | uniq -c | sort -rn
```

That substitution is worth stating plainly: it measures centrality *inside the
reference*, so it cannot see systems the reference itself omits (fire
protection, plant HVAC, instrument nitrogen as a system of its own).

### 14.2 Systems absent entirely, by centrality

| Section | System | Times named | Previously recorded |
|---|---|---|---|
| **11.3** | **Reactor Building Closed Loop Cooling Water** | **18** | implemented 2026-09-20 — §13.36 |
| 8.2 | Liquid Radwaste | 11 | no |
| 8.1 | Offgas | 10 | flow variable only, not as a system |
| 5.6 | Traversing Incore Probe | 10 | no |
| 11.7 | Fuel Pool Cooling and Cleanup | 10 | no |
| 11.8/11.9 | Refueling and Vessel Servicing | 9 | no |
| 4.2 | Secondary Containment | 8 | in passing only |
| 4.3 | RBSVS / Standby Gas Treatment | 8 | no |
| 8.3 | Solid Radwaste | 2 | no |
| 6.3 | Process Computer (R-104B) | — | no |
| 8.4 | Process Radiation Monitoring (R-104B) | — | proxied by `clad_damaged`, §13.20 |

RBCLCW leads by a wide margin, which is consistent with Table 11.3-1: it cools
the recirculation pumps, the RHR pump seals, RWCU, the CRD pumps, the fuel pool
heat exchangers and the drywell unit coolers. Six of those exist in this model
and their auxiliary cooling paths were absent at the time of that audit.
The implemented connections and remaining approximations are now in §13.36.

### 14.3 Isolation groups — four of the manual's seventeen

R-304B §4.4 enumerates Groups 1–17. Implemented: **1 & 14** (main steam),
**5** (RHR shutdown cooling), **8** (RBCLCW and drywell unit coolers, §13.34),
**9** (containment purge and N2 inerting, §13.34).

Absent: 2 (RHR, core spray, instrument air, N2 purge for TIP), 3 & 4 (RWCU),
6 & 15 (RCIC), 7 & 16 (HPCI), 10 (reactor water sample), 11 (drywell
equipment and floor drains), 12 & 13 (RCIC and HPCI vacuum breakers), 17 (post
accident sampling).

### 14.4 Valves and components absent

Reactor head vent (2.5.3.2); main steam line drains; extraction non-return
valves (§13.35, blocked on the missing overspeed model); reactor building to
suppression chamber vacuum breakers (§13.34); RCIC and HPCI vacuum breakers;
drywell equipment and floor drains; condenser water box backwash, which the
manual notes forces a power reduction (11.1.4.1); traveling screens and their
30-inch differential trip (§13.35); feedwater regulating valves, where the
model has a flow limit but no valve.

### 14.5 Present but partial

| System | Modelled | Not modelled |
|---|---|---|
| RWCU (2.8) | blowdown flow, pump load, RBCLCW coupling, NRHX isolation and pump cooling-water trip | filtration, demineralisers, chemistry, remaining Group 3 & 4 signals |
| Offgas (8.1) | `offgas_flow` as a number | recombiners, charcoal delay bed, radiation monitoring |
| Radiation monitoring (8.4) | `clad_damaged` as a proxy cause | actual monitors and setpoints |
| Main turbine trips | four of Table 3.2-1's eleven | overspeed, thrust bearing, lube oil, EHC fluid, vibration, speed feedback |

### 14.6 What was chosen, and what was declined

RBCLCW (§11.3) and secondary containment with RBSVS (§4.2, §4.3) were selected;
the plan is in `PLAN_RBCLCW_SECONDARY.md`. The second pair was chosen despite a
lower ranking because §13.34 recorded two Group 9 signals as unjudgeable without
a secondary containment — the gap blocks work already done.

RBCLCW was implemented on 2026-09-20. On the subsequent instruction, minimal
secondary containment was added using R-104B's SGTS design (§13.37). R-304B's
RBSVS-specific automatic signals and components remain outside that scope.

**Offgas was declined deliberately.** The model has no chemical or radiological
state in water or gas, so recombiners and a charcoal delay bed would change
nothing observable — the same dead-code shape §12.2 removed. It becomes
meaningful only after a chemistry state exists, and that ordering is the point.
