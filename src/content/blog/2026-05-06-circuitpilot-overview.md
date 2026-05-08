---
title: 'Part 1: CircuitPilot — A Multi-Agent System for Hardware Design'
description: "Part 1 of the Pi Extensions for Hardware Design series. An overview of CircuitPilot: how it's structured, and the staged workflow that keeps projects coherent."
slug: 'circuitpilot-overview'
pubDate: 'May 06 2026'
heroImage: '../../assets/blog-placeholder-5.jpg'
series: 'pi-extensions-for-hardware-design'
---

## How I Got Here

I design hardware. Each project means downloading datasheets and user manuals from suppliers, processing downloaded KiCad symbol/footprint libraries from Digikey or Mouser, searching how a pin is used in a thousands of pages' user manual, running design calculations for every subcircuit, and writing documentation so I can remember why I chose a 4.7 µH inductor six months later.

None of this is the fun part. Most of the work is essential but repetitive.

AI is changing all of this. Using language models for hardware design cuts datasheet reading from hours to minutes, and generates design docs that don't need rewriting. The rise of [OpenClaw](https://openclaw.org/) opens the door how AI agents interact with EDA tools and automate workflows that are hard to do with one-off scripts.

So I built [CircuitPilot](https://github.com/last-sociable-orange/pi-agent-team), a pi extension that splits the work across specialized sub-agents.

## The Setup

CircuitPilot uses pi's subagent extension to run a lead agent that delegates to three specialized agents:

```
I talk to → Lead Agent
              │
              ├── doc agent       → converts PDFs to Markdown
              ├── lib agent       → manages KiCad symbol/footprint/3D libraries
              └── designer agent  → runs circuit design and writes docs
```

A lead agent acts as a program manager — it receives my request, breaks it down, and delegates to the right sub-agent. The doc agent handles every PDF: it identifies the part, renames the file, and converts the datasheet to Markdown so language models can actually read it. The lib agent manages the KiCad library: it unzips supplier downloads, renames symbols and footprints to match project conventions, strips extra metadata, and runs integrity checks — pin mapping, pad mask layer, paste layer existence. The designer agent reads the datasheet, runs the circuit calculations, and writes a complete design document with every decision referenced back to the original page number. I check the doc, sign off, and move on. 

## Why Three Agents Instead of One

I tried doing everything with one agent first. Three problems kept coming up:

**Context pollution.** Converting a datasheet spews page chunks, image paths, and OCR cleanup into the conversation. By the time it finishes, the agent has forgotten the design spec I gave it earlier. Separate processes fix this.

**Tool permissions.** The doc agent needs `bash` to run `pdf_to_markdown.py`. The lib agent needs to edit KiCad's s-expression files directly. The designer needs `read` for datasheets and `write` for design docs. With one agent, every tool is available all the time — more surface area for mistakes. Per-agent tool lists mean the designer can't accidentally overwrite a KiCad symbol.

**Model selection.** PDF conversion is mechanical — a fast, cheap model works fine. Design calculations need multi-step reasoning — a stronger model is worth the cost. I use `opencode-go/qwen3.6-plus:medium` for the doc agent, `deepseek/deepseek-v4-pro:high` for the designer, and `deepseek/deepseek-v4-flash:high` for the lib agent. One agent, one model choice.

## How I Define an Agent

Each agent is a Markdown file with YAML frontmatter.

```markdown
---
name: doc
description: Extracting text and images from pdf, converting to markdown
tools: read, write, edit, bash
model: opencode-go/qwen3.6-plus:medium
---

# Doc Agent

## Overview
You are document agent assisting user to...
```

The frontmatter sets the name, description, tools, and model. The body is the system prompt. Files go in `.pi/agents/` (per-project) or `~/.pi/agent/agents/` (global). The extension picks them up automatically — no registration step.

## The Staged Workflow

Every file in my project moves through four stages, enforced by directory:

| Stage | Where | Meaning |
|-------|-------|---------|
| Unprocessed | `WIP/` | I just downloaded it |
| WIP | `<dir>/.wip/` | Agent is working on it |
| Review | `<dir>/.review/` | Agent is done, I need to check |
| Approved | `<dir>/` | I signed off, it's in production |

Agents never delete files — unwanted stuff goes to `.trash/`. If I ask for revisions, the agent moves files from `.review/` back to `.wip/`, makes changes, and returns them. I always know what state everything is in.

## Project Layout

My projects follow this structure:

```
Project/
├── AGENTS.md              # Lead agent system prompt
├── .pi/
│   ├── SYSTEM.md          # File conventions, naming rules, workflow stages
│   ├── agents/            # Sub-agent definitions
│   └── extensions/
│       └── subagents/     # The subagent tool extension (TypeScript)
├── CAD/                   # KiCad project (schematics, PCB)
├── Document/              # Design documents I write (or the designer writes)
├── Knowledge/             # Datasheets converted to Markdown
│   ├── knowledge.md       # One-line index of every document
│   └── IC-TPS62870-DS/   # One folder per document
├── Datasheet/             # Original PDF files
├── kicad_lib/             # KiCad libraries
│   ├── Symbol/Symbol/     # One symbol per file for non-standard parts
│   │   └── Standard/      # Shared symbols for passives, discretes
│   ├── Footprint/Footprint.pretty/
│   └── Step/              # 3D models
└── WIP/                   # Where I drop downloads
```

Each agent knows its output directory. The doc agent writes to `Knowledge/` and `Datasheet/`. The lib agent writes to `kicad_lib/`. The designer writes to `Document/`.

## Naming Rules

Everything follows a strict naming convention so I can grep, script, and cross-reference:

**Documents:** `<PRODUCT_TYPE>-<PRODUCT_NUMBER>-<DOCUMENT_TYPE>.pdf`
- `IC-TPS62870-DS.pdf` — datasheet
- `IC-MIMXRT1170-UM.pdf` — user manual
- `IC-TPS62870-AN.pdf` — application note

**Library files:** `<PRODUCT_TYPE>_<FULL_PRODUCT_NUMBER>.<ext>`
- `XTAL_830108160801.kicad_sym`
- `D_BAT54L2-TP.kicad_mod`
- `CON_7790.stp`

Product types follow reference designator conventions: `R`, `C`, `IC`, `D`, `L`, `Q`, `CON`, `SW`, `XTAL`, `FB`, `F`, `RLY`, `BAT`. The full product number includes the package suffix — `TJA1051TK/3`, not `TJA1051T` — so each file maps to exactly one orderable part.

## Working With Three Agents

- **Doc Agent** (Part 2): I drop PDFs into `WIP/`, the agent reads the first pages to identify product type/number, renames the file, converts the PDF to structured Markdown with image extraction and equation OCR. Output lands in `Knowledge/`. I now have a searchable text version of every datasheet.

- **Lib Agent** (Part 3): I drop supplier library zips into `WIP/`, the agent unzips them, finds `.kicad_sym`/`.kicad_mod`/`.stp` files, renames them with product type prefixes, strips unnecessary metadata fields, quality-checks pad layers and courtyards, and places them in `kicad_lib/`. Ready to use in KiCad.

- **Designer Agent** (Part 4): I tell it what I want to design (e.g., "1.8V 2A buck converter using TPS62870"). It reads the datasheet from `Knowledge/`, asks me for input specs, runs the calculations, checks every result against datasheet limits with 20% margin, and writes a design document with page-numbered references back to the source.

## What's Next in This Series

Parts 2–4 cover each agent in detail: what they do, how I use them, and where they go wrong. Part 5 covers the subagent extension itself — how the process spawning works, the three execution modes, and how to write your own agents.

---

*CircuitPilot is on [GitHub](https://github.com/last-sociable-orange/pi-agent-team) under MIT. It builds on [pi](https://github.com/mariozechner/pi) and the `pdf-to-markdown` / `pdf-utils` skills.*
