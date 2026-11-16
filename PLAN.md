# PLAN.md — CV-Abgleich Demo (version B)

Status: draft, waiting for "go". No code is written until then.

## 0. What exists today

The repo has one commit (`06fb791 Initial commit`) and one file, `README.md`, containing only the heading `# cv_classifier`. There is no code, no data and no config. Everything below is new.

## 1. Scope in one paragraph

Input: one CV (PDF, DOCX or pasted text) and one job description (pasted text). Output: (1) an anonymised candidate profile in one fixed German template, with PDF export, and (2) a match sheet, one row per job requirement, each with status, verbatim evidence, confidence and method. The tool never ranks, scores overall or rejects. Deterministic layers do the routine work; the LLM is called only for what they cannot settle, and every row records which layer resolved it.

## 2. Repo layout

```
backend/
  app/
    main.py              FastAPI app, 3 endpoints: POST /analyse, POST /export/pdf, GET /health
    config.py            Pydantic Settings: thresholds, provider, model IDs (env + config/settings.toml)
    models.py            Pydantic models for every structure (Document, Span, Redaction, Section,
                         Profile, Requirement, MatchRow, RunStats, ...)
    parsing/             docling_parser.py, text_parser.py   -> Document(text, lines with char offsets)
    anonymise/           presidio_engine.py, de_patterns.py  -> AnonymisedDocument + Redaction list
    sections/            rules.py (heading lexicon + date-pattern density)
    structuring/         rules.py (dates, employers, degrees), llm_fallback.py
    requirements/        bullets.py (rules), prose.py (LLM)
    matching/            embedder.py, skill_rules.py, matcher.py, quote_check.py
    render/              profile.py, templates/profil.html.j2, pdf.py (WeasyPrint)
    llm/                 base.py (LLMProvider), anthropic_provider.py, ollama_provider.py, prompts/
    logging_safe.py      log filter that drops request bodies and any CV text
  tests/
  pyproject.toml, uv.lock (or pinned requirements.txt), Dockerfile
frontend/                Next.js App Router + TS + Tailwind + shadcn/ui
config/agency.json       name, primary colour, footer text; neutral placeholder ("Muster Personalberatung", grey-blue)
data/synthetic/          20 generated CV PDFs + ground-truth JSON per CV (PII spans, structured fields)
data/jobs/               your 5 job descriptions + gold requirement list per JD
eval/                    run_eval.py, labels.csv (yours), reports/
scripts/gen_synthetic.py generator for the 20 CVs
docker-compose.yml       backend + frontend + ollama
C.md                     what still changes for offline version C
```

## 3. Stack, versions, licences

Versions are what PyPI/npm return today (2026-11-16). They get pinned in a lockfile at step 2; anything that fails to install or conflicts I report instead of silently swapping.

| Part | Choice | Pin | Licence | Note |
|---|---|---|---|---|
| API | FastAPI | 0.163.1 | MIT | |
| Models | Pydantic | 2.14.3 | MIT | every data structure |
| Server | uvicorn | 0.54.0 | BSD-3 | |
| Upload | python-multipart | 0.0.33 | Apache-2.0 | |
| Parsing | Docling | 2.150.1 | MIT | OCR off and table model off by default (CVs are text PDFs; saves RAM). Plain-text paste path skips Docling. |
| Torch | torch, CPU-only wheel | 2.15.0 | BSD-3 | CPU index keeps the image ~1.7 GB instead of ~10 GB (third-party report, to measure) |
| PII | presidio-analyzer / -anonymizer | 2.2.366 | MIT | spaCy NLP engine, German |
| NER | spaCy + `de_core_news_lg` | 3.8.18 / 3.8.0 | MIT / MIT | lg ≈ 570 MB wheel. Labels PER, LOC, ORG, MISC. Download from GitHub releases works from this session. |
| Fuzzy | rapidfuzz | 3.14.6 | MIT | skill alias and heading matching |
| Embeddings | sentence-transformers + `intfloat/multilingual-e5-small` | 6.3.0 | Apache-2.0 / MIT | 384-dim, 12-layer XLM-R based, ~470 MB on disk, needs `query:` / `passage:` prefixes. Alternative: `paraphrase-multilingual-MiniLM-L12-v2` (Apache-2.0, similar size). I pick e5-small; the eval can compare both cheaply. Param count and card details still to confirm on the model card (huggingface.co is blocked here, see §10). |
| PDF out | WeasyPrint + Jinja2 | 70.0 / 3.1.6 | BSD-3 / BSD-3 | HTML template reused for the on-screen profile; needs pango in the Docker image |
| Synthetic CVs | ReportLab + Faker (de_DE) | 5.1.0 / 40.54.0 | BSD / MIT | dev-only dependency |
| Hosted LLM | Anthropic SDK | anthropic 2.8.0 | MIT | model ID from the provider docs, see §6 |
| Local LLM | Ollama + `ollama` Python client | 0.6.3 | MIT | model `qwen3:8b` (Apache-2.0, ~5 GB Q4) with `qwen3:4b` (~2.5 GB) as the small option; tags to confirm on ollama.com (blocked here) |
| Tests | pytest | 9.2.0 | MIT | |
| Frontend | Next.js / React | 16.4.6 / 19.3.1 | MIT | App Router |
| Styling | Tailwind CSS / shadcn CLI | 4.3.5 / 4.23.0 | MIT | Tabs, Table, Badge, Button, Textarea |
| Types | TypeScript | the version `create-next-app@16.4.6` pins | Apache-2.0 | npm latest is 7.0.2 (new major); I will not force it if Next does not support it yet |

Backend Python 3.12 in Docker (safest for torch/spaCy wheels); this container has 3.13, which I will also test.

## 4. Pipeline design

All offsets are character offsets into one canonical string per document.

1. **Parse.** Docling → markdown-ish text; I keep a `lines: list[Line(start, end, text, page)]` table. DOCX through Docling too. Paste fallback: normalise line endings, same `Line` table. Two-column PDFs: Docling's layout model orders reading blocks; this is a known weak spot and a test case.
2. **Anonymise first.** Presidio Analyzer with spaCy `de_core_news_lg` plus custom recognisers:
   - phone: `+49`, `0049`, `(0)`, area codes with `/`, `-`, spaces; mobile `015x/016x/017x`
   - address: street + suffix (`straße|str.|weg|allee|platz|gasse|ring|damm`) + house number, and `\d{5}` PLZ + city
   - date of birth: date near `geb.|geboren|Geburtsdatum|Geburtstag`, plus age (`\d{2} Jahre alt`)
   - URLs: http(s), `www.`, bare `linkedin.com/in/…`, `xing.com/profile/…`, `github.com/…`
   - plus built-ins: email, IBAN; German context words for PERSON
   - AGG-sensitive lines: `Familienstand`, `Staatsangehörigkeit`, `Religion/Konfession`, `Geburtsort`, number of children, photo (Docling images are dropped, counted as "Foto entfernt")

   Output: cleaned text with placeholders like `[NAME]`, `[TELEFON]`, a `Redaction(category, start, end)` list (no original values kept in the response, only category and count, so the "Entfernte Daten" tab shows *what kind* was removed, not the value). An offset map old→new is kept in memory for the request. Nothing downstream ever sees the raw text.
3. **Sections by rules.** Heading lexicon with fuzzy match (Berufserfahrung / Beruflicher Werdegang / Praxiserfahrung / Work Experience …, Ausbildung / Studium / Bildungsweg, Kenntnisse / Skills / IT-Kenntnisse, Sprachen, Zertifikate / Weiterbildung). Fallback signal: density of date ranges. Unassigned lines go to `sonstiges`.
4. **Structuring.** Rules first: date ranges (`MM/YYYY – MM/YYYY`, `seit 2019`, `2018–heute`, `01.2020 - 03.2022`, month names), employer by legal-form suffix (GmbH, AG, SE, KG, e.V., mbH) and position on the line, degrees by lexicon (B.Sc., M.Sc., Diplom-Kaufmann, Bankkaufmann/-frau, Steuerfachangestellte, IHK, Fachinformatiker …). A section counts as "not parsed" when its lines have dates but the rules produced no entry, or coverage is below a threshold. Only those sections go to the LLM, as anonymised text, with a Pydantic JSON schema. Missing fields stay `null`; nothing is invented.
5. **Requirements.** Rules for bullets under headings like "Ihr Profil", "Das bringen Sie mit", "Anforderungen", "Nice to have". Tags by keywords: must (`erforderlich`, `zwingend`, `setzen voraus`, `mehrjährige`, default under the profile heading) vs. nice (`wünschenswert`, `von Vorteil`, `idealerweise`, `Plus`, `gerne`). Prose paragraphs go to the LLM, which must return requirements that are substrings or near-substrings of the JD.
6. **Matching per requirement.**
   - a. e5-small embeds CV lines (anonymised) and the requirement; top 3 by cosine.
   - b. **regel:** a skill/certificate lexicon (aliases, e.g. `SAP FI` = `SAP FI/CO`, `AWS SAA` = `AWS Certified Solutions Architect – Associate`, `Bilanzbuchhalter (IHK)`). If the requirement is *only* a skill or cert term (no years, no level, no "Erfahrung in der Führung …") and an alias hits a CV line by word-boundary regex or rapidfuzz ≥ threshold, then "erfüllt", method "regel", evidence = that line. Requirements with quantities ("5 Jahre Python", "verhandlungssicheres Englisch") do not go through rules.
   - c. **embedding:** best similarity < `SIM_FLOOR` → "nicht belegt", method "embedding", no LLM call.
   - d. **llm:** the requirement + the 3 lines only. Returns `{status, evidence_quote, confidence}` via JSON schema.
7. **Quote check.** `evidence_quote` must be a substring of the anonymised CV text *and* of one of the 3 lines sent. Fail → status "nicht belegt", `quote_check_failed = true`, shown in the UI. Matching is exact after one normalisation only: collapse whitespace runs (Docling line breaks). Open question Q4.
8. **Render.** Kurzprofil (3 lines) built by template from structured data (current/last role, years of experience computed from dates, top skills), no LLM. Then Berufserfahrung, Ausbildung, Kenntnisse, Sprachen. Formal German register, neutral wording. PDF via WeasyPrint from the same Jinja2 template.

`RunStats` per request: rows by method, share without LLM, LLM calls, input/output tokens, cost estimate, latency per stage, provider and model.

Config values `SIM_FLOOR` and `CONF_THRESHOLD` live in `config/settings.toml`, start as placeholders marked "untuned", and are set from the eval (§8).

## 5. LLM provider interface

```python
class LLMProvider(Protocol):
    name: str            # "anthropic" | "ollama"
    model: str
    def generate_json(self, task: PromptTask, schema: type[BaseModel]) -> LLMResult: ...
# LLMResult: parsed: BaseModel, input_tokens, output_tokens, latency_ms, raw_model_id
```

- `PromptTask` is provider-neutral: a task name, German instruction text, and input fields. Each provider turns it into its own wire format. No provider-specific prompt syntax outside `llm/anthropic_provider.py` and `llm/ollama_provider.py`.
- Chosen by one env var: `LLM_PROVIDER=anthropic|ollama`. Model IDs in env (`ANTHROPIC_MODEL`, `OLLAMA_MODEL`).
- Anthropic: structured output with a JSON schema (`output_config.format`), effort `low` for the per-row checks. Current models refuse a disabled-thinking setting, so thinking tokens are billed and counted in the cost.
- Ollama: JSON-schema `format` on `/api/chat`, temperature 0; I will check this against the current Ollama API docs at step 4.
- Both validate the returned JSON with Pydantic; one retry on invalid JSON, then the row becomes "nicht belegt" and is flagged.
- Only anonymised text is ever passed in. A guard asserts the text came from an `AnonymisedDocument` type, so raw text cannot reach a provider by mistake.

## 6. Hosted model and expected cost

Model IDs and prices from the provider docs (platform.claude.com, models overview, read today):

| Model | API ID | Input $/MTok | Output $/MTok |
|---|---|---|---|
| Claude Opus 5.5 | `claude-opus-5-5` | 4.00 | 20.00 |
| Claude Sonnet 5.5 | `claude-sonnet-5-5` | 2.00 | 10.00 |
| Claude Haiku 5.5 | `claude-haiku-5-5` | from 0.10 | from 0.50 |

My default is `claude-opus-5-5`. Picking a cheaper model is your call (Q1); the provider code is the same.

Estimate per CV (one CV against one JD), before any measurement. Assumptions: 12 requirements, about half reach the LLM (6 calls × ~550 in / ~380 out incl. thinking), one prose-JD split call (1,000 / 800), structuring fallback on ~30 % of CVs (1,500 / 1,200).
That gives ≈ 4,750 input and ≈ 3,450 output tokens.

| Model | Per CV | 100 CV×JD pairs (one eval pass) |
|---|---|---|
| Opus 5.5 | ≈ $0.09 | ≈ $9 |
| Sonnet 5.5 | ≈ $0.045 | ≈ $4.50 |
| Haiku 5.5 | ≈ $0.002 | ≈ $0.25 |

Thinking length is the main unknown; plan for up to 2× these numbers. Before the first full hosted run I do a 3-CV pilot, report the measured tokens and cost, and wait for your OK.

## 7. Synthetic data

`scripts/gen_synthetic.py`, seeded, deterministic, no LLM. 20 CVs: 10 IT (Backend, Data, DevOps, SAP-Berater, IT-Support), 10 Finance (Buchhaltung, Controlling, Bilanzbuchhaltung, Steuer, Treasury); junior / mid / senior mixed; 4 with gaps; 4 two-column layouts; 4 messy (inconsistent date formats, missing headings, tables, odd bullets, English headings mixed in). Fictional names, addresses and phones from Faker de_DE; phone numbers from the reserved drama ranges where possible, emails on `example.de`. Every CV gets a ground-truth JSON with the exact PII spans by category and the structured fields, written by the generator, so anonymiser recall and structuring accuracy are scored without hand labelling. The UI and README state the demo runs on synthetic data.

## 8. Eval

- You supply 5 JDs. I propose a gold requirement list per JD (`data/jobs/<jd>.requirements.csv`) for you to approve, so labels point at stable `req_id`s. Requirement splitting is scored separately against that list.
- `eval/labels.csv`: `cv_id, jd_id, req_id, expected_status` (you fill this by hand; I generate the empty sheet).
- Split: 10 CVs dev (tune `SIM_FLOOR` and `CONF_THRESHOLD`), 10 CVs test (report). Thresholds are chosen on dev only (Q5).
- Report (Markdown tables, copied into the README):
  - anonymiser: recall per category, every leak listed (cv, category, offset, context with the leaked value masked in the report)
  - structuring: field-level accuracy (exact for dates, normalised match for employer/degree)
  - requirements: precision/recall of the split against gold
  - match sheet: accuracy per confidence band (regel / embedding have no model confidence and get their own rows) and per method
  - share of rows resolved without an LLM call
  - quote check: rows caught
  - all of the above for both providers
  - cost and latency per CV (p50 / p90)
- No accuracy claim anywhere in README or UI without these tables.

## 9. UI

One page, German labels. Left: CV upload (PDF/DOCX) or text paste, JD textarea, "Analysieren". Right, three tabs: **Profil** (rendered template + "PDF exportieren"), **Abgleich** (table: Anforderung, Muss/Kann, Status, Beleg, Konfidenz, Methode; rows below `CONF_THRESHOLD` badged "bitte prüfen"; quote-check failures flagged), **Entfernte Daten** (counts by category). Footer: active provider and model, share of rows without LLM. Banner: "Demo mit synthetischen Daten". Agency name and colour from `config/agency.json`. No overall score anywhere.

## 10. Hosting, and where the stack does not fit

- **Frontend:** Vercel Hobby. Caveat: Vercel's Hobby plan is for non-commercial use. A demo shown to a client agency may count as commercial; then it needs Pro (Q6).
- **Backend memory estimate:** torch + Docling layout model + spaCy lg + e5-small ≈ 3–4 GB peak RSS (estimate from third-party figures, to be measured at step 2). Image ≈ 2.5–3 GB with CPU torch.
- **Does not fit:** Vercel functions; Render free (512 MB, 0.1 vCPU); Koyeb (no clear free compute tier any more per current reports); Cloud Run free tier only on paper (cold start loading 3 GB of models on each scale-up, card required).
- **Proposal: Hugging Face Spaces, Docker SDK, CPU Basic: 2 vCPU, 16 GB RAM, free, sleeps after 48 h idle.** That fits with room. Risks: forum reports from mid-2026 describe unannounced restrictions on free Docker Spaces, which I could not confirm (the HF docs are blocked in this session); a cold start after sleep takes about a minute; the Space URL is public. Fallback: a small EU VPS (e.g. Hetzner, ~4 GB RAM, a few €/month, EU data location) running the same docker-compose. Price to check on the day.
- **Ollama in the hosted demo:** does not fit any free host. `qwen3:8b` wants ~6 GB RAM on top of the backend and is slow on 2 vCPU. The hosted demo runs on the hosted provider; Ollama runs locally via docker-compose and in the eval.

## 11. Data rules, how they are enforced

- No database, no files written per request: uploads are read into memory, parsed, discarded. Docling temp files (if it insists on a path) go to a `tempfile.TemporaryDirectory` deleted in `finally`.
- Logging: `logging_safe.py` installs a filter on root and uvicorn loggers; access log keeps method, path, status, duration only. Exception handler returns a generic message and logs the exception type, not the message (Pydantic/Docling errors can echo input).
- Only `AnonymisedDocument` text can be passed to `LLMProvider` (type guard + test).
- A test greps the synthetic PII values against every outgoing LLM payload in a mocked run.
- No real CV is committed; `.gitignore` blocks `data/real/` and `*.local.*`; a pre-commit check refuses PDFs outside `data/synthetic/`.

## 12. Version C readiness (designed for, not built)

No hosted-only features (no web search, Files API, batches) in the pipeline; prompt formats only inside provider classes; models downloaded at image build, not at request time; `docker-compose.yml` starts backend, frontend and Ollama. `C.md` will list what still changes (e.g. bundling the Ollama model into the image, offline font files for WeasyPrint, removing the Anthropic dependency from the default image, self-hosted frontend instead of Vercel).

## 13. Risks

1. **Anonymiser recall.** spaCy German NER misses names in odd positions (header lines, file names, email-derived names like `j.mueller@`). Mitigation: rules for header block, email local parts, and the "Name:" pattern; every leak listed in the eval. This is the risk that matters most, since a leak sends PII to a hosted API.
2. **Over-redaction.** spaCy tags employers, universities and tools as PER/ORG/LOC (e.g. "Jenkins", "Hamburg Süd"). Over-redacting employers damages the profile. Needs your decision on what to keep (Q3).
3. **Two-column PDFs.** Reading order may interleave columns; section detection then fails. Measured on the 4 two-column CVs.
4. **LLM confidence is self-reported** and usually poorly calibrated. That is why accuracy is reported per confidence band and the threshold is set from data.
5. **Small eval.** 20 CVs × 5 JDs, with a 10/10 split, gives wide error bars. The README will say so and show counts next to every percentage.
6. **No EU-only inference on the Anthropic API.** `inference_geo` accepts only `global` and `us` today; workspace geo is `us` only. Only anonymised text goes out, but a German agency may still need a DPA review (Q2).
7. **Free hosting is unstable** (see §10).
8. **Ollama eval latency.** This container has 4 CPU, no GPU. `qwen3:8b` on CPU will take seconds per row; the latency table will reflect CPU-only hardware and I will label it that way.
9. **Gendered job titles** ("Buchhalterin") reveal gender after anonymisation. Option: keep as written, or normalise to a neutral form in the profile (Q3).
10. **Session network.** huggingface.co, ollama.com and the Ollama registry are blocked in this cloud session. Without them I cannot download the Docling layout models, the embedding model or any Ollama model, so steps 2, 4 and 6 are blocked until those hosts are allowed (Q7).

## 14. Open questions

- **Q1 Hosted provider and model.** Anthropic with `claude-opus-5-5` (my default), or a cheaper Sonnet/Haiku 5.5? Cost table in §6.
- **Q2 Data location.** Is global/US inference of anonymised text acceptable for the demo?
- **Q3 What gets removed.** Proposal: remove names, contact data, addresses, DOB/age, birthplace, nationality, marital status, religion, photo, URLs. Keep employers, universities, cities of employers. Normalise gendered titles: yes/no?
- **Q4 Quote check.** Exact substring after whitespace collapse only, or also tolerate hyphenation/quote-mark differences?
- **Q5 Threshold tuning.** OK with 10 dev / 10 test CVs?
- **Q6 Vercel plan.** Is Hobby acceptable for this demo given its non-commercial terms?
- **Q7 Network.** May I propose adding `huggingface.co` (+ its CDN hosts), `ollama.com` and `registry.ollama.ai` to this session's environment? I will also need an Anthropic API key, entered through the secure credential form, before the 3-CV pilot.
- **Q8 Ollama model.** `qwen3:8b` (better German, slower) or `qwen3:4b`?
- **Q9 Job descriptions.** When will the 5 JDs arrive? I can draft synthetic ones in the meantime for development only, never for the reported numbers.

## 15. Order of work

1. PLAN.md (this file). Stop for "go".
2. Parsing + anonymiser + tests on 3 synthetic CVs (generator built here, first 3 CVs). Measure RAM.
3. Structuring and profile render, PDF export.
4. Matching (regel / embedding / llm), quote check, both providers.
5. UI.
6. Remaining 17 CVs, eval script, README tables, docker-compose, C.md.

After each step: a three-line status and a stop. Branch `claude/wizardly-bardeen-fs2gwm`, small commits, one PR to `main`.
