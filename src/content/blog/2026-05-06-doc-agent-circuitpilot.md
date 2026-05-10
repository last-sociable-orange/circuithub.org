---
title: 'Part 2: The Doc Agent — Turning PDFs into Markdown the AI Can Read'
description: "Part 2 of the Pi Extensions for Hardware Design series. How I use the doc agent to convert datasheets into structured Markdown so the designer agent can read them."
slug: 'doc-agent-circuitpilot'
pubDate: 'May 06 2026'
heroImage: '../../assets/blog-placeholder-5.jpg'
series: 'pi-extensions-for-hardware-design'
---

## Why I Convert Datasheets to Markdown

When I'm deep in a design and need to check a pin mapping, a register field, or absolute maximum ratings, I open a PDF and Ctrl-F my way to the answer. It works. But I do this dozens of times per design, and the PDF is often 60 pages. User manuals for processors are thousands of pages — grepping them is slow and reading them linearly is impossible.

The real reason I convert datasheets, though, is so an LLM can read them. If the designer agent (Part 4) can search a Markdown file for "feedback voltage" or "inductor selection guidelines" and pull the exact section it needs, it can run design calculations against real datasheet numbers. If the datasheet is still a PDF, the LLM gets garbled text — tables break, multi-column layouts confuse the tokenizer, images are invisible.

So the first step in any hardware design with AI assistance is: **convert every PDF to clean Markdown**. The doc agent does this for me.

## How I Use It

The doc agent is a sub-agent in CircuitPilot. It has one job: take PDFs from `WIP/` and produce organized, searchable Markdown in `Knowledge/`. Here's what happens when I use it.

### 1. I Drop PDFs Into WIP/

I download datasheets from TI, ST, NXP, whoever, and dump them in `WIP/`. Filenames are whatever the supplier gave me — `tps62870.pdf`, `slvaf83.pdf`, `Datasheet (3).pdf`. I don't rename anything. That's the agent's job.

### 2. The Agent Identifies Each File

I tell the lead agent "process the PDFs in WIP/." It delegates to the doc agent, which reads the first page or two of each PDF using PyMuPDF (`pdf-utils` skill). From those pages it figures out:

- **Product type** — is this an IC, capacitor, connector, crystal? It uses reference designator conventions.
- **Product number** — manufacturer part number, pulled from the document metadata or title page text.
- **Document type** — datasheet, user manual, app note, errata.

If it can't determine any of these from the first two pages, it asks me. It won't guess a part number.

### 3. Files Get Renamed

The agent renames each PDF to a consistent format:

```
<PRODUCT_TYPE>-<PRODUCT_NUMBER>-<DOCUMENT_TYPE>.pdf
```

Real examples from my projects:
- `tps62870.pdf` → `IC-TPS62870-DS.pdf`
- `mimxrt1170_evk_ug.pdf` → `IC-MIMXRT1170-UM.pdf`
- `slvaf83.pdf` → `IC-TPS62870-AN.pdf`

All caps, underscores for illegal characters. Every filename now tells me exactly what's inside without opening it.

### 4. Conversion to Markdown

The agent copies the renamed PDF to a dedicated folder under `Knowledge/` and runs the `pdf-to-markdown` skill, which uses pymupdf4llm:

```bash
python3 pdf_to_markdown.py \
  Knowledge/.wip/IC-TPS62870-DS/IC-TPS62870-DS.pdf \
  -o Knowledge/.wip/IC-TPS62870-DS/IC-TPS62870-DS.md \
  --image-dir Knowledge/.wip/IC-TPS62870-DS/images/
```

What I get:

- **Headings**: PDF heading hierarchy preserved as `#`, `##`, `###`.
- **Tables**: Real Markdown tables, not screenshots. Pipe-delimited, readable.
- **Images**: Extracted as PNGs into `images/`, referenced with relative paths.
- **Equations**: OCR'd from image captures, inserted as LaTeX alongside the original image so I can verify.

### 5. Cleanup

Raw pymupdf4llm output is noisy — OCR artifacts, absolute paths, junk text from image captions. The agent runs a cleanup script (bundled with the skill) that:

1. Strips OCR noise from image text blocks
2. Rewrites image paths to relative (`images/figure-01.png`)
3. Scans extracted images for equations, OCRs them, and inserts LaTeX

The cleanup is deterministic. Same input, same output every run.

### 6. I Review, Then Approve

The agent moves everything to `.review/` and tells me to check. I skim the Markdown — does the electrical characteristics table look right? Are equations rendered correctly? If something's off, I tell it, and it moves files back to `.wip/` for revision. When I'm satisfied, files move to `Knowledge/` and `Datasheet/`.

Nothing gets deleted. The agent moves stuff to `.trash/` instead of `rm`. Supplier documents occasionally get pulled offline, so I keep the originals.

## What the Output Looks Like

After processing, `Knowledge/` looks like this:

```
Knowledge/
├── knowledge.md            # One-line index of every document
└── IC-TPS62870-DS/
    ├── IC-TPS62870-DS.md   # The converted datasheet
    └── images/
        ├── IC-TPS62870-DS.pdf-0001-38.png  # Block diagram
        ├── IC-TPS62870-DS.pdf-0002-15.png  # Efficiency curve
        └── ...
```

One folder per document keeps things isolated. `knowledge.md` gives LLM a quick index — one line per document with product type, number, and a brief description. I can grep the whole knowledge base in one command.

## What This Enables

Once a datasheet is Markdown, the designer agent can read it, research and answer my questions, and generate structured design documents with confidence a lot faster than I do. I can also verify LLM's outputs against it.

## Limitations

Scanned datasheets with no text layer produce garbage. OCR is best-effort — if the PDF is a scan of a photocopy, the output won't be useful. Complex tables with merged cells or multi-row headers sometimes need manual cleanup. And the agent only handles PDF — no proprietary formats.

For text-based datasheets from major suppliers (TI, ST, NXP, ADI, Microchip, etc.), it works reliably.

## Using It

```bash
git clone https://github.com/last-sociable-orange/pi-agent-team
cd pi-agent-team
./setup.fish
```

Then in my project: drop PDFs into `WIP/`, tell CircuitPilot to process them, review the output, approve. The agent handles the rest.

---

*The Doc Agent is part of [CircuitPilot](https://github.com/last-sociable-orange/pi-agent-team). Read Part 3 for the Lib Agent and Part 4 for the Designer Agent.*
