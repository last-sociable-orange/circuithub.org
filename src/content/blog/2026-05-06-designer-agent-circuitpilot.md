---
title: 'Part 4: The Designer Agent — Circuit Calculations With a Paper Trail'
description: "Part 4 of the Pi Extensions for Hardware Design series. How I use the designer agent to run circuit calculations, cross-reference datasheets, and write design documents with traceable references."
slug: 'designer-agent-circuitpilot'
pubDate: 'May 17 2026'
heroImage: '../../assets/blog-placeholder-5.jpg'
series: 'pi-extensions-for-hardware-design'
---

## Why I Write Design Documents

A design document records why I made each choice. Why 4.7 µH and not 2.2 µH for the inductor. Where the 20% margin came from. Which page of the datasheet says the switching frequency is 2.2 MHz.

Without this, a design review is "trust me." Six months later, even I won't remember the reasoning. If I need to respin the board with a different output voltage, I have to re-derive everything from scratch.

The designer agent does the calculations and writes the document, with page-numbered references back to the datasheets in `Knowledge/`. I review the math and sign off. The paper trail is built in.

## How I Use It

The designer agent reads Markdown datasheets (produced by the doc agent), collects design specs from me, runs the calculations, validates against datasheet limits, and writes the document. Here's the full flow.

### 1. Check What's Available

First, the agent reads `Knowledge/knowledge.md` — the one-line-per-document index the doc agent maintains. This tells it what datasheets, user manuals, and app notes are available. If the datasheet I need isn't there, it tells me to get the doc agent to process it first.

### 2. Collect Specifications

The agent doesn't jump into math. It asks me for the full design spec first. For a buck converter:

- Input voltage range (min, nom, max)
- Output voltage
- Output current (max, typical)
- Switching frequency (or "use datasheet default")
- Efficiency target
- Ripple target (output voltage, inductor current)
- Transient response requirements
- Constraints (board area, height, cost)

It won't proceed until I've answered everything. I can skip a field by saying "use typical values from the datasheet," but the agent makes me acknowledge each parameter. This catches missing specs before I'm knee-deep in math.

### 3. Read the Relevant Datasheet Sections

With specs in hand, the agent pulls the datasheet sections it needs. Because the datasheet is Markdown (not PDF), it can search for specific parameters — "feedback voltage," "maximum duty cycle," "inductor selection" — and extract only the relevant sections.

For a buck converter, it typically reads:
- Electrical characteristics (Vin range, Vfb, Iq, Rds(on))
- Typical application schematic
- Inductor selection guidelines
- Output capacitor selection
- Feedback/resistor divider network
- Layout recommendations

Each piece becomes a potential reference in the design document.

### 4. Run the Calculations

Standard design formulas based on the datasheet. For a buck converter:

```
D = Vout / (Vin × η)                   # duty cycle
ΔIL = (Vin - Vout) × D / (fsw × L)    # inductor ripple current
IL_peak = Iout + ΔIL/2                 # peak inductor current
Vout_ripple = ΔIL / (8 × fsw × Cout)   # output ripple
```

Feedback network:

```
R2 = Vfb × R1 / (Vout - Vfb)
```

The agent selects standard component values (E12/E24 for resistors, standard inductor values) where applicable.

### 5. Validate Against Datasheet Limits

Every result gets checked against absolute maximum ratings and recommended operating conditions, with at least 20% margin:

| Parameter | Calculated | Limit | Margin |
|-----------|-----------|-------|--------|
| Peak inductor current | 2.1 A | 3.5 A (Ilim min) | 40% |
| Output voltage ripple | 12 mV | 50 mV (target) | 76% |
| Feedback divider accuracy | 0.798 V | 0.8 V ± 1% | Within tolerance |

If anything fails the margin check, the agent flags it and suggests alternatives — different inductor value, larger capacitor, different frequency. It won't silently produce a marginal design.

### 6. Write the Document

The agent produces a structured document in MyST Markdown. Three things matter:

**Blockquotes from the datasheet.** Every factual claim cites the source:

```markdown
> The TPS62870 operates with a switching frequency of 2.2 MHz typical,
> adjustable from 1.8 MHz to 4 MHz via the FSET pin.
>
> — IC-TPS62870-DS, Electrical Characteristics, page 5
```

**Cross-reference links** to the knowledge base:

```markdown
See [IC-TPS62870-DS](../Knowledge/IC-TPS62870-DS/IC-TPS62870-DS.md#electrical-characteristics)
for full operating conditions.
```

**Calculation steps with formulas:**

```markdown
### Output Voltage Setting

Target: Vout = 1.8 V

Feedback voltage Vfb = 0.8 V (from datasheet, section 7.5).
Formula from section 9.2.2:

$$ R_2 = V_{fb} \times R_1 / (V_{out} - V_{fb}) $$

With R1 = 100 kΩ (recommended):

$$ R_2 = 0.8 \times 100\text{k} / (1.8 - 0.8) = 80\text{ kΩ} $$

Nearest standard value: **80.6 kΩ** (E24 series).
```

### 7. Review

The document lands in `Document/.review/`. I check the math, verify the datasheet references, flag anything that looks off. If I ask for changes, the agent moves the file back to `Document/.wip/`, revises, and returns to `.review/`. When I approve, it moves to `Document/`.

## A Real Session

Here's what a typical interaction looks like:

> **Me**: Design a 1.8V 2A supply from a 5V rail using the TPS62870.

> **Lead agent** (delegates to designer): Checking the knowledge base... Found IC-TPS62870-DS. A few questions:
> - Input voltage range: 5V nominal. Min and max?
> - Switching frequency: use the 2.2 MHz default, or a different value?
> - Output ripple target?
> - Any space/height constraints?

> **Me**: 4.5V to 5.5V input. Use the default 2.2 MHz. Ripple under 30 mV. No special constraints.

> **Designer agent**: Working through the calculations...

The output document includes:
- Inductor: 4.7 µH, Isat > 2.3A
- Output capacitors: 2× 22 µF ceramic
- Feedback resistors: 100k / 80.6k
- Efficiency estimate: ~92%
- Layout checklist (quoted from datasheet layout section)
- Margin table against datasheet limits
- Page-numbered references for every value

## What the Designer Agent Doesn't Do

It doesn't draw schematics. I still place symbols and connect nets in KiCad.

It doesn't do layout. It might tell me to keep the feedback trace short and away from the switch node (from the datasheet layout guidelines), but it doesn't route anything.

It doesn't replace judgment. It follows datasheet formulas and checks margins. If a design is marginal or the datasheet doesn't cover my use case, the agent won't catch it — I will.

---

*The Designer Agent is part of [CircuitPilot](https://github.com/last-sociable-orange/pi-agent-team). Read Part 2 for the Doc Agent and Part 3 for the Lib Agent. Part 5 covers the subagent extension that powers all three.*
