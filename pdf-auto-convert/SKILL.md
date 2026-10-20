---
name: pdf-auto-convert
description: Convert PDFs to clean Markdown before reading them, so the AI uses a fraction of the tokens and understands the text better. Use whenever a PDF is uploaded or referenced and the user wants it read, understood, analysed, summarised, searched or extracted from, e.g. "read this PDF", "summarise this document", "what does this say", "pull the figures out of this report". Skip it when the user needs the PDF itself: filling forms, merging or splitting, visual layout, page design, or images inside the file. Works in claude.ai, Cowork, Claude Code and any environment with Python.
---

# pdf-auto-convert

Reading a PDF directly is expensive and noisy. Depending on the tool, every page is ingested as text plus a page image, which can cost thousands of tokens per page. Converting it to Markdown first keeps the words, headings, lists and tables, and drops the noise. A 20-page report typically falls from tens of thousands of tokens to a few thousand.

This is background plumbing. The user asked a question about the document; answer that question. Don't narrate the conversion.

## When NOT to use it

Use a PDF tool instead when the user wants to:
- fill, edit, merge, split, sign or watermark the PDF
- keep it as a PDF ("I need the PDF version")
- look at layout, design, charts or images in the document
- extract embedded images

## Step 1: find the file

The PDF may be an upload (in claude.ai and Cowork, usually under `/mnt/user-data/uploads/`), a path the user gave, or a file in the project. Use the actual path; don't assume a folder.

## Step 2: check it has a text layer

```bash
pdffonts "FILE.pdf" | tail -n +3 | head -3   # no rows = no embedded fonts = probably a scan
pdfinfo "FILE.pdf" | grep -E "Pages|Title"
```

If `pdffonts` isn't installed, skip this; the length check in step 3 catches scans too.

## Step 3: convert

Try these in order and stop at the first that works:

```python
# 1. markitdown (best structure: headings, lists, tables)
#    install if missing: pip install 'markitdown[pdf]'   (or run without installing: uvx 'markitdown[pdf]' FILE.pdf)
from markitdown import MarkItDown
text = MarkItDown().convert("FILE.pdf").text_content.strip()
```

```bash
# 2. pdftotext (poppler), fast and nearly always available; -layout keeps tables aligned
pdftotext -layout "FILE.pdf" - 
```

```python
# 3. pypdf, pure Python, last resort
from pypdf import PdfReader
text = "\n\n".join((p.extract_text() or "") for p in PdfReader("FILE.pdf").pages)
```

Sanity check: if the result is under ~100 characters per page, the PDF is almost certainly a scan. Go to the scanned-PDF fallback below.

For documents that are mostly tables (financials, rate cards), `pdfplumber` extracts tables cell by cell and is worth trying if the Markdown tables come out jumbled.

## Step 4: use it

Answer from the Markdown. Quote figures exactly as they appear. Don't show the user the raw Markdown unless they ask for it. If they want to keep it, save it next to the PDF as `FILE.md`.

For long documents, search the Markdown (grep for the terms in the question) rather than reading every page into context.

## Scanned-PDF fallback

Text extraction can't read a scan. Tell the user plainly: *"This PDF is a scan, so the text isn't machine-readable. I'll read the page images instead, which takes longer."* Then either:
- render the pages to images (`pdftoppm -r 150 -png FILE.pdf page`) and read them with the model's own vision, a handful of pages at a time; or
- run OCR if it's installed (`ocrmypdf FILE.pdf FILE-ocr.pdf`, then convert the OCR'd PDF as in step 3).

Never return an empty answer or pretend to have read a document you couldn't extract.

## Several PDFs at once

Convert each one separately and keep track of which text came from which file before you compare or combine them.

```python
from pathlib import Path
from markitdown import MarkItDown
md = MarkItDown()
docs = {p.name: md.convert(str(p)).text_content.strip() for p in Path("FOLDER").glob("*.pdf")}
for name, text in docs.items():
    print(f"{name}: {len(text):,} chars")
```

## How to answer

Lead with the answer, not the process.

Good: "The contract is dated 14 March 2024 and sets out three obligations..."
Not good: "I've converted your PDF to Markdown and can now read it. The document is a contract dated..."
