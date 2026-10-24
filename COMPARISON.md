# Three-Way Comparison: Open-Source Libraries vs. Docling/MarkItDown vs. Azure AI Document Intelligence

> Numbers come from `benchmark_2pages.csv` (PDFs) and `web_benchmark_slice.csv` (web pages). Ground-truth counts for the PDFs were made by hand from the source papers. Items marked `[ ]` are manual entries still to be filled in (spot-check results, web ground truth, machine details, versions).

## 1. Setup and method

| # | Approach | Tools | Output form |
|---|----------|-------|-------------|
| 1 | Open-source libraries | pypdf, pdfplumber (PDF); requests + BeautifulSoup (web); our own reassembly into Markdown (`md_pdf.py`, `html_to_markdown`) | Text, JSON tables, layout JSON, image files, plus one assembled Markdown file |
| 2 | Markdown converters | Docling, MarkItDown | Markdown directly |
| 3 | Enterprise service | Azure AI Document Intelligence (prebuilt-layout, free tier F0) | Markdown (tables as HTML inside the Markdown) |

**Test set**

| ID | Type | Source | Pages | Used in the comparison |
|----|------|--------|------:|------------------------|
| PDF-1 | PDF | arXiv:2609.05822, "Generalizing HVAC Control With Domain Randomized Reinforcement Learning" (two columns, 3 numbered equations, 5 figures, algorithm box, no data tables) | 4 | pages 1-2 |
| PDF-2 | PDF | arXiv:2610.07998, "The Challenges of PROTAC Permeability Prediction" (single column, 3 multi-panel figures, 1 results table, no numbered equations) | 8 | pages 1-2 |
| WEB-1 | Web page | Wikipedia: Python (programming language) | n/a | slice of 7,155 characters |
| WEB-2 | Web page | Wikipedia: India | n/a | slice of 7,735 characters |

**Method**
- **Same input for every tool.** The Azure free tier reads 2 pages per request, so each PDF was cut to its first 2 pages (`azure_benchmark.py`) and all four tools processed that same 2-page file. For web pages, each page was cleaned (menus, headers, footers removed), cut at a block boundary to one slice, and the same slice went to all four tools (`azure_web_benchmark.py`).
- **Timing.** Open-source, Docling and MarkItDown: two runs, fastest reported (so Docling's model load is excluded). Azure: one run (a paid, network call), so its time includes network latency and has no warm-up run.
- **Accuracy** is measured by counting headings, tables, formulas (LaTeX) and images in each tool's Markdown (`md_metrics`) and comparing with a manual ground-truth count for the same pages, plus spot checks. For the open-source column, tables are counted from the pdfplumber JSON and images from the saved image files (as in `opensource_metrics`), so those two counts are not measured the same way as the Markdown-based counts of the other tools.
- **Reliability:** the `error` column is empty in both result files, so all 16 runs (8 PDF, 8 web) completed.

**Limits of this test set (read before the results).** The first 2 pages of both papers contain **no table and no display equation**. The only table in the set (Table 1, PDF-2) is on page 4, and the three numbered equations of PDF-1 are on page 3. The formula and table columns below therefore test mainly for false positives, not for correct extraction. PDF-1 has 4 pages, below the 5-page minimum in the assignment.

## 2. Accuracy

### 2.1 PDFs, first 2 pages (counts from `benchmark_2pages.csv`)

Ground truth for headings counts section headings only (Abstract, numbered sections; for PDF-1 also "CCS Concepts" and "Keywords"); the title is not counted.

| Doc | Source | Headings | Tables | Formulas (LaTeX) | Images |
|-----|--------|---------:|-------:|-----------------:|-------:|
| PDF-1 | **Ground truth (manual)** | 8 | 0 | 0 (inline math only) | 1 (Figure 1, a vector diagram) |
| PDF-1 | pypdf + pdfplumber | 8 | 12 | 0 | 0 |
| PDF-1 | Docling | 10 | 0 | 0 | 2 |
| PDF-1 | MarkItDown | 0 | 8 | 0 | 0 |
| PDF-1 | Azure DI | 12 | 0 | 0 | 1 |
| PDF-2 | **Ground truth (manual)** | 5 | 0 | 0 (inline statistics only) | 1 (Figure 1) |
| PDF-2 | pypdf + pdfplumber | 5 | 0 | 0 | 1 |
| PDF-2 | Docling | 8 | 0 | 0 | 1 |
| PDF-2 | MarkItDown | 0 | 0 | 0 | 0 |
| PDF-2 | Azure DI | 6 | 0 | 0 | 1 |

**What the numbers show**
- **Headings:** MarkItDown produced none on either paper, so its Markdown has no hierarchy. Docling and Azure both rebuilt headings but reported more than the section headings in the source (Docling 10 and 8 against 8 and 5; Azure 12 and 6). More is not better here: the extra items need the spot check in 2.3 before either can be called more accurate. The open-source assembly matched the ground truth exactly on both papers (8 and 5).
- **Tables:** the source has none on these pages. Docling and Azure report 0 on both papers, which is correct. pdfplumber reports 12 and MarkItDown 8 on PDF-1, so both invent tables there; the PDF-1 page 2 diagram (Figure 1) is a likely cause (boxes and text that look like cells) `[confirm in 2.3]`. On PDF-2 all four tools report 0.
- **Images:** the open-source extractor only finds embedded raster images. `pdfimages` confirms there are none on pages 1-2 of PDF-1 (Figure 1 is vector graphics) and one on page 2 of PDF-2 (Figure 1), which matches the open-source counts of 0 and 1. So it misses the one real figure of PDF-1. Azure (1 and 1) matches the ground truth on both papers. Docling matches on PDF-2 (1) and reports 2 on PDF-1, one more than the single figure `[check what the second is, 2.3]`. MarkItDown extracts no images from PDFs.
- **Formulas:** 0 for every tool, which is consistent with the ground truth but proves little, because these pages have no display equations. Docling only converts formulas to LaTeX when `do_formula_enrichment` is switched on; it is off by default in Docling's pipeline options, and this project does not enable it. The formula question is open until a page with equations is tested (PDF-1, page 3).
- **Text volume:** all four tools produced similar amounts of text (PDF-1: 9,650-10,608 characters; PDF-2: 7,226-7,678). This shows no tool dropped large blocks, but it does not show that the text is in the right order. (For the open-source column, characters are those of the plain-text file.)

### 2.2 Web pages, same slice for every tool (counts from `web_benchmark_slice.csv`)

| Page | Source | Headings | Tables | Images | Characters |
|------|--------|---------:|-------:|-------:|-----------:|
| python_programming | **Ground truth (count in the slice HTML)** | `[ 1 ]` | `[ 2 ]` | `[ 6 ]` | n/a |
| python_programming | requests + bs4 (open source) | 1 | 1 | 1 | 5,020 |
| python_programming | Docling | 1 | 0 | 1 | 4,364 |
| python_programming | MarkItDown | 1 | 4 | 6 | 29,838 |
| python_programming | Azure DI (HTML slice) | 1 | 2 | 0 | 7,767 |
| india | **Ground truth (count in the slice HTML)** | `[ 0 ]` | `[ 1 ]` | `[ 11 ]` | n/a |
| india | requests + bs4 (open source) | 0 | 1 | 0 | 7,160 |
| india | Docling | 0 | 5 | 11 | 168,530 |
| india | MarkItDown | 0 | 5 | 11 | 29,945 |
| india | Azure DI (HTML slice) | 0 | 1 | 0 | 8,752 |

**What the numbers show**
- **Headings:** all four tools agree on both slices (1 on Python, 0 on India). The India slice contains no heading, so it does not test headings.
- **Tables:** the tools disagree widely (Python: 0, 1, 2, 4; India: 1, 1, 5, 5). Both converters report more tables than the other two on India (5 against 1). Without the ground truth for the slice, the right count cannot be stated; fill the `[ ]` cells (see the counting note below) and then state which tool is correct.
- **Images:** Azure reports 0 on both slices, and the open-source assembly reports 1 and 0, while MarkItDown reports 6 and 11 and Docling 1 and 11. Whether the India slice really contains 11 images, and whether the open-source and Azure outputs dropped them, depends on the ground truth `[ ]`.
- **Output size:** Docling's India output is 168,530 characters from a 7,735-character slice, about 22 times the slice. This is an outlier and needs inspection before the number is used `[open data/output/web_markdown_slice/docling/india.md and look for what fills it, for example very wide padded table rows]`. MarkItDown's output is about 30,000 characters on both pages; it is likely to contain long link and image URLs `[confirm]`.

*Counting note (how to fill the web ground truth):* count `<h1>`-`<h6>`, `<table` and `<img` tags in `data/input/web/<name>_slice.html`.


### 2.3 Accuracy summary
- **PDF structure:** Docling and Azure are the only tools that rebuilt heading structure while reporting no false tables on these pages. MarkItDown has no headings and, on PDF-1, 8 false tables. The open-source assembly got the heading counts right but reported 12 false tables on PDF-1 and cannot see vector figures.
- **Not yet tested:** tables (PDF-2, page 4) and equations (PDF-1, page 3), the two hardest element types in the assignment.
- **Web:** headings agree everywhere; tables and images differ, and the conclusions wait for the slice ground truth.

## 3. Performance

PDFs (first 2 pages of each paper):

| Doc | Pages | pypdf + pdfplumber (s) | Docling (s) | MarkItDown (s) | Azure DI (s) |
|-----|------:|-----------------------:|------------:|---------------:|-------------:|
| PDF-1 | 2 | 4.27 | 12.05 | 1.51 | 12.83 |
| PDF-2 | 2 | 1.76 | 11.50 | 0.86 | 13.01 |

Seconds per page, same data divided by 2: open source 2.1 and 0.9; Docling 6.0 and 5.8; MarkItDown 0.8 and 0.4; Azure 6.4 and 6.5.

Web pages (one slice each):

| Page | requests + bs4 (s) | Docling (s) | MarkItDown (s) | Azure DI (s) |
|------|-------------------:|------------:|---------------:|-------------:|
| python_programming | 0.12 | 0.34 | 0.21 | 3.17 |
| india | 0.17 | 0.70 | 0.23 | 3.53 |

- **MarkItDown** is the fastest converter on PDFs (0.9-1.5 s for 2 pages).
- **Docling** and **Azure** take about the same time on PDFs, 11.5-13 s for 2 pages, which is 8-15 times MarkItDown. Docling's figure excludes model loading; Azure's is a single run that includes network time.
- **Web pages are cheap locally:** Docling needs 0.3-0.7 s on an HTML slice against 11.5-12 s on a 2-page PDF, because the PDF path runs layout models. Azure takes 3.2-3.5 s per slice, about 15 times MarkItDown and 20-26 times requests + bs4, which is the fastest tool (0.12-0.17 s).
- The open-source PDF time (1.8-4.3 s) includes saving text, tables, layout JSON, images and the assembled Markdown, so it is not a pure parsing time.
- Docling's one-time model load was removed by running twice; the cold-start cost was not measured `[ ]`.
- Machine: `[CPU / RAM / GPU]`. Azure timings also depend on region and network.

## 4. Ease of use

Lines of code are non-blank, non-comment lines in the project's own functions.

| Criterion | pypdf + pdfplumber | Docling | MarkItDown | Azure DI |
|-----------|--------------------|---------|------------|----------|
| Code for PDF to output | 53 lines (11 for text, 42 for tables, layout and images) `[re-count if the code changed]` | 27 lines (10 converter setup, 17 convert and export) `[re-count]` | 8 lines `[re-count]` | `[ ]` lines in `azure_di.py`, including the 2-page chunking and joining of Markdown on the free tier |
| Setup | `pip install` only | `pip install`, large dependencies, models downloaded on first run | `pip install markitdown[pdf]` | Azure account, a Document Intelligence resource, endpoint and key in `.env`; free tier F0 reads 2 pages per request |
| Output you get | Raw pieces, plus our own assembly code for Markdown | Markdown with headings, tables, image references | Flat Markdown; no headings, no images from PDFs | Markdown with headings; tables as HTML |
| Documentation | `[ ]` | `[ ]` | `[ ]` | `[ ]` |

Pypdf and pdfplumber need the most code and still return unassembled pieces. MarkItDown is the least code. Azure needs the most setup but returns structured Markdown without local models.

## 5. Cost

Azure list prices (October 2026). Microsoft's pricing page renders its prices dynamically, so these figures are from published summaries and should be confirmed on the [official pricing page](https://azure.microsoft.com/pricing/details/ai-document-intelligence/): Read about $1.50 per 1,000 pages; prebuilt models including Layout about $10 per 1,000 pages; free tier 500 pages per month. Commitment tiers can lower the rate for very large volumes.

| Volume | pypdf / pdfplumber | Docling | MarkItDown | Azure DI (Layout, $10 / 1,000) | Azure DI (Read only, $1.50 / 1,000) |
|--------|-------------------|---------|------------|-------------------------------:|------------------------------------:|
| 100 pages | ~$0 | ~$0 | ~$0 | $1.00 (free tier) | $0.15 (free tier) |
| 10,000 pages | compute only | compute only | compute only | $100 | $15 |
| 1,000,000 pages | compute only | compute only | compute only | $10,000 | $1,500 |

For the open-source columns, estimate compute with `hours = pages x seconds_per_page / 3600` and `cost = hours x VM hourly price`. Using the measured seconds per page from section 3 (2-page runs, so only a rough guide):

| Volume | pypdf / pdfplumber (0.9-2.1 s/page) | Docling (5.8-6.0 s/page) | MarkItDown (0.4-0.8 s/page) |
|--------|------------------------------------:|-------------------------:|----------------------------:|
| 10,000 pages | 2.5-5.8 h | 16-17 h | 1.1-2.2 h |
| 1,000,000 pages | 250-583 h | 1,611-1,667 h | 111-222 h |

Docling's compute bill is about ten times MarkItDown's on CPU; a GPU would reduce it. Self-hosted tools also cost engineering time and servers, while a managed service costs money per page but removes maintenance. The crossover depends on volume and on whether documents may leave your environment.

## 6. Scalability and production integration

| Concern | pypdf + pdfplumber | Docling | MarkItDown | Azure DI |
|---------|--------------------|---------|------------|----------|
| Scaling out | Easy; lightweight, parallelizes by file | Heavy models; needs more RAM, GPU helps; scale with workers or containers | Easy; lightweight | Scales by calls, not by your servers; free tier limited to 500 pages a month and 2 pages per request |
| Operational burden | Highest (you own the reassembly code) | Medium (model updates, resource sizing) | Low | Low maintenance; you manage keys, quotas and cost |
| Data privacy | Stays local | Stays local | Stays local | Documents are sent to Azure |
| Latency per request | 1.8-4.3 s per 2 pages | about 12 s per 2 pages | 0.9-1.5 s per 2 pages | about 13 s per 2 pages, plus network |
| Failure handling | Per-page try/except (as in `extractor.py`) | Slow and memory-hungry on long PDFs | Degrades quietly on complex layouts (8 false tables on PDF-1) | Network and quota errors to handle; vendor lock-in |
| Fit in the FastAPI + S3 pipeline | Fine for raw extraction and images | Good: Markdown plus images ready for S3, but slow per request on small hosts | Good for fast conversion | Good if sending documents out is acceptable |

## 7. Verdict

Based on the measurements above. The PDF conclusions are provisional: the compared pages hold no table and no equation, and the web ground truth is still to be filled in.

| Content type | Best choice | Evidence |
|--------------|-------------|----------|
| PDFs where structure matters | Docling (local) or Azure (managed) | Both rebuilt headings with no false tables on pages 1-2; both about 12-13 s per 2 pages. Both reported extra headings, to be spot-checked |
| Fast conversion of text-heavy PDFs | MarkItDown | 0.9-1.5 s per 2 pages; but no headings and 8 false tables on PDF-1 |
| Raw images and table data from PDFs | pypdf + pdfplumber | Embedded images match `pdfimages`; misses vector figures; 12 false tables on PDF-1 |
| Web pages | requests + bs4 assembly, or MarkItDown | Fastest (0.12-0.17 s); final choice waits for the table and image ground truth |
| High-volume, cost-sensitive | MarkItDown or requests + bs4 | Lowest time per page and no per-page fee |

**Overall recommendation:** Use **Docling** as the standardization step for PDFs when document structure must be preserved, accepting a speed cost of about 8-15 times MarkItDown. Choose **Azure** instead if you need managed scaling and can accept per-page cost ($10 per 1,000 pages for Layout), the free-tier limits, and sending documents to a cloud service. Use **MarkItDown** where speed matters and flat text is enough, and for quick web conversion. Keep **pypdf/pdfplumber** and **requests + bs4** for raw images and table data, remembering that they cannot see vector figures. Re-check this recommendation after testing a page with a table (PDF-2, page 4) and one with equations (PDF-1, page 3).

**Limitations:** two PDFs and two web pages only; PDF-1 has 4 pages; only the first 2 pages of each PDF were compared, so tables and formulas were not exercised; counts are counts, not correctness; Azure was run once, the others twice; one machine `[ ]`. Tool versions: pypdf `[ ]`, pdfplumber `[ ]`, Docling `[ ]`, MarkItDown `[ ]`.
