# Building Code Compliance Parsing

This repository contains notebooks and sample artifacts for extracting structured content from building code PDFs (e.g., BNBC chapters). The main flow is: convert PDFs into Docling JSON, post-process that JSON into a clause tree with lists/tables/figures/equations, optionally apply OCR, and optionally enrich clauses with cross‑references.

## Notebook overview

### `main_parser.ipynb`
This notebook runs the core “no‑OCR” pipeline against a PDF that already has a usable text layer.

* **Docling conversion with formula enrichment**: Uses `DocumentConverter` with `PdfPipelineOptions.do_formula_enrichment = True` so equation regions are tagged as `FORMULA` and carry LaTeX in the exported JSON. The notebook then exports the Docling document to a JSON file for downstream parsing.
* **Structured parsing pass**: Loads the Docling JSON and walks the document body in reading order to build a clause tree keyed by numeric clause IDs. It detects clause headers, splits list items vs. paragraph text, nests list items based on indentation/marker heuristics, and merges text blocks into clause nodes.
* **Tables, figures, and equations**: Merges table fragments across pages, extracts/infers table and figure captions, and crops figure/equation images from the original PDF using PyMuPDF with Docling bounding boxes. It also extracts equation LaTeX (with optional de‑spacing fixes) into a structured equations JSON.
* **Outputs**: Writes `structured_clauses.json`, `structured_tables.json`, `structured_images.json`, and `structured_equations.json`, plus image crops in an output directory.

### `img2eqn.ipynb`
This notebook focuses specifically on converting cropped equation images into LaTeX using a vision‑language model.

* **Model setup**: Loads Qwen2‑VL (e.g., `Qwen2‑VL‑7B‑Instruct`) with its processor and prepares a prompt that requests a JSON response with `latex` and an optional `label`.
* **Batch image processing**: Iterates through an image directory, runs inference per image, and uses a robust JSON extraction helper to handle model output formatting.
* **Outputs**: Writes a JSON list mapping image filenames to extracted LaTeX and labels. This is intended as a post‑processing step for equation crops produced by the parser notebooks.

### `references.ipynb`
This notebook enriches clause data with cross‑references to sections, tables, figures, equations, and chapters.

* **Regex‑based extraction**: Defines patterns for references like “Section 2.5.9.1”, “Table 6.2.23”, etc., then normalizes and de‑duplicates numeric paths. It also supports simple range handling (e.g., `2.5.9.1–2.5.9.5`).
* **Optional LLM pass**: Includes a lightweight LLM extractor (e.g., Qwen2.5‑3B‑Instruct) that can be applied selectively to clauses that likely contain references but weren’t caught by regex. The LLM output is normalized and merged with regex results.
* **Outputs**: Produces `structured_clauses_with_refs.json`, adding a `references` block per clause with provenance metadata for regex vs. LLM sources.

### `main_parser_ocr.ipynb`
This notebook mirrors the main parser but adds an OCR‑first preprocessing stage to improve results when PDFs have problematic or missing text layers.

* **PDF flattening step**: Rasterizes each page to images at a chosen DPI, then stitches those images back into a new “flattened” PDF. This strips hidden text layers that can confuse OCR and standardizes the input for OCR.
* **Docling OCR conversion**: Runs Docling with `do_ocr = True` and `do_formula_enrichment = True` on the flattened PDF, exporting a Docling JSON similar to the no‑OCR flow but derived from OCR text.
* **Reuse of the structured parser**: Reuses the same parsing pipeline as `main_parser.ipynb` to build the clause tree, and to extract tables/figures/equations and their associated images/LaTeX.


## Sample outputs

The `structured_output/` directory contains example JSON outputs (clauses, tables, figures, equations) along with cropped equation images. The `with_ocr/` and `without_ocr/` folders include example outputs for comparison between OCR and non‑OCR runs.
