---
title: 'Getting AI to Read Datasheets — A PDF-to-Markdown Pipeline for Engineering Documents'
description: "Why OCR is the wrong approach for feeding PDF datasheets and user manuals into AI coding agents, and how a PyMuPDF-based extraction pipeline delivers clean, structured Markdown output instead."
slug: 'pdf-to-markdown-skill'
pubDate: 'May 01 2026'
heroImage: '../../assets/blog-placeholder-4.jpg'
---

## Why I Needed This

AI coding agents — tools like Pi, Claude Code, Cline, and others — can read text files natively, but they can't read PDFs. That's a problem when you're an electronics engineer trying to get an AI assistant to help with design work, because virtually every component datasheet, application note, and user manual ships as a PDF.

The obvious solution is to convert PDFs to something the AI can ingest. The popular approach among AI tools right now is OCR — treat every PDF as a scan, run it through Tesseract or a cloud OCR service, and shovel the result into the context window.

I think OCR is the wrong approach for technical documents, and here's why.

## The Problem With OCR

Most engineering PDFs are *born-digital* — they were written in Word, FrameMaker, or InDesign and exported to PDF. The text is embedded as actual character data with fonts, positioning, and structure. OCR is designed for the opposite case: scanned paper documents where the only information is pixel patterns.

Running OCR on a born-digital PDF is unnecessary — the text is already there, so any OCR pass is pure overhead. The practical downsides are time and cost:

- **Time**: OCR is slow. A typical 50-page datasheet takes a few seconds to extract via direct parsing, but minutes or even hours through an OCR pipeline — especially if it's a cloud OCR service. When you have tons of datasheets or user manuals that have thousands of pages, that waiting time adds up quickly.
- **Cost**: Cloud OCR services charge per page. If you're running an agent that frequently consults datasheets — say, during component selection or schematic review — those API costs accumulate quickly. Local OCR (Tesseract, or AI based MinerU, PaddleOCR) avoids the per-page fee but usually they require a beefy computer with expensive video card to deploy and run.
- **Integration friction**: OCR doesn't drop easily into an automated workflow. You either use a web-based service (manual upload, not scriptable) or sign up for a paid API, write client code, handle rate limits and authentication, and manage retries. Compare that to importing a Python library and calling one function — the difference in setup effort is substantial.

Direct extraction from the PDF's content stream solves both problems: it's free, and it runs in under a second for most documents.

## The Approach: PyMuPDF + pymupdf4llm

[PyMuPDF](https://pymupdf.readthedocs.io/) (fitz) is a mature PDF library that reads the PDF's internal content stream directly — character codes, font mappings, positioning matrices, and image objects. It doesn't need to guess what the pixels say because the text is already there.

[pymupdf4llm](https://github.com/pymupdf/RAG) is a wrapper that takes PyMuPDF's structured output and formats it as Markdown. It handles:

- **Text formatting**: Bold, italic, monospace, headers — it maps PDF font styles to Markdown syntax.
- **Tables**: Detects tabular content from positional analysis and outputs proper Markdown tables.
- **Images**: Extracts embedded images to separate files and inserts Markdown image references.
- **Headers/footers**: Can strip running headers and page numbers that would otherwise clutter the output.

## The Script: `pdf_to_markdown.py`

The script itself is straightforward — a thin CLI wrapper around `pymupdf4llm.to_markdown()` with some conveniences:

```bash
# Basic extraction
python pdf_to_markdown.py LM324-datasheet.pdf

# Specific pages (saves time on large documents)
python pdf_to_markdown.py LM324-datasheet.pdf --pages 1-5

# Without images (faster, smaller output)
python pdf_to_markdown.py LM324-datasheet.pdf --no-images
```

The output is three files:

| File | Purpose |
|------|---------|
| `LM324-datasheet.md` | Concatenated Markdown — feed this to your AI |
| `LM324-datasheet.json` | Page-level chunks for cross-referencing |
| `images/` | Extracted figures, graphs, package drawings |

## Equation Handling

`pymupdf4llm` does a decent job on extracting text/table/image from structured pdfs. There is one thing it does **not** handle well: **equations**. Most engineering PDFs render mathematical expressions as embedded vector graphics or image objects — they aren't represented as text in the content stream. `pymupdf4llm` will extract these as image files, but it can't convert them to LaTeX on its own.

This is where the skill's image output becomes important. Since the skill extracts all images to a known directory, a downstream AI agent with image capability (qwen3.6-plus, for example) can:

1. Scan the extracted images for ones that contain equations (typically recognizable by their position relative to figure references in the Markdown)
2. Use an LLM to read the image and transcribe the equation to LaTeX syntax
3. Insert the LaTeX equation into the Markdown file immediately after the image reference, keeping the original image in place as a fallback

This separation of concerns is deliberate: `pdf_to_markdown.py` handles the deterministic PDF parsing, while the equation OCR is deferred to an LLM that can interpret the visual content. LLM doesn't read and translate the entire pdf as OCR anymore. It just handles a tiny friction of the pdf and it is much faster. This approach takes the best of two sides with great time and cost saver.

In practice, this workflow is handled by a [doc agent](https://github.com/last-sociable-orange/pi-extensions) that orchestrates the full pipeline — extract, clean up image paths, OCR equations, and insert LaTeX — turning a raw datasheet PDF into a Markdown file with proper mathematical notation.

## Usage in Context

In practice, this fits into a agent workflow like so:

1. User downloads the pdf datasheet and throw it into project `WIP` folder.
2. User calls the agent to turn it into markdown. 
2. Agent calls the pdf-to-markdown skill on pdf.
3. The skill produces a markdown with all images and equations inserted.
4. User asks the agent a question about the product design.
4. The agent reads the Markdown file, finds the related contents, and answers the question.

## Requirements

```bash
pip install pymupdf4llm
```

That's it. The script itself is a single file with no other dependencies. Python 3.9+.

## The Script

The script lives in the [pi-skills repository](https://github.com/last-sociable-orange/pi-skills/tree/main/pdf-to-markdown/scripts):

- `pdf_to_markdown.py` — converts PDF to Markdown + JSON page chunks

## Disclaimer

These scripts are provided as-is for personal and hobbyist use. PDF extraction quality varies with the source PDF's structure. Some PDFs — particularly those with heavy use of form fields, mathematical notation, or non-standard encodings — may produce imperfect Markdown. Always verify critical specifications against the original datasheet.

---

*Code and issues on [GitHub](https://github.com/last-sociable-orange/pi-skills).*
