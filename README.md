# Accord — agreement metadata extraction

A minimal document workspace that reads uploaded files at runtime and extracts agreement value, start/end dates, notice period, and the two contracting parties. Supports batches, document previews, evidence quotes, uncertainty notes, and CSV/JSON downloads.

## Run locally

Requires Python 3.12 or newer. Tested on Python 3.12.

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.lock.txt
cp .env.example .env
python -m uvicorn app.main:app --host 127.0.0.1 --port 8000
```

Open **http://127.0.0.1:8000**. On Windows, activate with `.venv\Scripts\activate`.

For extraction, click **Connect AI**, select **Groq** or **OpenAI**, and enter a key from that provider. A key entered in the UI stays in page memory and disappears on reload. The status “Key configured” indicates presence, not validated credentials. Previewing files needs no API key. Switching provider clears the key input so credentials are not accidentally sent to the other service.

The supplied `.env.example` selects Groq. For a server-side Groq key, configure `.env` as follows and restart the server:

```dotenv
AI_PROVIDER=groq
GROQ_API_KEY=your_new_groq_key_here
GROQ_MODEL=qwen/qwen3.8-27b
```

Alternatively, set `AI_PROVIDER=openai`, `OPENAI_API_KEY`, and `OPENAI_MODEL=gpt-4.1-mini`. Without an `AI_PROVIDER` setting, existing installations default to OpenAI. Never use a Groq key with an OpenAI model/endpoint, or vice versa. Groq is a different service from xAI's Grok; xAI keys are not supported here.

Models can be changed in Settings or the provider-specific environment variable. Use a model with image input and JSON output support (OpenAI additionally needs structured output support). The provider account needs access to that model and available API usage; requests can incur charges. The app fixes each provider's endpoint internally; no endpoint setting is needed.

`requirements.txt` contains supported dependency ranges; `requirements.lock.txt` captures the versions verified for this project.

## Use the workspace

1. Drag in one or more documents, or choose **browse files**.
2. Select a file and open **Source document** to inspect readable text and images.
3. Select **Extract details** (or **Extract N documents**) to process the queue.
4. Review field values, supporting text, and notes. A field can be found, inferred, uncertain, or missing.
5. Export all completed documents as CSV or the selected result as JSON.

Different templates and new filenames use the same runtime pipeline. No filename-specific values, dataset lookups, or hard-coded extraction rules are used. The six output fields follow the assignment; the input format and layout are flexible. This is agreement metadata extraction, not arbitrary-schema extraction for every kind of document. Unrelated documents should produce missing fields rather than invented agreements.

## Supported formats and limits

| Input | Extensions | Reading method |
|---|---|---|
| PDF | `.pdf` | Text plus rendered images of every page, including scanned pages |
| Word | `.docx` | Paragraphs, tables, headers/footer paragraphs, text boxes, footnotes/endnotes and embedded raster images |
| Images | `.png`, `.jpg`, `.jpeg`, `.webp`, `.tif`, `.tiff`, `.bmp`, `.gif` | Vision input; frames/pages are included; tall scans are split with overlap |
| OpenDocument | `.odt` | Document XML text and embedded raster images |
| Rich text | `.rtf` | Decoded text; embedded objects are not rendered |
| Plain text | `.txt`, `.md` | Text decoding |
| Structured data | `.csv`, `.tsv`, `.json`, `.xml` | Decoded text/cells |
| HTML | `.html`, `.htm` | Visible textual content; scripts/styles removed, external resources not fetched |
| Email | `.eml` | Subject and body; attachments must be uploaded separately |
| Excel | `.xlsx` | All sheet cell values and embedded raster images; saved formula values |
| PowerPoint | `.pptx` | Slide text, tables, notes and embedded raster images |

There is no universal file reader. Legacy `.doc`, `.xls`, `.ppt`, password-protected files, HEIC, archives, audio, and video are not supported. Export unsupported documents to PDF, modern Office formats, or text first. The UI's supported-format list comes from the backend.

Per file: **20 MB**, **30 PDF pages / image frames / slides**, **40 image sections**, **120,000 text characters**. Workbooks are limited to 100,000 visited cells; Office archives to 100 MB expanded content. Larger documents fail with a clear message rather than silently losing their tail. The browser queue is limited to 20 files and processes extraction sequentially.

Office text extraction is not a full layout renderer. Complex drawings, charts, linked objects, nested tables, or unusual Office features may not be captured. Export to PDF when exact visual fidelity matters. Model vision handles scans directly; there is no separate OCR installation. Poor handwriting, rotation, blur, or very small text can still limit accuracy.

## Architecture and approach

```text
Browser upload → FastAPI → format reader → text + image sections
                                            ↓
                         OpenAI Responses / Groq Chat Completions
                                            ↓
                         validation → six fields + evidence + notes
                                            ↓
                            UI / API / batch predictions → evaluation
```

- **Readers** decode file formats only; they do not select agreement values. PDFs include both text and rendered images. Tall images are tiled to preserve readability, with 100-pixel overlap.
- **Extraction** uses a pretrained multimodal language model, zero-shot, with a fixed task prompt and Pydantic response validation. OpenAI uses strict structured output; Groq uses JSON mode with the schema in its instructions and one bounded retry on schema-validation failure. The small, noisy dataset is not used for fine-tuning or as examples. Neither `train.csv` nor `test.csv` is read during inference.
- **Groq vision** sends up to three image sections in each request. Longer documents are first transcribed in batches by the vision model; every transcript is combined with the document's original text for final model-based extraction. No images are intentionally skipped. Truncated responses or oversized combined transcripts produce explicit errors. This adds calls, latency, cost, and possible transcription errors, which are disclosed in the result warnings.
- **Field interpretation** distinguishes recurring rent from deposits, commencement from signing, and parties from witnesses. An explicit renewal notice takes precedence; termination notice is used only as a disclosed fallback. Thirty days per month is a disclosed approximation. Derived end dates and converted durations are marked inferred.
- **Validation** checks response structure and rejects impossible output dates. Validation and file routing are ordinary application logic; agreement metadata is not extracted with regex or static conditions.
- **Evidence** is a model-generated source quote plus an explanation. These are review aids, not verified citations or calibrated confidence estimates.

The implementation follows [OpenAI structured outputs](https://developers.openai.com/api/docs/guides/structured-outputs), [OpenAI image inputs](https://developers.openai.com/api/docs/guides/images-vision), and [Groq vision and JSON-mode guidance](https://console.groq.com/docs/vision). Groq's [OpenAI-compatible endpoint](https://console.groq.com/docs/openai) is `https://api.groq.com/openai/v1`; Groq requests use Chat Completions, not OpenAI's Responses schema.

## Batch predictions and evaluation

Set `GROQ_API_KEY` in `.env`, then:

```bash
python -m app.cli predict data/test --provider groq --output output/test_predictions.csv
python -m app.cli evaluate --predictions output/test_predictions.csv --labels data/test.csv --output output/metrics.json
python -m app.cli audit
```

Use `--provider openai` with an `OPENAI_API_KEY` to use OpenAI instead. `--model` overrides the selected provider's default. Prediction reads only the documents in the selected folder. It writes both the assignment-compatible CSV and a companion JSON with evidence, provider, model ID, warnings, and failures. Failed files retain empty prediction rows and cause a nonzero exit status; they are not silently omitted from evaluation. Predictions should be treated as untrusted data when opening the raw CLI CSV in spreadsheet software. The UI's separate CSV export neutralizes formula-like cells for spreadsheet use.

The primary per-field metric is the assignment's literal exact-match recall:

```text
recall = exact matches / number of nonblank ground-truth values for that field
```

Missing predictions count as failures. Blank ground-truth values are excluded because there is no labelled value to compare; denominators are reported per field. A supplementary whitespace-trimmed score is shown separately. Case, punctuation, and dates are otherwise not normalized. Compound extensions such as `.pdf.docx` are handled when matching file IDs. Duplicate IDs are rejected.

**Live prediction status:** genuine test-set predictions and recall scores have not been generated. No scores or predictions are fabricated. Run the commands above after configuring an active key to produce the remaining assignment outputs. Provider calls in tests are mocked; they validate integration behavior, not live account access or extraction accuracy.

### Supplied dataset issues

- `24158401-Rental-Agreement` is labelled in both train and test CSVs, while its file appears only in the test directory. Its train label must not be used to tune or prompt test inference.
- `46239065-Standard-Rental-Agreement-Rental-With-Performance-Fee.docx` has no training label.
- Training end dates `31.11.2009`, `31.04.2011`, and `31.02.2011` are impossible calendar dates.
- One training notice label is blank. Names and dates contain inconsistent surrounding spaces.
- The `47854715` agreement states one month for renewal and two months for termination; the supplied notice label is 60 days. This implementation prioritizes explicit renewal notice and reports the distinction rather than manipulating outputs to fit the label.
- The two Geraldine agreements are related versions; the short version does not contain all clauses found in the long version.

Original documents and labels are left unchanged. These inconsistencies can reduce exact-match scores even when an extraction is faithful to the source.

## REST API

Interactive documentation: **http://127.0.0.1:8000/docs**.

| Endpoint | Purpose |
|---|---|
| `GET /api/config` | Supported formats, limits, selected provider, provider models and whether each provider's server key is present |
| `POST /api/preview` | Multipart `file`; returns decoded text and normalized image sections |
| `POST /api/extract` | Multipart `file`, optional `provider` (`groq` or `openai`) and `model`; optional `X-API-Key` header overrides only the selected provider's server key |

With a server key in `.env`:

```bash
curl -X POST http://127.0.0.1:8000/api/extract \
  -F 'file=@data/test/24158401-Rental-Agreement.png' \
  -F 'provider=groq' \
  -F 'model=qwen/qwen3.8-27b'
```

Responses contain `filename`, `provider`, `model`, and `result`. Each field in `result` contains `value`, `status`, `evidence`, and `explanation`; `result` also includes document type, summary, currency, and warnings. Values are strings or null, dates use `DD.MM.YYYY`, and monetary amounts omit currency and separators. Errors use a human-readable `detail`. The server never falls back to another provider's key.

## Tests

```bash
python -m pytest -q
```

Tests cover every supplied document, scanned PDFs, tall images, multipage TIFFs, Office tables/images, additional formats, limits, corrupted files, unsafe XML entities, API errors, model refusal, output validation, exact-match denominators, leakage auditing, Groq image batching, JSON retries, truncated responses, credential isolation, and provider mismatches. Provider responses are mocked so the tests need no key or API spending.

For browser workflow tests, start the app, install the optional browser-test dependencies, and run:

```bash
npm install
npx playwright install chromium
npm run test:ui
```

The browser test uses real local readers and mocked extraction responses, checks uploads, settings, batch results, previews, exports, error recovery, and mobile layout. Screenshots are written to the ignored `output/` directory.

## Project structure

```text
app/
  main.py        FastAPI routes and static UI
  readers.py     Input decoding, images and limits
  extraction.py  Model prompt, request and response validation
  schemas.py     Typed extraction response and CSV mapping
  cli.py         Batch inference, evaluation and dataset audit
  config.py      Environment settings
  static/        Minimal browser interface (HTML/CSS/JavaScript)
tests/           Reader, API, evaluation and browser tests
data/            Original supplied dataset, unchanged
```

## Local-use boundaries

The app is designed for a single user on `127.0.0.1`. There is no login, persistent job queue, or public-deployment hardening. Before exposing it to a network, add authentication, request/rate limits, HTTPS, and isolated file-processing workers.

Document contents are sent only to the selected provider for extraction. OpenAI requests set `store=False`; Groq uses its Chat Completions API without that OpenAI-specific option. Each provider's own retention policies still apply. The app does not deliberately persist uploaded documents or UI results, although the upload framework can temporarily spool large files to the OS temporary directory and closes them after reading. CLI outputs are intentionally saved. Browser data is lost on reload, so export results you need to keep.
