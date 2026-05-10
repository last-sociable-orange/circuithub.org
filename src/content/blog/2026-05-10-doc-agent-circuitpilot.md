---
title: 'Part 2: The Doc Agent — Turning PDFs into Markdown the AI Can Read'
description: "Part 2 of the Pi Extensions for Hardware Design series. How I use the doc agent to convert datasheets into structured Markdown so the designer agent can read them."
slug: 'doc-agent-circuitpilot'
pubDate: 'May 10 2026'
heroImage: '../../assets/blog-placeholder-5.jpg'
series: 'pi-extensions-for-hardware-design'
---

## Why I Convert Datasheets to Markdown

When I'm deep in a design and need to check a pin mapping, a register field, or absolute maximum ratings, I open a PDF and Ctrl-F my way to the answer. It works. But I do this dozens of times per design, and some PDFs, like user manuals for processors, are thousands of pages — grepping them is slow and reading them linearly is impossible.

The real reason I convert datasheets, though, is so an LLM can read them. If the designer agent (Part 4) can search a Markdown file for "feedback voltage" or "inductor selection guidelines" and pull the exact section it needs, it can run design calculations against real datasheet numbers. If the datasheet is still a PDF, the LLM gets garbled text — tables break, multi-column layouts confuse the tokenizer, images are invisible.

So the first step in any hardware design with AI assistance is: **convert every PDF to clean Markdown**. The doc agent does this for me.

## How I Use It

The doc agent is a sub-agent in CircuitPilot. It has one job: take PDFs from `WIP/` and produce organized, searchable Markdown in `Knowledge/`. 

### Document Lifecycle

I brought the Document Lifecycle idea into the document management workflow. This avoid LLM accidentally changes the documents we've reviewed and approved. 

```
WIP/ → (rename + identify) → .wip/ → (convert + cleanup) → .review/ → (approve) → Knowledge/ + Datasheet/
```

The agent moves files forward through this pipeline automatically. I only intervene at the review step. If I reject something, the agent moves it back to `.wip/` with notes on what to fix.

Here's what happens when I use it.

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

## How Documents Are Organized

The doc agent doesn't just convert PDFs — it builds a structured library that both I and the other agents can navigate without guesswork.

### The Full Directory Tree

Every document passes through four directories, each with a clear purpose:

```
.data/documents/
├── WIP/              # I drop raw PDFs here
├── Datasheet/        # Original PDFs, renamed and sorted
│   └── IC-TPS62870-DS/
│       └── IC-TPS62870-DS.pdf
├── Knowledge/        # Converted Markdown, organized by part
│   ├── knowledge.md  # One-line index of all documents
│   └── IC-TPS62870-DS/
│       ├── IC-TPS62870-DS.md
│       └── images/
└── .trash/           # Nothing is ever deleted — just moved here
```

- **`WIP/`** — Landing zone. I dump any PDF here, no naming convention required. The agent watches this directory and processes files on demand.
- **`Datasheet/`** — The original PDF, renamed to the standard `<TYPE>-<PN>-<DOCTYPE>.pdf` convention. This is my permanent archive. If the supplier removes the PDF from their site, I still have it.
- **`Knowledge/`** — The converted Markdown, extracted images, and the master index. This is what agents read. I don't touch files in here directly — the agent manages everything.
- **`.trash/`** — When I tell the agent to remove a document, it moves the files here instead of deleting them. I can recover anything for 30 days before the agent purges old trash.

### How knowledge.md Works

The index file is maintained automatically. Every time a document moves from `.review/` to `Knowledge/`, the agent appends a line:

```
IC | TPS62870 | DS | Step-down converter, 2.4V-6V in, 0.8-3.3V out, 3A | 2026-05-10
```

The format is `TYPE | PN | DOCTYPE | summary | date-processed`. The agent generates the summary from the first paragraph of the datasheet, so every entry is a useful hint about what the part does.

This document is useful for other sub-agents, like `lib` or `designer`, when they are trying to grep quickly about a particular part information or look into what datasheets are available for further study. 

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
