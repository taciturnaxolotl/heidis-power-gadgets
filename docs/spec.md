# Heidi's Power Gadget series — Build Spec

**Normative.** One requirement and one value per item. Rationale, reversed decisions and review corrections live in [`design-history.md`](design-history.md) and are not requirements.

*A three-tier family of USB-C PD-powered supplies sharing a design language and a block library.*

---

## 0. Product family

| Tier | Status | Context | Form | Anchor silicon | Output |
|---|---|---|---|---|---|
| **Mini** | **Pre-capture** — see §6 | Soldered into a project, permanent | Castellated, ~15×26mm, 4-layer, single-sided | HUSB238 + CH32V003 | 3V3 / 5V / 9V / 12V @ 2A |
| **Midi** | **Concept** | Prototyping / hackathon handout | Breadboard-straddle | HUSB238 (± MCU) | 3V3 + 5V + switched raw |
| **Maxi** | **Architecture concept** | Personal bench instrument | Metal case | Full analog stack | Adjustable CV/CC |

**Reuse:** one PD front-end *block pattern* across all three — same I²C interface, same strap-or-host population choice. HUSB238 (Mini/Midi) and HUSB238A (Maxi candidate) share a vendor and register model. A pattern, not one literal sheet, until each tier's part and population are fixed.

**Design language:** warm/amber, simple retro. Consistent silkscreen and mark across tiers.

**Licensing:** if deriving from CERN-OHL-S or CC BY-SA reference designs, those are copyleft — clean-room anything closed.

---

## 1. MINI — hardware architecture

**Job:** solder it into a project and have correct power on the rail you need, permanently inline, unreachable after install.

### 1.1 Output

| Rail | Source | Rated |
|---|---|---|
| 5 / 9 / 12V | negotiated from the brick, no regulator | **2A** |
| 3V3 | on-module synchronous buck, from whatever rail is negotiated | **2A** |

- **Published claim:** *up to 2A capability; guaranteed current depends on source contract, input voltage, ambient and thermal validation.*
- **Maximum power 24W** (12V × 2A).
- **12V ceiling** enforced by (a) firmware whitelist on `PDO_SELECT`, and (b) the input LM73100's OVLO. **Not** by VSET, which I²C overrides.
- **12V is an optional PD rail.** The menu offers only rails the source advertises with sufficient current, so an absent 12V PDO is never selectable.
- **No high-voltage legacy protocol.** D+/D− provide BC1.2 and Apple-divider **5V current classification** only.

### 1.2 PD sink — HUSB238

**Part: `HUSB238_004DD`** (SOP′ **no**, RDO mismatch → **Request 5V**). Fallback `002DD` (C7471904, in stock) is mismatch → *Next PDO*, which requires firmware to verify the contract after every request. **Never `001DD`/`003xx`** — SOP′ makes the module emulate an e-marker and invite >3A through an unrated cable.

| Pin | Connection |
|---|---|
| VIN | VBUS via 1kΩ, 1µF to GND |
| CC1/CC2 | direct to receptacle (PHY internal, no 5.1k) |
| D+/D− | to receptacle via **100Ω each** |
| SDA/SCL | MCU, 4.7k pull-ups to 3V3 |
| VSET | **0Ω** → 5V |
| ISET | **0Ω** → 1.25A |
| GATE | **unconnected** |

- **VSET/ISET set the autonomous power-on and MCU-dead state only.** I²C overrides both.
- **Absolute maxima:** VIN/GATE 30V · CC1/CC2 25V · **D+/D− 12V** · VSET/ISET/SDA/SCL 6V. ESD ±6kV HBM. θJA 75°C/W.
- **Supply current 3.1mA**, drawn from VBUS — not from the housekeeping rail.
- **D+/D− series resistance is 100Ω.** These lines carry µA-level DC sensing, not high-speed data. The ceiling is 400Ω: BC1.2 primary detection sources ~0.6V and sinks up to 150µA through both resistors plus the charger's ≤200Ω short, against a 300–350mV threshold. 470Ω is marginal; 1k fails.

#### Register map (I²C `0x08`)

| Addr | Register | Contents |
|---|---|---|
| `0x00` | PD_STATUS0 | `[7:4]` contract voltage: 0=unattached, 1=5V, 2=9V, 3=12V, 4=15V, 5=18V, 6=20V · `[3:0]` contract current |
| `0x01` | PD_STATUS1 | `[7]` CC orientation · `[6]` attached · `[5:3]` PD_RESPONSE (0=none, 1=success, 3=invalid, 4=unsupported, 5=no GoodCRC) · `[2]` 5V contract · `[1:0]` 5V current (default/1.5/2.4/3A) |
| `0x02`–`0x07` | SRC_PDO_5V…_20V | `[7]` DETECT · `[3:0]` available current |
| `0x08` | SRC_PDO | `[7:4]` PDO_SELECT (RW) |
| `0x09` | GO_COMMAND | `[4:0]`: `00001` request · `00100` Get_SRC_Cap · `10000` hard reset |

**Current code (both fields) — nonlinear:**

| Code | A | Code | A | Code | A | Code | A |
|---|---|---|---|---|---|---|---|
| `0000` | 0.50 | `0100` | 1.50 | `1000` | 2.50 | `1100` | 3.50 |
| `0001` | 0.70 | `0101` | 1.75 | `1001` | 2.75 | `1101` | 4.00 |
| `0010` | 1.00 | `0110` | 2.00 | `1010` | 3.00 | `1110` | 4.50 |
| `0011` | 1.25 | `0111` | 2.25 | `1011` | 3.25 | `1111` | 5.00 |

**PDO_SELECT encoding is not sequential:** 5V=`0001`, 9V=`0010`, 12V=`0011`, 15V=`1000`, 18V=`1001`, 20V=`1010`.

**Firmware whitelist:** permit only `0001` / `0010` / `0011`. Reject everything else including reserved codes.

#### Chip behaviours firmware must handle
- **OVP** at 1.2× requested voltage, 50µs debounce. Response is switching off an external PMOS — **this design has none, so the chip's OVP has no effect on the load path.** The input LM73100 provides overvoltage disconnect.
- **OTP** autonomously requests 5V regardless of the established PDO, then restores on cooldown. A 12V load can drop to 5V with no firmware involvement.
- **UVP** = requested − 2V, disabled by default.
- **Legacy fallback:** no contract → wait 1.5s → Apple Divider 3, then BC1.2.
- **Dead-battery Rd** present on CC while unpowered.

### 1.3 Power path

```
USB-C ─┬─ ESD (§1.6) ─┬─ HUSB238 VIN
       │              ├─ TPS70933 housekeeping ── 3V3 (MCU, LEDs)
       │              └─ VBUS_SENSE divider
       └─ 1µF ── [LM73100 #1: input, OVLO 13.5V] ── protected bus ─┬─ 22µF
                                                                   ├─ AP63203 buck ─ [LM73100 #3] ─┐
                                                                   └─ [LM73100 #2: raw] ───────────┴─ VOUT
```

**Power-domain split is normative:** the housekeeping LDO, HUSB238 and `VBUS_SENSE` sit **upstream** of the input switch, so an OVLO trip cannot cut the supervisor's power or its ability to report. TPS70933 is 30V-rated and survives a sustained 20V fault.

#### Input switch — LM73100 #1
| Pin | Value |
|---|---|
| OVLO divider | **100k / 9.76k** → 13.5V nominal (13.0–14.0V worst case) |
| PGTH | **tied to 3V3** — makes PG a pure fault flag |
| PG | → `PG_INPUT`, 10k pull-up to 3V3 |
| IMON | **tied directly to GND** (unused) |
| EN/UVLO | ⚠ gate 23 |
| dVdt | ⚠ gate 22 |

Trips in 1.2µs. **Sustained-fault envelope bounded at 20V** — LM73100's recommended operating maximum is 23V (28V is absolute maximum only), and 20V is the highest rail a firmware fault can request.

#### Raw output switch — LM73100 #2
| Pin | Value |
|---|---|
| OVLO divider | **100k / 9.76k** → 13.5V. Single-fault redundancy: acts only if the input switch fails closed. Retained deliberately. |
| PGTH | **tied to 3V3** |
| PG | → `PG_RAW`, 10k pull-up to 3V3 |
| IMON | **2.7k to GND**, BAV199-class low-leakage clamp to 3V3, 2.2k series to ADC, 100nF |
| EN/UVLO | `EN_RAW` via 10k, pull-down, subject to interlock |
| dVdt | ⚠ gate 22 |
| OUT | **Schottky GND→OUT** (OUT abs max −0.3V; inductive host loads drive it negative) |

#### 3V3 output switch — LM73100 #3
| Pin | Value |
|---|---|
| OVLO divider | **23.2k / 10k** → 4.0V. Guards against a failed buck driving its output high. |
| PGTH | **tied to 3V3** |
| PG | unconnected |
| IMON | **tied directly to GND** |
| EN/UVLO | `EN_3V3SW` |
| dVdt | ⚠ gate 22 |
| OUT | **Schottky GND→OUT** |

**Why not a discrete P-FET here:** a P-channel body diode conducts drain→source, so a 12V common VOUT walks back into the buck. Back-to-back FETs plus a level shifter is an LM73100 built from parts. Its OUT is rated 28V and blocks reverse at ~5µA unpowered; a TPS22917-class part tops out at 5.5V and would fail.

**PGTH tied to 3V3** on all three switches makes PG a device-fault indicator (de-asserts on EN low, OVLO, thermal) rather than an output-voltage tracker. One divider cannot serve 5V→20V: it needs ratio ≤4.17 to assert at 5V and ≥3.64 to stay under PGTH's **6V absolute maximum** at 20V back-drive, which is too tight against the 1.183–1.223V threshold spread. Output-voltage verification is done by `VOUT_SENSE`. ⚠ Verify the tie is safe with that switch's VIN = 0V — the housekeeping rail is always on, so the condition is reachable.

#### Housekeeping — TPS70933
30V in, 150mA, SOT-23-5. Always on, upstream of the input switch. Output capacitance limited to **1.5–47µF effective**.

#### 3V3 buck — AP63203
3.8–32V in, 2A, fixed 3.3V, TSOT-23-6, θJA 89°C/W. **L = 3.9µH, I_SAT ≈5A provisional** (from the 3.9A valley-limit maximum plus minimum-on-time rise), **C_OUT = 2×22µF**, C_IN = 10µF. EN floats enabled → **pull-down required**.

- Hardware current bounds are cycle-by-cycle only: peak 2.5–3.1A, valley 2.5–3.9A. **Not a 2A limit.**
- **10k bleed + clamp on the output.** The disabled 3V3 switch leaks into a floating buck output node whose FB pin is rated 6V. ⚠ gate 15.

#### Interlock
`EN_3V3SW` → gate of a small N-FET → drain on the `EN_RAW` node → source to GND. MCU drives `EN_RAW` through **10k series** so the FET always wins.

| `EN_3V3SW` | `EN_RAW` node | Result |
|---|---|---|
| high | forced ~0V | raw off regardless of MCU |
| low | follows MCU | raw under firmware control |
| Hi-Z (reset) | pulled down | both off |

**This is static priority, not bounded break-before-make.** It prevents a steady-state both-on condition; it does not bound transition overlap. **The firmware sequence (§2.1) is the guarantee.**

#### Discharge
150Ω 1W 2010 + N-FET across VOUT, MCU-driven, **hardware-inhibited** by a diode-OR of `EN_RAW` and `EN_3V3SW` into an inhibit FET on the discharge gate.

| Parameter | Value |
|---|---|
| Timeout → fault | 200ms |
| Max host capacitance (published) | **400µF** |
| Firmware pre-check | **refuse to assert discharge above 14V** |
| Qualification energy | **0.26J** (14V into 150Ω for 200ms) |

Without the pre-check, both parts must be qualified at 0.53J / 133mA, since `VOUT_SENSE` is sized for 20V back-drive.

### 1.4 Fail-safe states

Output-off must not depend on firmware starting, so every enable has its safe state set in hardware.

| Node | Requirement |
|---|---|
| `EN_RAW`, `EN_BUCK`, `EN_3V3SW`, `DISCHARGE` | pull-downs |
| AP63203 EN | floats **enabled** → pull-down 10k–47k mandatory |

With no firmware: VBUS sits at the VSET rail (5V), both output switches open, discharge off, load sees nothing.

### 1.5 Sense networks

| Node | Top | Bottom | Full-scale | Note |
|---|---|---|---|---|
| `VBUS_SENSE` | 33k | 5.6k | 20V → 2.90V | **Upstream** of the input switch, sized for the full fault envelope |
| `VOUT_SENSE` | 33k | 5.6k | 20V → 2.90V | Same values — sized for external back-drive |
| `BUCK_SENSE` | 10k | 5.1k | 3.33V → 1.12V | Valid to ~9.6V at minimum VDD (3.234V) |
| `IMON` | — | 2.7k | 2A → 0.98V | + clamp + 2.2k series + 100nF |

All ADC nodes: 100nF at the pin. Verify source impedance against the ADC sampling requirement.

### 1.6 Protection

| Threat | Handled by |
|---|---|
| Sustained overvoltage 13–20V | Input LM73100 OVLO, 1.2µs disconnect |
| Short circuit | LM73100 fast trip, 21.9A typ (no min/max) |
| Sustained overload | Firmware IMON trip (§2.2) |
| Reverse current | LM73100 integrated back-to-back FETs |
| Negative VOUT transient | Schottky GND→OUT on both output paths |
| ESD | ⚠ **gates 4c / 20b — no approved device** |

**ESD status:** ESDA25P35-1U1M (VBUS) and PESD24VS1UB (CC) were evaluated and **both fail alone**. VBUS residual ≈+45V/−16.2V against LM73100 limits of +28V/−15V. CC breakdown starts at 26V minimum against a 25V CC limit. No single device satisfies both "does not conduct at sustained 20V" and "clamps below the protected pin."

**Quantified non-threats**, recorded so they are not re-litigated: a 12V/2A load switched off through ~1µH of cable stores 2µJ → ~2V on 1µF. Hot-plug always starts at vSafe5V, so ringing peaks near 10V. The relevant standard is **IEC 61000-4-2 (ESD)**, not 61000-4-5 surge.

### 1.7 Component values

| Item | Value |
|---|---|
| HUSB238 VIN | 1µF X7R 25V + 100nF, 1kΩ series |
| Upstream bulk | **1µF only** — minimises hot-plug ring energy |
| Protected-bus bulk | 22µF X7R 25V (~11µF effective at 12V) |
| Buck C_IN / C_OUT | 10µF / 2×22µF X7R |
| TPS70933 | 1µF in, 10µF out (within 1.5–47µF effective) |
| LM73100 ×3 | 1µF at IN and OUT each |
| Decoupling | 100nF per IC |
| I²C pull-ups | 4.7k |
| PG pull-ups | 10k |
| LED series | 3× (one per charlieplex pin) |

**Derate every ceramic for DC bias.**

### 1.8 MCU pin map — CH32V003F4P6, TSSOP-20

**16 usable, 16 assigned, no spare.** 24MHz internal HSI.

| Pin | Port | Signal |
|---|---|---|
| 5 | PA1 | `VBUS_SENSE` (ADC A1) |
| 6 | PA2 | `VOUT_SENSE` (ADC A0) |
| 14 | PC4 | `BUCK_SENSE` (ADC A2) |
| 19 | PD2 | `IMON` (ADC A3) |
| 11 | PC1 | `SDA` (I²C1 default) |
| 12 | PC2 | `SCL` (I²C1 default) |
| 15 / 16 / 17 | PC5 / PC6 / PC7 | `LED_A/B/C` charlieplex |
| 13 | PC3 | `BUTTON` |
| 10 | PC0 | `EN_RAW` |
| 8 | PD0 | `EN_BUCK` |
| 20 | PD3 | `EN_3V3SW` |
| 1 | PD4 | `DISCHARGE` |
| 2 | PD5 | `PG_RAW` |
| 3 | PD6 | `PG_INPUT` |
| 18 | PD1 | **SWIO — reserved, never repurpose** |
| 4 | PD7 | **NRST — left at default** |

- I²C uses its **default** mapping. The `SCL_1` remap lands on PD1 and would cost SWIO.
- Per-pin ±25mA absolute max, 8mA recommended. Charlieplex at ~25% duty needs ~8mA instantaneous — **verify brightness at 8mA**. Total VDD limit 80mA.
- Any further signal costs NRST or forces QFN-28.

### 1.9 Housekeeping budget

WCH publish no maximum IDD, only typicals. This budget uses **typical × 1.5**, stated explicitly.

| Item | Budget |
|---|---|
| MCU @24MHz HSI | 7.0mA (4.6mA typ × 1.5) |
| LEDs (charlieplexed, one lit) | 2.0mA |
| I²C pull-ups | 1.4mA |
| PG pull-ups (2× 10k) | 0.66mA |
| **Total** | **≈11mA** |

At 12V in: **96mW**, ~24°C rise even at a pessimistic 250°C/W. 13× margin against the 150mA rating. HUSB238's 3.1mA is **not** in this budget — it draws from VBUS.

### 1.10 UI

- **Tap** cycles 3V3 → 5V → 9V → 12V, **skipping rails the source cannot supply**.
- **Long press (2s)** toggles the lock. Also the **fault-clear gesture** — taps are disabled while locked, so a tap-based clear would be unreachable. Clearing does not unlock.
- **Four amber LEDs** silkscreened `3V3 5V 9V 12V`, one lit at a time. **One red fault LED**, set apart so it never reads as a fifth rail.

| Indication | Meaning |
|---|---|
| amber alone | selected, negotiated, delivered |
| red + amber | the rail went away after selection, or a saved selection is unavailable |
| red alone | protection trip |

- **Saved-but-unavailable selection:** hold output disabled, blink the saved rail's amber, light red. **Never silently substitute.** The selection stays saved and comes up on a capable source without reconfiguration.
- Charlieplexing is invisible to the user: only one amber is ever lit, and red + amber is time-multiplexed at ~1kHz. Mixed forward voltages are safe because the minimum series pair (1.8V + 1.8V = 3.6V) exceeds the 3.3V rail — **this would not hold at 5V**.

### 1.11 Layout

- **~15 × 26mm, four-layer, TOP SIDE ONLY.** A castellated module reflows flat onto the host; bottom-side parts would collide. The bottom layer is uninterrupted ground and thermal copper.
- 0201 passives; ~87 placements.
- Castellations along both 26mm edges (~20 positions each at 1.27mm): **≥3 each for VOUT and GND**, SWIO, GND, plus anchor pads flanking the connector.
- Connector at one short end. USB-C insertion force lands on the module's own joints — anchor pads are structural, not electrical.
- Keep the buck hot loop tight and away from the ADC dividers.
- Group config and power castellations on **different edges**.
- **Programming: SWIO + GND on top-side test pads** — castellations get buried by host routing. Program with USB-C attached so the module self-powers; the programmer supplies SWIO and GND only.

---

## 2. MINI — firmware

### 2.1 Switching sequence

```
COMMON PRELUDE
  disable both paths
  → wait 1ms for turn-off
  → verify VOUT ≤14V before asserting discharge   (else fault)
  → assert discharge
  → verify VOUT < 1.0V                            (200ms timeout → fault)
  → deassert discharge
  → verify no rebound for 10ms

RAW BRANCH (5 / 9 / 12V)
  → write PDO_SELECT, then GO_COMMAND = 00001
  → poll PD_STATUS1[5:3]; treat PD_RESPONSE as advisory
  → CONFIRM PD_STATUS0 reports the rail requested   (mandatory on 002DD)
  → verify VBUS_SENSE within ±5% of expected
  → enable raw LM73100
  → confirm VOUT_SENSE in band AND PG_RAW asserted

3V3 BRANCH
  → qualify the active contract against the 3V3 budget
  → if insufficient, request a suitable PDO over I²C and re-verify
  → if still insufficient → refuse, red LED
  → enable AP63203
  → verify BUCK_SENSE
  → enable 3V3 LM73100
  → confirm VOUT_SENSE ≈3.3V

LEGACY BRANCH (no contract ~2s after attach)
  → read PD_STATUS1: 5V_VOLTAGE must be 1
  → 5V_CURRENT gives the classification
  → permit ONLY the 5V rail
  → qualify 3V3 against the classification
  → enable raw LM73100, confirm VOUT_SENSE and PG_RAW
```

**Completion is judged on attached state, contract voltage/current and stable VBUS under a deadline.** `PD_RESPONSE` freshness and clearing semantics are unproven, so it is advisory only.

**Every PG test must be qualified by VIN-valid.** `PG_RAW` and `PG_INPUT` are only meaningful while that switch's local VIN is present; with VIN absent, PG is high-impedance and reads high through its pull-up.

| Parameter | Value |
|---|---|
| Turn-off wait | 1ms (⚠ based on a 4.5µs *typical*; no maximum published) |
| Discharge threshold | VOUT < 1.0V |
| Discharge timeout | 200ms |
| Rebound check | 10ms |
| Contract settle poll | up to 500ms |
| ADC acceptance band | ±5% |
| PG timeout after enable | 10ms |

**Continuous monitoring:** poll `VBUS_SENSE` and `PD_STATUS0` while running. An unrequested contract change is a fault (HUSB238 OTP can drop the rail to 5V unprompted).

### 2.2 Overload trip

Error stack, worst-case linear: IMON gain −20.4% / +19.3% · R_IMON 1% · ADC reference (LDO tolerance) ~2% · **ADC total deviation ±6 LSB** = ±2.0% at the 0.98V operating point · clamp leakage ~0.1%. **Total ≈ +24% / −25.5%.**

The **clamp** bounds the IMON pin; **R_IMON provides measurement scaling.** Sizing alone cannot bound the pin, because IMON gain is uncharacterised above 5.5A while the fast trip is 21.9A typical with no maximum.

Firmware trips when the *reading* reaches T, so actual trip = `T/(1+e)`:

```
earliest = 2.5 / 1.24  = 2.016A
latest   = 2.5 / 0.745 = 3.356A
```

| Rated | Nominal trip | Published window | Copper qualified |
|---|---|---|---|
| **2A** | **2.50A** | **2.00 – 3.36A** | **≥3.4A** |

- **3.4A is a CONTINUOUS requirement.** A unit at the error extreme never trips below 3.36A, so a stable source sustains it indefinitely. Qualify castellations, copper, connector and both series LM73100s **at maximum R_ON (44.85mΩ)** — 0.36W at 2A (179mV drop), 0.97W at 3.36A. ⚠ gate 1b.
- **Contract current qualifies or refuses; it never raises the trip.**
- **Source-limited faults:** `VBUS_SENSE` below band for >100ms while enabled → fault. ⚠ gate 6b for thresholds.
- Latency target: detect ≤50ms, output off ≤60ms. Latency bounds the thermal excursion, not the rating.

### 2.3 Source qualification

`available_3V3 = (V_min × I_advertised × η_min − P_internal) / 3.3V`, using minimum source voltage (−5%), advertised current, and internal consumption.

**Raw 2A requires a contract of ≥2.25A** — housekeeping and the HUSB238 draw in parallel with the load, so a 2.0A contract cannot supply a 2.0A output. On a 2.0A contract, **publish and refuse above 1.75A**; the module has no programmable limiter and cannot throttle.

| Legacy `5V_CURRENT` | Classified | 3V3 available (4.75V corner) | Verdict |
|---|---|---|---|
| `00` | 500mA | ≈0.61A | refuse 2A |
| `01` | 1.5A | ≈1.90A | refuse 2A |
| `10` | 2.4A | ≈3.1A | 2A ✓ |
| `11` | 3A | ≈3.9A | 2A ✓ |

The same power-based rule applies to PD contracts — a 9V or 12V PDO may advertise as little as 0.5A. Qualify on `V × I`, never on voltage alone.

⚠ η_min is assumed 0.9; no guaranteed worst-case efficiency bound is established.

### 2.4 Fault policy — single normative table

*Every other reference to latching, retrying or recovery defers to this table.*

| Fault | Detected by | Immediate action | Retry | Clear |
|---|---|---|---|---|
| Overload | IMON > trip, ≤50ms | disable output | 3× | long press |
| Short | PG de-assert + VOUT low | switch already latched | 3× via EN toggle | long press |
| OVLO (input) | `PG_INPUT` low + `VBUS_SENSE` high | switch already open | auto — OVLO self-recovers on hysteresis; firmware re-verifies before enabling | — |
| Thermal | PG de-assert | switch already latched | 3× (5s backoff gives cooldown) | long press |
| `CONTRACT_LOST` | PD_STATUS0 changed unrequested | disable output | 3× | long press |
| Source sag | `VBUS_SENSE` below band >100ms | disable output | 3× | long press |
| Discharge timeout | VOUT high after 200ms | abort, stay off | no | long press |
| Watchdog hang | IWDG reset cause on boot | output off via pull-downs | 3× consecutive | long press |
| Contract insufficient | qualification | refuse to enable | n/a — a refusal, not a fault | re-plug / reselect |

**Rules for every row:**
- Retry budget is **shared and consecutive**, backoff 100ms / 1s / 5s, counter in `.noinit` RAM behind a magic word.
- Counter **decays to zero after 60s** of stable operation, so unrelated faults never accumulate.
- **4th consecutive attempt latches** — red LED, output off.
- **A latch does not survive a power cycle.** `.noinit` is lost on true power removal, so re-plugging gives a fresh budget. Deliberate: unplugging is the one recovery every user can perform. A persistent fault re-latches within three attempts.
- **Watchdog:** enable IWDG, kick **only from the main loop**, period well under the copper's thermal time constant at 3.4A. Read the reset-cause flag on boot.

### 2.5 Persistence

Two **page-aligned** 64-byte pages, four 16-byte records each, copy-forward.

| Offset | Field | Width |
|---|---|---|
| 0 | `seq` | u16 LE |
| 2 | `voltage` | u8 (0=3V3, 1=5V, 2=9V, 3=12V) |
| 3 | `lock` | u8 |
| 4 | reserved (0xFF) | 8B |
| 12 | `crc16` | u16 — CRC-16/CCITT-FALSE over offsets 0–11 |
| 14 | `commit` | u16 — erased 0xFFFF, written 0x0000 **last** |

- Valid ⇔ commit == 0x0000 **and** CRC matches **and** every field in range.
- Newest ⇔ greatest `seq` by signed 16-bit modular compare `(int16_t)(a−b) > 0`.
- **Copy-forward:** page A full → write and commit into page B → *then* erase A. A valid record exists at every instant.
- Any invalid or torn read → **5V, unlocked, output disabled**.
- Save on lock and on selection settling ~1–2s. Never per tap. Flash endurance 10k min / 80k typ.
- ⚠ gate 19: exact WCH erase/program calls, linker reservation, interrupts disabled with IWDG kicked immediately prior, minimum programming VDD (2.8V), read-back verification. A 16-bit program is *short*, not atomic against power loss.

---

## 3. MINI — acceptance criteria

*The user-facing datasheet. **TBD** entries require bench measurement (§6 group D).*

| | 3V3 | 5V | 9V | 12V |
|---|---|---|---|---|
| Source prerequisite | contract ≥2.25A, or legacy ≥2.4A | contract ≥2.25A | 9V PDO ≥2.25A | 12V PDO ≥2.25A |
| Output voltage | 3.3V ±TBD% | source rail less path drop | " | " |
| Rated continuous | up to 2A, contract-limited | 2A | 2A | 2A |
| Overload trip nominal | qualify-and-refuse only | 2.50A | " | " |
| Trip window | n/a | 2.00 – 3.36A | " | " |
| Path drop @2A (max R_ON) | regulated | 179mV | " | " |
| Ripple | TBD — specify at 20MHz BW | passes source ripple | " | " |
| Turn-on ramp / overshoot | TBD (gate 22) | TBD | " | " |
| Load-step response | TBD | TBD | " | " |
| Max capacitive load | 400µF | 400µF | " | " |
| Reverse-drive | blocked; Schottky fitted | same | " | " |
| Ambient derating | 2A to ~70°C (vendor curve — validate on our copper) | TBD | " | " |

**Also published:** maximum 24W · **not a guaranteed 3A device** · faults auto-retry 3× then latch until a long press · a latch clears on power cycle.

---

## 4. MIDI — concept

- Regulated 3V3 + 5V + switched raw PD on a clearly labelled header.
- Breadboard-straddle, slide switch per rail, test points.
- Current limiting, status LEDs, silkscreen pinout + QR.
- Two populations of one layout: **dumb** (HUSB238 + VSET strap) and **smart** (+ MCU over I²C, INA226, small readout).

---

## 5. MAXI — architecture concept

**Goals:** one channel done extremely well — clean, accurate, scriptable. Adjustable CV/CC + 4-wire remote sense. Amber/retro swappable front panel. USB-C + BLE + WiFi. Metal case.

```
Wide input → Pre-reg (tracking buck) → Linear pass → Output (relay + sense)
                    ▲                        ▲              │
                    │ DAC setpoints          │ analog CV/CC │ force/sense
                 Control MCU ←──── ADC readback ────────────┘
                    │ SPI
                 Comms MCU (ESP32-S3)
```

**Core principle:** the fast regulation loop is analog and firmware-independent. MCUs set references and read back, never sitting inside the loop. Default state of everything is disconnected.

### Blockers before any part selection
- **Buck vs buck-boost.** 9–36V in cannot make 30V out. Constrain V_out,max as a function of V_in, raise minimum V_in, or accept buck-boost.
- **Input ORing.** PD sink and DC jack are **not** simply parallel — needs ideal-diode ORing, defined priority, backfeed limits.
- **Wide input is not yet decoupled from PD wattage** — a PD-fed Maxi remains bounded by contract wattage until the numeric envelope exists.
- **Rated output current, power, short-circuit duration, ambient and cooling envelope** — none stated.
- **Floating vs earth/input/USB-referenced output** — decides whether the power path and comms need isolation, and changes everything downstream.
- **Pass-device DC SOA** and tracking-failure clearing time.
- Quantitative bounds: noise bandwidth, accuracy, transient response, remote-sense compensation, reverse-energy tolerance, OVP/OCP.
- **"Relay = true disconnect" is overstated** — sense lines, bleed resistor, ESD devices and semiconductor paths remain connected. Enumerate what stays attached.

### Test plan (once built)
Short output at max current · reverse-connect · overvolt past armed OVP · yank a sense lead · force 0V with tracking inhibited · hang firmware · power-cycle. Confirm each response and fault indication.

---

## 6. Gate table — the single authority

Status is derived only from this table. An open gate in group C, D or E does not block schematic capture; an open gate in group A does.

### A. Schematic-capture blockers
| # | Gate | Status |
|---|---|---|
| 1 | Overvoltage path | ✅ third LM73100, OVLO 13.5V |
| 2 | Input disconnect topology | ✅ no PMOS |
| 3 | OVLO arbitration | ✅ one disconnect element |
| 8 | Interlock circuit | ✅ N-FET static inhibit |
| 9 | MCU-less population | ✅ deleted — MCU-only |
| 12 | ADC / IMON networks | ✅ dividers and clamp topology specified |
| 17 | MCU pin map | ✅ 16 of 16, no spare |
| 18 | Discharge inhibit circuit | ✅ diode-OR inhibit |
| **4c** | **CC ESD network** | 🔴 **OPEN** — no device satisfies both constraints. Choose: native 25V rating with nothing fitted (documented risk acceptance), or two-stage with series impedance (CC tolerates ~100Ω; Rd is 5.1k, BMC ~300kHz). |
| **20b** | **VBUS ESD network** | 🔴 **OPEN** — residual exceeds LM73100 on both rails. Needs a bounded connector-to-IC transfer analysis covering HUSB238 VIN, TPS70933 and the ADC path, positive **and** negative. |
| **22** | **LM73100 inrush / dVdt** | 🔴 **OPEN** — no `C_dVdt` for any of the three switches; reconcile with the 10ms PG timeout and linear-mode SOA. |
| **23** | **Housekeeping hold-up + brownout** | 🔴 **OPEN** — max VBUS interruption, hold-up capacitance, BOR config, restart behaviour. A source collapse can reset the MCU before the 100ms sag timer, and `.noinit` does not survive power loss. |

### B. BOM-freeze blockers
| # | Gate | Status |
|---|---|---|
| 11 | Capacitors | ✅ specified, DC-bias derated |
| 14 | Area envelope | ✅ ~15×26mm, 4-layer, top-only |
| **1b** | **Continuous 3.4A path qualification** | 🔴 **OPEN** — copper, connector, castellations, both switches at max R_ON, continuously — or fit an eFuse |
| **15** | **Disabled-buck leakage** | 🔴 **OPEN** — no guaranteed off-state leakage maximum; a Zener does not bound it either |
| **13b** | **Discharge FET + clamp selection** | 🔴 **OPEN** — exact parts from pulse and SOA curves at 0.26J |
| **5b** | **Inductor I_SAT** | 🟡 ≈5A provisional; needs guaranteed minimum-on-time data |

### C. Firmware-release blockers
| # | Gate | Status |
|---|---|---|
| 6 | Overload trip values | ✅ 2A rated, 2.50A nominal, 2.00–3.36A |
| 7 | 3V3 qualify-and-refuse | ✅ decision taken |
| 21 | Legacy 5V branch | ✅ thresholds at the −5% corner |
| 4d | 3V3 cold-boot renegotiation | ✅ may request a better PDO |
| 10b | Fault / lock UX | ✅ unavailable-selection boot, long-press clear |
| **6b** | **Source-limited fault thresholds** | 🟡 rule stated; values to write |
| **19** | **Flash implementation contract** | 🟡 architecture settled; WCH API, linker, interrupts, min-VDD outstanding |

### D. Prototype qualification (never blocks capture)
ESD residual at every protected pin · buck thermal on final copper at 5/9/12V in · hot-plug transient · discharge into 400µF and into a live host · MCU current at 24MHz · acceptance-table measurements (ripple, ramp, overshoot, load step).

### E. Product release
All acceptance TBDs measured and published · all group D tests passed · HUSB238 Rev 2.5 obtained, diffed against Rev 2.0, normative reference re-pinned.

---

## 7. Normative references

- **HUSB238 Data Sheet Rev 2.0** (01/2021) — [mirror](https://cdn-learn.adafruit.com/assets/assets/000/125/150/original/husb238_datasheet_full.pdf). All datasheet citations refer to this revision. ⚠ Rev 2.5 is advertised; obtain and diff (gate E).
- **HUSB238 Register Information Rev 1.1** — [SparkFun](https://cdn.sparkfun.com/assets/0/9/8/a/a/USB_PD_Sink_HUSB238_Registers.pdf) · [Hynetek](https://en.hynetek.com/uploadfiles/site/219/news/eb6cc420-847e-40ec-a352-a86fbeedd331.pdf)
- Driver references — [Adafruit_HUSB238](https://github.com/adafruit/Adafruit_HUSB238) · [CircuitPython](https://github.com/adafruit/Adafruit_CircuitPython_HUSB238) · [ltyridium/HUSB238-lib](https://github.com/ltyridium/HUSB238-lib)

---

## 8. Parts

| Role | Part | LCSC | Package |
|---|---|---|---|
| PD sink | HUSB238 `004DD` (`002DD` fallback) | C7471904 (002DD) | DFN-10 3x3 |
| MCU | CH32V003F4P6 | C5187096 | TSSOP-20 |
| Housekeeping LDO | TPS70933DBVR | C89347 | SOT-23-5 |
| 3V3 buck | AP63203WU-7 | C780769 | TSOT-23-6 |
| Switches ×3 | LM73100RPWR | C3210761 | VQFN-10-HR 2x2 |
| USB-C, 5A | TYPE-C-31-M-12 | C165948 | SMD |
| IMON clamp | BAV199-class low-leakage | TBD | SOT-23 |
| Schottky ×2 | GND→OUT clamp | TBD | SOD-323 |
| Discharge FET + 150Ω 1W | SOA-selected | TBD | SOT-23 + 2010 |
| Interlock / inhibit FETs | dual N-FET ×2 | TBD | SC-70-6 |
| Inductor | 3.9µH, ≥2.7A DC, I_SAT ≈5A | TBD | shielded |
| VBUS / CC / D+D− ESD | 🔴 unapproved — gates 4c, 20b | — | — |
| LEDs | 4× amber, 1× red, matched Vf ≥1.8V | TBD | 0402 |

**Other tiers:** HUSB238A (C24833806, QFN-16, EPR 48V/5A, PPS) · INA226 (C49851) · TPS55288 / MP28167 buck-boost.
