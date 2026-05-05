---
title: 'Extending KiCad with Python Scripting for Team Workflows'
description: 'Practical techniques for automating KiCad tasks — BOM generation, design rule checking, and version control integration — for design teams.'
pubDate: 'Mar 28 2026'
heroImage: '../../assets/blog-placeholder-2.jpg'
---

KiCad's Python API (`pcbnew`, `eeschema`) is one of its superpowers. For teams designing hardware collaboratively, scripting can turn a chaotic manual process into a repeatable pipeline.

## Why Automate KiCad?

If you've ever:
- Manually cross-referenced a 200-line BOM against distributor stock
- Spent an hour checking that every net in a 4-layer board has proper clearance
- Tried to diff two revisions of a complicated schematic in git

…you know why automation matters. Let's look at practical scripts that help teams move faster.

## 1. Automated BOM Generation with Distributor P/Ns

The built-in BOM generator is fine for hobby projects. For production, you want distributor part numbers, lifecycle status, and pricing.

```python
import pcbnew

board = pcbnew.LoadBoard("project.kicad_pcb")
components = board.GetFootprints()

bom = []
for comp in components:
    ref = comp.GetReference()
    value = comp.GetValue()
    # Access custom fields stored in the footprint
    mfr_pn = comp.GetFieldByName("MPN")
    distributor_pn = comp.GetFieldByName("DIGIKEY_PN")
    bom.append({
        "ref": ref,
        "value": value,
        "mpn": mfr_pn,
        "digikey": distributor_pn,
    })
```

Hook this into a CI pipeline and you can auto-generate a procurement-ready BOM on every commit.

## 2. Custom Design Rule Checking (DRC)

KiCad's built-in DRC is excellent, but sometimes you need team-specific rules:

```python
def check_trace_angles(board):
    """Flag any trace with an acute angle (< 45°) as a manufacturing risk."""
    for track in board.GetTracks():
        if track.GetClass() == "PCB_TRACK":
            # Check angle between consecutive segments
            # ... geometry calculation ...
            if angle < 45:
                print(f"Warning: acute angle at {track.GetStart()}")
```

Run this as a pre-commit hook to catch issues before they land on the main branch.

## 3. Gerber Validation & Comparison

Teams often need to verify that generated Gerbers match the PCB design exactly:

```python
# Load Gerber and compare layer stack
# Check that all layers are present and named consistently
# Validate aperture assignments against a team standard
```

This is especially useful when onboarding new team members who may not have the exact KiCad version.

## Setting Up a KiCad CI Pipeline

Here's a practical GitHub Actions workflow:

```yaml
name: KiCad CI
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run DRC
        run: python scripts/drc_check.py
      - name: Generate BOM
        run: python scripts/generate_bom.py
      - name: Export Gerbers
        run: python scripts/export_gerbers.py
      - name: Validate
        run: python scripts/validate.py
```

## Key Takeaways

1. **Start small** — automate the one thing that wastes the most time each week
2. **Standardize fields** — agree on custom field names (MPN, DATASHEET, DISTRIBUTOR_PN) across the team
3. **Version control everything** — scripts, templates, and DRC rules all belong in the repo
4. **Review automation outputs** — never blindly trust a script; validate the first few runs

KiCad scripting transforms it from a single-user EDA tool into a platform for team-scale hardware development. If you haven't explored `pcbnew` yet, this week is a great time to start.
