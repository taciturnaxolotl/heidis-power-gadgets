# Heidi's Power Gadgets — Design History

**Non-normative.** Rationale, reversed decisions and errors caught in review. Nothing here is a requirement; `spec.md` is the only normative document. This exists so the same ground isn't re-argued, and so the reasoning behind non-obvious choices survives.

---

## Part changes

| Was | Now | Why |
|---|---|---|
| CH224K PD sink | **HUSB238** | CH224K accumulated eleven workarounds: CFG pins at 8V/4.1V abs max, floating-CFG defaulting to 9V, VBUS pin at 93% of rating, and no source-capability or contract telemetry at all. |
| TLV767 LDO for 3V3 | **AP63203 buck** | An LDO made current and survivability fight each other: 1.7W at 1A from 5V, impossible from 12V. The buck loses ~0.3W and its 32V rating removed the buck as a transient weak link. |
| One LDO for housekeeping + 3V3 | **TPS70933 + AP63203, separate** | A short on the 3V3 output would brown out the supervisor, which then reboots and re-enables into the same short. |
| Discrete P-FET on the 3V3 path | **Second LM73100** | A P-channel body diode conducts drain→source, so a 12V common VOUT walks back into the buck. Flipping it only moves the leak. |
| GATE-driven PMOS input disconnect | **Third LM73100** | Removed the PMOS orientation/VGS/SOA questions *and* the OVLO arbitration problem — a comparator cannot release a gate the HUSB238 is actively pulling low. |
| SMAJ13A → SMAJ24A → **no surge TVS** | ESD diode network (unapproved) | The gate was framed around IEC 61000-4-5 surge, which does not apply to a bus-powered port. Reframed to 61000-4-2 ESD, the inductive and hot-plug cases quantify as non-events. |
| 3A USB-C receptacle | **5A part** | Margin at rated current is the cheapest robustness on the board. |
| HUSB238 `006DD` | **`004DD`** | `006DD` does not exist. `001DD`/`003xx` enable SOP′, which makes the module *emulate* an e-marker and invite >3A through an unrated cable. |

## The CH32X035 fork, closed

The CH224K's workarounds would all have vanished on a CH32X035 with a native PD PHY — but that demanded writing a PD policy engine, and a stack bug ships soldered into someone's project. The HUSB238 took the same deal without the firmware risk: PD stays in silicon, source capabilities are readable over I²C, and the chip is still safe with a dead MCU. The last two properties are why it beat both alternatives.

## Reversed decisions

| Decision | Outcome |
|---|---|
| Delete the long-press lock | **Reinstated** at Kieran's direction. Unlock became the same gesture, which documents better than the original hold-during-power-on. |
| Single amber LED with blink codes | **Four amber + one red** — blink codes judged confusing. |
| Charlieplexing | Dropped when strapping CFG2 freed a pin, then **reinstated** when `BUCK_SENSE` consumed it. Keeps NRST as a hardware reset. |
| MCU-less population | **Dropped.** Output would be live from attach, and OTP renegotiates to 5V with nothing to notice. Freed VSET/ISET to be strapped for safety. |
| Delete D+/D− to dodge the ESD problem | **Kept**, with 100Ω series resistors — these lines carry µA-level DC sensing, not high-speed data. |
| Delete both OVLO dividers to save area | **Reversed.** "The buck output never rises" is circular — it never rises *while the buck works*, and a failed buck is exactly what the 3V3 OVLO catches. Raw-path OVLO kept deliberately as single-fault redundancy. |
| Delete `PG_RAW` as redundant | **Reversed.** It was load-bearing in the state table, fault policy and switching sequence. Deleting it from the BOM while firmware depended on it was an inconsistency, not a saving. |
| Latch on first watchdog reset | **Bounded retry (3× with backoff, 60s decay), then latch.** Latch-on-first bricks a buried module over a transient; unbounded retry loops into a persistent fault at ~100% duty. |
| Keep 3V3 in the Mini | **Confirmed** — it's the rail most host boards need. |
| Host-provided USB-C connector | **Rejected.** Would save 68mm² and ~5mm width, but turns "solder it in and you have power" into "solder it in, add a connector, route CC and VBUS." |

## Errors caught in review

Grouped by root cause, because the pattern is more useful than the list.

### Datasheet values asserted from summarised tool output rather than the primary PDF
| Claim | Correction |
|---|---|
| CFG1 pull-up to VBUS | CH224A guidance misapplied to the CH224K; would have exceeded an 8V abs max on a 12V rail. |
| HUSB238 `006DD` variant exists | It doesn't. Six variants, and no GATE_ON_TIME column — that table was fabricated. |
| VSET is a hardware ceiling | I²C overrides it entirely (datasheet p9). The 12V ceiling is firmware policy plus the input OVLO. |
| `SNK_PDO2_VOLTAGE` is I²C-writable | It's a factory fuse. |
| LM73100 fast trip ≈12A, derived from 350mV/R_DS | **21.9A typical, fixed**, no min/max specified. |
| Unused IMON may be left unconnected | Datasheet requires tying it **directly to GND**. |
| **PGTH rated −0.3 to 28V** | **6V absolute max, 5.5V recommended.** The proposed divider would have exceeded it under external back-drive. |
| Datasheet Rev 2.5 | The mirrored file is **Rev 2.0**. |

**Lesson: for anything protection-critical, the primary PDF wins over a tool query.** Every one of these was caught by reading the actual document.

### Derived results stated confidently without doing the derivation
| Claim | Correction |
|---|---|
| Series resistor fixes the VBUS-pin DC margin | A high-impedance sense pin carries no DC current, so there is no IR drop. |
| Overload trip window = `T×(1±e)` | **`T/(1+e)`** — the error was applied in the wrong direction. |
| Copper qualified for 4.1A | **3.4A**, from the corrected maths at a 2A rating. |
| ADC contributes ±2 LSB | CH32V003 specifies **±6 LSB total deviation**. At a 0.30V signal that is ±6.4%, enough to invalidate the trip window. |
| Nothing bounds 3V3 current in hardware | AP63203 has cycle-by-cycle peak (2.5–3.1A) and valley (2.5–3.9A) limits. |
| Inductor I_SAT > 3.1A peak limit | Derive from the **3.9A valley** maximum plus minimum-on-time rise; ≈5A provisional. |
| LM73100 handles sustained 24V | Recommended operating max is **23V**; 28V is absolute maximum only. Envelope bounded at 20V. |
| 3V3-path OVLO refuses to close into a high rail | OVLO monitors **IN**, not OUT. |
| PG is a universal fault flag | Combined indicator; meaningless before enable, and high-impedance when its own VIN is absent. |
| `BUCK_SENSE` needs no divider | 3.33V buck max against a 3.234V rail minimum violates ADC input ≤ VDD. |
| The transient question is retired by the buck's 32V rating | 32V is a recommended operating ceiling, not system-level transient immunity. |
| TVS + fuse is a fail-safe | Not proven — a current-limited source may never clear the fuse, and a TVS is not guaranteed to fail short. |
| A 47k bleed bounds the disabled-buck node | The 15.9µA figure is the *powered* spec; the unpowered case has no maximum. A Zener doesn't bound it either. |
| 2A rating enforced by a 60ms trip latency | A unit at the error extreme never trips below 3.36A, so the requirement is **continuous**, not latency-bounded. |
| R_IMON bounds the IMON pin by construction | The **clamp** provides the bound; the resistor provides measurement scaling. Gain is uncharacterised above 5.5A, so extrapolation is unsound. |

### Scope and threat-model errors
| Claim | Correction |
|---|---|
| QC fallback covers the missing 12V PDO | HUSB238 has no QC — D+/D− do BC1.2/Apple 5V *current classification* only. |
| D+/D− series resistors 100Ω–1k | **100Ω.** 400Ω is the ceiling; 1k breaks BC1.2 detection outright. |
| IEC 61000-4-5 surge is the threat | It applies to mains and long-line ports. A bus-powered module is a **61000-4-2 ESD** problem. |
| Two-sided assembly is acceptable | A castellated module reflows flat onto the host — bottom-side parts would collide. |
| Legacy 5V/1.5A supports 2A of 3V3 | ≈1.90A at the −5% corner. |
| A 2.0A contract supports a 2.0A load | Housekeeping draws in parallel; requires **≥2.25A**. |
