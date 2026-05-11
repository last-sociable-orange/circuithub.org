---
title: 'Part 3: The Lib Agent — Cleaning Up KiCad Libraries'
description: "Part 3 of the Pi Extensions for Hardware Design series. How I use the lib agent to process downloaded KiCad libraries: renaming, stripping metadata, quality checking, and organizing symbols, footprints, and 3D models."
slug: 'lib-agent-circuitpilot'
pubDate: 'May 11 2026'
heroImage: '../../assets/blog-placeholder-5.jpg'
series: 'pi-extensions-for-hardware-design'
---

## The Library Grind

Every component in my KiCad project needs three files:

- A **symbol** (`.kicad_sym`) — the schematic shape with pin names/numbers
- A **footprint** (`.kicad_mod`) — PCB pad geometry, courtyard, silkscreen
- A **3D model** (`.stp` or `.step`) — for mechanical clearance and board renders

Suppliers like Würth, Samtec, and Hirose provide downloadable library packs. I download a zip, unzip it, and find a pile of files with names like `830108160801.kicad_sym`. Is that a crystal? A connector? No idea without opening it.

Then I have to deal with the contents. Supplier symbols include extra metadata — vendor URLs, long descriptions, alt part numbers — that I don't want in my library. Footprints are missing standard fields. 3D model paths are hardcoded to some engineer's Windows machine at the supplier.

For one component, fine. For 50, it's hours of repetitive editing. The lib agent handles this for me.

## How I Use It

### 1. I Drop a Library Zip Into WIP/

I download a library pack from a supplier — typically a zip like `LIB_830108160801.zip` — and drop it in `WIP/`. That's all I do. I don't unzip it or look inside.

### 2. The Agent Unzips and Identifies

The agent moves the zip to `kicad_lib/.wip/`, unzips it into its own folder, and locates the symbol, footprint, and 3D files by extension:

- `.kicad_sym` → symbol
- `.kicad_mod` → footprint  
- `.stp` or `.step` → 3D model

Then it determines two things. **Product type** comes from cross-referencing the datasheet already in `Knowledge/`. If `Knowledge/` contains `XTAL-830108160801-DS/`, the type is `XTAL`. The **full product number** comes from the zip folder name.

If there's no datasheet in the knowledge base, the agent tells me and asks me to get the doc agent to process it first.

### 3. Files Get Renamed

```
830108160801.kicad_sym → XTAL_830108160801.kicad_sym
830108160801.kicad_mod → XTAL_830108160801.kicad_mod
830108160801.stp       → XTAL_830108160801.stp
```

Now I can see at a glance that this is a crystal, the part number is 830108160801, and which file is which.

### 4. Symbol Cleanup

Supplier symbols carry extra fields I strip out. My symbol keeps exactly four fields:

```
Reference:    CON     — product type (gets replaced in schematic)
Value:        7790    — product number
Footprint:    ""      — blank, set per-instance in schematic
Datasheet:    ""      — blank, linked via database library
Description:  ""      — blank
```

Everything else — Digikey URLs, long descriptions, alt part numbers — gets removed. The symbol file contains exactly one symbol named the same as the file. `CON_7790.kicad_sym` contains one symbol named `CON_7790`.

Why so minimal? Per-instance fields belong in the schematic or the database library. A symbol should represent the function of a component (four pins, op-amp), not a specific supplier's package variant. I set the footprint when I place the symbol on the schematic.

### 5. Footprint Cleanup

Default fields get set to standard values:

```
Reference:    "REF**"              — KiCad default, replaced on placement
Value:        "BAT54L2-TP"         — full product number, set to hidden
Description:  ""                   — blank
Datasheet:    ""                   — blank
Keywords:     ""                   — blank
```

The agent adds one field missing from most supplier footprints:

```
(fp_text user "${REFERENCE}" (at 0 -0) (layer F.Fab) ...)
```

This prints the reference designator on the fabrication layer — essential for assembly, absent from most downloads.

And it fixes the 3D model path to use KiCad's project-relative path variable:

```
(model "${KIPRJMOD}/../kicad_lib/Step/D_BAT54L2-TP.stp"
  (at (xyz ...))
  (scale (xyz 1 1 1))
  (rotate (xyz 90 0 0)))
```

`${KIPRJMOD}` resolves to the KiCad project directory, so this works regardless of where I clone the project.

### 6. Quality Check

Before showing me the files, the agent runs a read-only quality check. It doesn't modify anything — just reports what it found:

| Check | Applies to |
|-------|-----------|
| Pin names and numbers match datasheet | Symbol |
| Pin types correct (in/out/power/passive) | Symbol |
| All pins 100 mil length | Symbol |
| Text 50 mil height/width | Symbol |
| Pad count matches symbol pin count | Footprint |
| SMD pads have solder mask and paste layers | Footprint |
| TH pads have solder mask layer | Footprint |
| Courtyard present | Footprint |
| Pin 1 indicator (polarized parts) | Footprint |
| Polarity indicator (polarized parts) | Footprint |

The agent flags issues but doesn't block — if I approve a library with warnings, that's on me.

### 7. Output

The agent moves everything to review and gives me a report:

```
kicad_lib/.review/LIB_830108160801/
├── XTAL_830108160801.kicad_sym   ✓ passed
├── XTAL_830108160801.kicad_mod   ✓ passed
└── XTAL_830108160801.stp         ✓

Trashed: 830108160801.pdf, README.txt
```

I review, approve, and the files land in their final locations:

```
kicad_lib/Symbol/Symbol/XTAL_830108160801.kicad_sym
kicad_lib/Footprint/Footprint.pretty/XTAL_830108160801.kicad_mod
kicad_lib/Step/XTAL_830108160801.stp
```

Ready to use in KiCad.

## Symbol Library Organization

I organize symbols into two tiers:

```
kicad_lib/Symbol/
├── Standard/
│   └── Standard.kicad_sym       ← shared: R_0402, C_0603, SOT23...
└── Symbol/
    ├── IC_TPS62870.kicad_sym    ← one symbol per file
    ├── IC_TJA1051TK-3.kicad_sym
    └── CON_7790.kicad_sym
```

**Standard components** (resistors, capacitors, common transistors) share symbols in `Standard.kicad_sym`. A 10k 0402 resistor uses the same symbol as a 1k 0402 — they only differ by value.

**Non-standard components** (ICs, connectors, specialized parts) each get their own file with one symbol. This avoids merge conflicts and keeps dependencies clear.

## Limitations

The agent edits KiCad's s-expression format with text replacement. It doesn't parse and re-serialize — a bad edit could produce a file KiCad won't load. The review stage is my safety net.

It doesn't create symbols or footprints from scratch. It cleans and organizes existing ones from supplier downloads.

It doesn't handle KiCad database library entries (`.kicad_dbl` / SQLite). I use [`kicad-lib-gen`](https://github.com/last-sociable-orange/kicad_lib_gen) for that — it's a separate tool that fetches Digikey data and populates the component database.

## Typical Workflow

I'm working on a design with 30 components from 5 suppliers. My process:

1. Download all the library zips, dump into `WIP/`
2. Run the doc agent first to get datasheets into `Knowledge/`
3. Tell the lead agent to process the libraries
4. Review the quality check reports
5. Approve or request fixes
6. Files are in `kicad_lib/`, ready to link in KiCad

The doc agent and lib agent run in parallel when they can — converting datasheets doesn't block library processing, and vice versa.

---

*The Lib Agent is part of [CircuitPilot](https://github.com/last-sociable-orange/pi-agent-team). Read Part 2 for the Doc Agent and Part 4 for the Designer Agent.*
