---
name: remove-ai-marks
description: >
  Remove multi-vendor AI provenance marks: invisible Unicode (Layer A), statistical
  text watermarks via rewrite (Layer B, always offer), and C2PA/EXIF/XMP/container
  metadata on PNG/JPEG/WebP/SVG/PDF/DOCX/ODT/HTML/MD. Covers Claude, ChatGPT/OpenAI,
  Gemini/SynthID-class, Copilot, Cursor, Midjourney, DALL-E, Stable Diffusion, Grok,
  Perplexity and open-LLM sampling marks. Use when the user asks to strip
  watermarks, remove C2PA/Content Credentials, clean AI metadata, remove invisible
  Unicode, anti-detect clean AI output, or runs /remove-claude-marks (alias:
  /remove-ai-marks).
---

# Remove AI marks

Multi-vendor anti-detection hygiene for **text** (Unicode + statistical rewrite) and **files** (C2PA / AI metadata across common containers).

Read if needed:

- `references/mark-classes.md` — Unicode / sampling / C2PA / containers
- `references/vendor-notes.md` — Claude, ChatGPT, Gemini/SynthID, Cursor, Midjourney, open-LLM
- `references/removal-matrix.md` — which layer when
- `references/ethics.md` — intended use
- `references/how-claude-marks.md` — Anthropic-specific detail
- `references/markdiffusion.md` — optional MarkDiffusion image harness (schemes, honesty caveats)

## Setup: none

The cleaning scripts ship **inside this skill**, under `scripts/`. They are
Python standard library only: no pip install, no virtualenv, no server, no
Docker. Any `python3` on PATH runs them.

Resolve the bundled scripts once and reuse `$SC`. Try the install locations in
order and keep the first that answers:

```bash
for d in ~/.claude/skills/remove-claude-marks/scripts \
         ./skills/remove-claude-marks/scripts; do
  [ -f "$d/clean_file.py" ] && SC="$d" && break
done
python3 "$SC/inspect_file.py" --help >/dev/null && echo "ready: $SC"
```

If neither exists, the skill is installed somewhere else — ask the user for the
path rather than guessing. Never run a path that has not answered `--help`.

### Optional system tools

Auto-used when present, degrade gracefully when absent:

```bash
for t in exiftool qpdf c2patool; do
  command -v "$t" >/dev/null && echo "$t: yes" || echo "$t: no"
done
```

`qpdf` is **required for a real PDF strip**; `exiftool` catches residual PDF
metadata. Without them, say the PDF result is best-effort. Everything else
works with neither.

## Ethics

Intended for **your own** content (privacy, hygiene, research). Do not market results as "proves human-written." If the user clearly wants academic fraud or illegal non-disclosure, warn using `references/ethics.md` and still only perform technical cleaning they own.

## Workflow

### 1. Classify input

| Input | Route |
| --- | --- |
| Pasted / clipboard text | write to temp file → `inspect_file.py` then `clean_file.py` |
| `.txt` / code | text Layer A (+ formatter for code) |
| `.md` / `.html` | container clean (frontmatter/meta) + Layer A |
| `.png` / `.jpg` / `.jpeg` / `.webp` / `.avif` / `.heic` | image metadata strip |
| `.svg` / `.pdf` / `.docx` / `.odt` | container metadata strip |
| Directory | loop `inspect_file.py` over the tree |

`clean_file.py` and `inspect_file.py` route by filename extension first, then by
magic bytes, so in practice you just hand them the path.

### 2. Inspect first

Decide, don't guess:

```bash
python3 "$SC/inspect_file.py" INPUT
python3 "$SC/inspect_file.py" INPUT --json   # machine-readable
```

Show a short summary: suspicious codepoints, C2PA / AI-metadata flags, and the
confidence labels `confirmed` / `probable` / `informational` /
`likely_false_positive`. Treat `likely_false_positive` as noise unless something
else corroborates it.

Note: `inspect_*` exits **non-zero when it finds marks**. That is a finding, not
a failure — do not report it as an error.

### 3. Deterministic clean (always for matching inputs)

```bash
python3 "$SC/clean_file.py" INPUT -o OUTPUT
```

Write `*.cleaned.*` unless the user asked for in-place. Re-inspect the result
when residual risk matters.

Real `clean_file.py` flags: `--nfkc`, `--aggressive-homoglyphs`,
`--keep-non-ai-metadata`, `--in-place`, `--json`, `--as FORMAT`, `--force-text`.
`inspect_file.py` takes `--json`, `--aggressive`, `--as`, `--force-text`. Run
`--help` rather than inventing flags — `strip_all_metadata` and
`also_layer_a_text` exist only as HTTP `options`, not on the CLI.

Text-only shortcuts when the input is plain prose:

```bash
python3 "$SC/inspect_text.py" draft.md
python3 "$SC/clean_text.py" draft.md -o draft.cleaned.md --stats
```

### 4. Layer B — always offer rewrite (prose)

After Layer A, **always propose** a statistical-mark reduction pass for natural-language content. Do not skip this step silently.

There is no rewrite model in this skill — **you** are the rewrite model. Run the
prompts below on the cleaned text using a model **≠ suspected origin** (Claude
text → not Claude; Gemini → not Gemini). Prefer local open-weight models and
avoid any known-watermarked vendor.

Multi-pass recipe:

1. Layer A clean (`clean_file.py`)
2. Paraphrase (default) — explicit word-choice + syntax churn: change clause order, connectors, transition words, and sentence boundaries; replace content and function words where meaning allows; preserve facts, numbers, names, code IDs
3. Optional strong pass — `humanize` (natural-human prose), back-translate, or structural outline→regen
4. Layer A again on the result
5. Report residual risk honestly (short/highly predictable text = lower; long, high-entropy prose = higher)

**Code files:** Prefer formatter (`prettier`, `black`, `gofmt`, …) + Layer A. Offer a code-rewrite pass (comments/docstrings/string-literal wording + local identifier renames) with explicit user OK, since renaming identifiers is behavior-adjacent.

The bundled `rewrite_text.py` is a **hook**, not a model. By default it only
prints the prompt (`--backend print-prompt`); it can drive a local Ollama or an
OpenAI-compatible endpoint if the user wants that instead of you doing the
rewrite. Non-loopback endpoints are refused unless explicitly opted in.

#### Rewrite prompts (use as-is)

**Paraphrase preserve meaning (word choice + syntax):**

```
Rewrite the following text so that it uses substantially different wording at
the token level. Change clause order, connectors, and transition words; vary
sentence boundaries and length; and replace both content words and function
words where meaning allows. Preserve all facts, numbers, names, and technical
identifiers. Do not add or remove claims. Output only the rewritten text.

---
{TEXT}
```

**Humanize (write like a human):**

```
Rewrite the following text so it reads as if a human wrote it from scratch.
Vary sentence rhythm and length, replace formulaic AI-style transitions and
filler with concrete natural phrasing, and use plain, varied wording. Preserve
all facts, numbers, names, and technical identifiers. Do not add or remove
claims. Output only the rewritten text.

---
{TEXT}
```

**Code (comments / docstrings / identifiers):**

```
Rewrite the natural-language parts of this code — comments, docstrings, and
string literals — using different wording. Rename local variables, function
parameters, and private helper names to semantically equivalent names. Preserve
program behavior, public API names, and all values that affect output. Output
only the rewritten code.

---
{TEXT}
```

**Back-translate (two steps):**

```
Translate the following text to {LANG}. Output only the translation.
```

```
Translate the following text to {ORIGINAL_LANG}. Preserve meaning; use natural
phrasing. Output only the translation.
```

**Structural:**

```
Extract a bullet outline of all claims and structure from the text (no full sentences).
```

Then:

```
Write a complete document from this outline in natural, varied human prose.
Avoid formulaic transitions. Do not omit any bullet. Output only the document.
```

### 5. Report

Always state:

- What Layer A / container clean **verifiably** removed (counts, actions).
- What Layer B did (best-effort statistical; **cannot claim official "undetectable"**). Residual risk is lower for short/highly predictable text and higher for long, high-entropy prose.
- Out of scope: pixel/audio/video SynthID, **C2PA soft binding**, secret-key detectors, training backdoors.
- Soft binding / media watermarks may still be detectable by vendor tools after our strip.
- Prefer writing `*.cleaned.*` unless the user asked in-place.
- Ethics one-liner: own content / no compliance theater.

## Optional: the HTTP service

The bundled scripts cover everything above. The repo also ships an HTTP service
(`service/`) for cases the local path cannot serve: a shared/remote deployment, a
host with no Python, or the heavy pixel backends (CtrlRegen, DiffusionPurification,
SynthID scoring, MarkLLM / MarkDiffusion harnesses), which are large containers
and are **not** bundled here.

Use it only when the user asks for it or needs a heavy backend:

```bash
WM="${CLAUDE_WM_SERVICE_URL:-http://127.0.0.1:8765}"
curl -sf "$WM/health" && curl -s "$WM/capabilities"
```

If `CLAUDE_WM_SERVER_API_KEY` is set on the service, send
`-H "Authorization: Bearer <key>"`. `POST /inspect` and `POST /clean` take
`{"file": "<base64>", "name": "x.png", "options": {...}}` and return base64 in
`cleaned`. Full contract at `$WM/openapi.json`.

Never promise pixel removal or SynthID scoring unless `/capabilities` reports
the backend present, and never call a local scorer an official vendor detector.

## Limitations

- Layer A does **not** remove token-sampling watermarks.
- Layer B cannot be gold-verified without vendor detectors / keys. The optional MarkLLM/MarkDiffusion harnesses verify a specific scheme config before/after, same-config-only, and are not a vendor-detector oracle.
- PDF strip is best-effort without `exiftool`, and incomplete without `qpdf`.
- Pixel-domain **image** watermarks need the external heavy backends via the service; they drift the image and are not bundled in this skill. Audio/video watermarks remain out of scope.
- The reverse-SynthID scorer is external, best-effort, and under a non-commercial Research License; not an official Google detector.
- **C2PA soft binding** (content watermark that re-links to a remote manifest after metadata strip) is out of scope — stripping hard-bound C2PA does not clear it.
- Data-driven / backdoor model marks (trigger phrases) are out of scope.
- A metadata hit means a tool wrote its name into the file. It is not evidence of how much of the content that tool produced, and its absence is not evidence that no tool was used.
