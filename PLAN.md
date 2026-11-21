# TidyBox — Plan

> **This file is the source of truth for the project.** Every design decision lands here first, gets reviewed here, and the code follows it. When something changes, update the relevant section and add a line to the [Decision log](#decision-log).
>

## Status

| Phase | State |
|---|---|
| 0. Accuracy test | not started |
| 1. Index + extract + dry-run CLI | not started |
| 2. Review window + moves + undo + search | not started |
| 3. Learning loop | not started |
| 4. Image themes | not started |
| 5. Public release | not started |

## Goal

A local Windows app that suggests where each item in an "inbox" folder belongs inside a user's folder tree (or whether to delete it), learns from corrections, and lets you search for anything it moved. Everything runs locally: no cloud, because inboxes contain tax returns, passports and bank statements.

Companion to FolderBadges: the badge shows how much is waiting; the sorter helps clear it. Separate repo, light link (a "Sort Inbox…" tray item).

## Principles

- **Suggest, don't act.** Nothing moves until the user accepts. Deletes go to the Recycle Bin. Every batch is undoable.
- **Cheap first.** Rules and plain parsers before the model; the model only for what's actually unclear.
- **Your filed folders are the training data.** No fine-tuning; similar already-filed items show where new ones go.
- **Measure, don't guess.** An accuracy test on already-filed files picks models and settings.

## Test setup (real Inbox untouched)

- `Desktop\SorterTest\Inbox` — **copies** of 30–50 items, outside SubDesktop:
  - copies of already-filed files (CV, visa, payslip, book) → known answers
  - files whose names say nothing (scans, WhatsApp images, screenshots, `image.png`)
  - a video and an audio file
  - a whole folder (e.g. an `AUDIT 20xx` copy)
  - junk: exact duplicate, `(1)` copy, zip + its extracted folder, installer, drafts and query exports
  - things that fit nowhere (recipe, appliance manual)
  - related groups (photos from one day, two versions of a document)
- Optional `SorterTest\expected.csv` (`file,expected_location,notes`) as the answer key.
- SubDesktop is **read-only** until moving is tested on a copy of a few folders.

## Stack (current leaning)

```
Parsers (PdfPig, OpenXML, csv)  ─┐
Windows metadata + ffmpeg frames ─┼─► one small Qwen-VL model ─► item card ─► SQLite + FTS5
                                  │
Images (and video contact sheets) ┴─► image embedding model ─► vectors ─► SQLite (themes, similarity)

optional later: text embedding model, Whisper
```

- Language: C# / .NET 10. Solution `TidyBox.slnx`: `TidyBox.Core` (no UI), `TidyBox.Cli` (dry run + accuracy test, exe `tidybox`), `TidyBox.Tests` (xUnit). UI project added in phase 2.
- Model runtime: LLamaSharp (llama.cpp); fallback to a local llama.cpp server or Ollama if multimodal support lags.

## Pipeline

### 1. Index — SQLite catalogue

| Table | Holds |
|---|---|
| `items` | path, parent, name, type, size, dates, content hash, status (`filed`/`inbox`/`sealed`/`ignored`) |
| `folders` | description, sample names, naming pattern (`{Person} CV {yyyyMMdd}`), **child_axis** (what subfolders are split by), sealed flag |
| `extractions` | text / OCR / captions, keyed by content hash (moves don't re-extract) |
| `fts` | FTS5 over names, paths, extracted text, card keywords |
| `embeddings` | *(optional)* `(item_id, model_id, vector)` |
| `entities` | people / organisations / places + alternative spellings + linked folders |
| `decisions` | suggested vs chosen, which tier decided it, confidence |
| `rules` | pattern → target, usage count, override count |
| `moves` | undo log, grouped by batch |

- Database indexes on `path`, `hash`, `parent_id`.
- Kept fresh by a file watcher plus a check on startup (modified time and size); the hash turns delete-plus-add into a move.
- **Sealed folders** (never looked inside, count as one item): `.git`, `node_modules`, many `.exe`/`.dll`, season folders, app installs.
- **child_axis**: the LLM labels each folder's subfolders once: `topic | person | organisation | year | date | doc-type | place | project | event`.

### 2. Extract — every item becomes a text card (~1–2k characters, capped)

| Type | Cheap first | Only if still unclear |
|---|---|---|
| PDF | built-in text | render page 1 → model OCR |
| docx / xlsx / pptx | OpenXML text | n/a |
| csv / json / sql | header, sample rows, keys | n/a |
| Image | photo metadata (date, GPS, device), name timestamp | model caption + OCR |
| Video | name parsing, Windows metadata | frames → one contact sheet → model |
| Audio | ID3 tags | *(later)* Whisper on first 60s |
| zip / 7z | list contents | n/a |
| exe / msi / apk | version info | n/a |
| Folder (moved as one unit) | summary of children | n/a |

The card holds: summary, keywords and synonyms, entities, document type, date. LLM cards are only generated for **Inbox items**. Filed items use path + name + extracted text, which keeps indexing fast.

**Video caps**
- A fixed 6–8 frames whatever the length; skip the first and last ~5%.
- Shrink the frames and tile them into **one 3×3 contact sheet**, so one model call per video.
- Fast seeking to the nearest full frame (no full decode).
- Frames go to a temp folder and are deleted afterwards; only text is stored.
- A time budget per file (~30s), then fall back to metadata only.
- ffmpeg is downloaded on first use (not shipped); try Windows' built-in frame grabbing first.

### 3. Shortlist — combine signals

1. Votes from similar filed items (full-text match, or embeddings if added later)
2. Folder profile match
3. Exact-word match (FTS5): names, entities
4. Entity boost
5. Rules and recently used folders

Combined with reciprocal rank fusion → **top 8 folders + their parents**. Confidence = how far the top candidate is ahead + how strongly similar items agree.

### 4. Decide — cascade

```
group related items → rules → strong match → LLM → "not sure, ask me"
```

- Grouping: name-stem matches, photos taken close together, a zip next to its extracted folder → decided together.
- Delete suggestions mostly come from rules (duplicate hash, `(1)` copies, extracted installers, drafts, superseded versions).
- LLM input: card + 8 candidates (path, description, example names, child_axis, naming pattern) + 3 most similar past corrections.
- LLM output forced by a grammar: `{action, target, new_subfolder, rename, confidence, reason}`; **target must be one of the candidates**.
- Nesting and new folders follow child_axis (e.g. `Work/CVs` is split by person → find the person → that subfolder, or create it).

### 5. Review

List grouped by target, coloured by confidence, keyboard-first, searchable folder picker, "accept all high-confidence", reason shown.

### 6. Act + learn (the loop)

- Moves as one transaction per batch, with undo log; name clashes handled; Recycle Bin for deletes.
- Accepted items **become filed examples straight away** → the next similar item benefits.
- Corrections → saved and used as worked examples in future prompts; count against the wrong target.
- Rule mining: N identical decisions, 0 overrides → offer a rule; overridden rules are demoted.
- Implicit feedback: manual moves out of the Inbox (seen by the file watcher) count as labelled examples.
- Earned autopilot: automatic moves offered per folder or rule only after a high acceptance rate (e.g. 98% over the last 50).
- The `decisions` table doubles as the accuracy test's history (replay against a new model or settings).

## Search ("where did it go?")

- FTS5 over names, paths, extracted text, card keywords (+ embeddings if added).
- Query expansion by the LLM (optional).
- **Recently moved** view from the move log (old → new location, undo).
- Open the file or show it in Explorer; tray search box + global hotkey.

## Images — grouping by theme

**Out of scope:** organising a photo library by holiday or moment. That's what photo apps (Google Photos, Apple Photos, Immich) are for, and people mostly do it by hand.

**In scope:** loose images that land in the inbox, such as WhatsApp images, screenshots, phone photos of documents and saved pictures, grouped and routed **by theme**. This is also how image embeddings get built and tested.

1. **Embed every image** with an image embedding model (see [Image embedding models](#image-embedding-models)). Video contact sheets get embedded too.
2. **Assign a theme** by comparing against text prompts for a starter list of themes, no training needed:
   `document photo · ID / card · receipt / invoice · screenshot · chat screenshot · meme · people · food · scenery / travel · product / shopping · diagram / slide · other`
3. **Route by theme:**
   - **Document-like themes** (document photo, ID, receipt, screenshot with text) → OCR and caption with the main model → go through the **normal document pipeline** (e.g. a photographed CNIC → `Documents/CNICs`). This is probably the biggest win for a messy inbox.
   - **Pictures** (people, food, scenery…) → matched against image folders the user already has (the average vector of each folder's images, e.g. `Photos/Marathon Photos`); otherwise suggested as a theme folder.
4. **Discover new themes:** images that match no theme well are clustered. The main model captions 3–4 images per cluster and names the cluster; the user can keep it as a new theme or folder.
5. **Near-duplicates and bursts** → perceptual hash (no model), offered as "keep the best of these?".
6. **Batches:** images with timestamps close together (from the file name, e.g. `WhatsApp Image 2025-11-05 at 11.08.01`) and the same theme are suggested together.

## Models

### Main model

One small multimodal model (Qwen-VL 2B/4B class, Q4) for cards, OCR, captions, decisions, folder descriptions and child_axis labels. One download (~3–4 GB at 4B), one model in memory. Loaded only while sorting, unloaded when idle.

### Embedding models (optional — only if full-text search + cards falls short)

What matters: small and runs on CPU · handles ≥500 tokens · multilingual (Urdu / mixed scripts) · instruction-aware · shrinkable vectors (Matryoshka) · licence allows release · available as ONNX/GGUF for C#.

| Model | Size | Context | Multilingual | Instructions | Shrinkable | Licence | Notes |
|---|---|---|---|---|---|---|---|
| **Qwen3-Embedding-0.6B** | 0.6B (~400 MB Q8) | 32k | ✅ strong | ✅ | ✅ 32–1024 | Apache 2.0 | Best quality here; same family as main model; GGUF |
| **EmbeddingGemma-300M** | 308M (<200 MB quantized) | 2k | ✅ 100+ | task prompts | ✅ 768→128 | Gemma terms | Built for on-device use; likely pick for CPU-only |
| nomic-embed-text v1.5 | 137M | 8k | ❌ mostly English | prefixes | ✅ | Apache 2.0 | Long context; has a matching image model |
| bge-m3 | 568M | 8k | ✅ | ❌ | ❌ | MIT | Also produces keyword-style vectors; heavier |
| multilingual-e5-small | 118M | 512 | ✅ | prefixes | ❌ | MIT | Small multilingual baseline |
| **bge-small-en-v1.5** | 33M | 512 | ❌ | ❌ | ❌ | MIT | Tiny baseline: "is bigger worth it?" |
| all-MiniLM-L6-v2 | 22M | 256 | ❌ | ❌ | ❌ | Apache 2.0 | Too-short context; skip |
| ~~jina-embeddings-v3~~ | 570M | 8k | ✅ | ✅ | ✅ | **Non-commercial** | Licence blocks release |

**To test:** Qwen3-Embedding-0.6B, EmbeddingGemma-300M, bge-small-en-v1.5.
**Storage:** 256 dimensions × 50k files ≈ 50 MB (25 MB at half precision).

### Image embedding models

Needed for [image themes](#images--grouping-by-theme). Must put images and text in the **same space**, so theme prompts ("a photo of a receipt") can be compared against images directly.

| Model | Size | Text side | Licence | Notes |
|---|---|---|---|---|
| **SigLIP 2** (base / so400m) | ~375M / ~1.1B | short prompts only | Apache 2.0 | Strong at matching images to prompts; multilingual; ONNX available |
| **nomic-embed-vision v1.5** | ~92M vision + nomic text v1.5 | **full 8k text model** | Apache 2.0 | Shares a space with nomic-embed-text v1.5, so **one space for documents and images** |
| **Qwen3-VL-Embedding-2B** | ~2B | full text | Apache 2.0 | Text, image and video in one space; same family as the main model; heaviest |
| OpenAI CLIP ViT-B/32 | ~150M | 77-token prompts | MIT | Old but tiny; a good baseline |
| MobileCLIP | very small | short prompts | check licence | Built for phones; licence may block release |
| ~~jina-clip-v2~~ | ~900M | full text | **Non-commercial** | Licence blocks release |

**To test:** SigLIP 2 base, nomic-embed-vision v1.5, CLIP ViT-B/32 as baseline.

**Note:** if nomic or Qwen3-VL-Embedding wins, the same model could also cover the optional text embeddings, which would mean one embedding space for everything.

### Hardware presets

Detect GPU / VRAM / RAM at first run → preset (CPU-only / ~6 GB / 12 GB+) → optional 30-second accuracy test on the user's own filed files to confirm. Every model can be disabled (fall back to metadata / OCR).

## Memory and disk budget

| Component | RAM | Disk |
|---|---|---|
| SQLite + FTS5 | ~10–50 MB | ~50–300 MB / 50k files |
| Embeddings (if used) | ~25–80 MB | same |
| Embedding model (if used) | ~150–500 MB | ~100–400 MB |
| Main model (4B Q4 + vision add-on) | ~3–4 GB, while sorting only | ~3–4 GB |
| Image embedding model | ~0.3–1.5 GB, while indexing images only | ~0.2–1.5 GB |

Idle (tray + search): ~100–200 MB.

## Users with no structure

- Cluster at the top level only (UMAP + HDBSCAN on embeddings, or group by card tags) → the LLM names the clusters or matches them to a template (life areas / PARA / Johnny Decimal) → the user edits the tree → those folders become fixed starting points.
- Deeper levels are built by axis (person / year / event), not by clustering.
- Guiding the clustering: seeded clustering, constraints from corrections ("belong together" / "don't"), instruction-aware embeddings, a per-user classifier.

## Phases

0. **Accuracy test** on SubDesktop (leave-one-out) + `SorterTest`.
1. **Index + extract + dry-run command-line tool** (prints suggestions, moves nothing).
2. **Review window + moves + Recycle Bin + undo + search.**
3. **Learning loop** (corrections, rule mining, autopilot).
4. **Image themes** (image embeddings, theme assignment, document-photo routing, new-theme discovery, near-duplicates). The image embedding model can be brought in earlier if the accuracy test shows images are the weak spot.
5. **Public release** (structure for users without one, first-run setup, hardware presets, model download).

## Open questions

- User's GPU / VRAM.
- Scope: Inbox only, or also tidy what's already in SubDesktop?

## Decision log

Newest first. One line per decision: what was decided, and why.

| Date | Decision | Why |
|---|---|---|
| 2026-11-21 | Project named **TidyBox**; repo at `SubDesktop\Code\tidy-box`; plan moved here as `PLAN.md` | Name chosen by user |
| 2026-11-21 | C# / .NET 10 throughout, no Python prototype | One stack, matches FolderBadges; LLamaSharp, PdfPig, OpenXML and Windows OCR all available |
| 2026-11-12 | Images grouped **by theme**, not by holiday or moment | Organising events is a photo-app job and usually manual; themes route loose inbox images (especially photographed documents) and get image embeddings working |
| 2026-11-12 | Image embedding model is part of the core plan (phase 4, earlier if needed) | Needed for theme grouping; shortlist: SigLIP 2, nomic-embed-vision, CLIP baseline |
| 2026-11-12 | Text embeddings optional; start with FTS5 + LLM-written cards | Easier to inspect, no extra model; add embeddings only if the accuracy test says so |
| 2026-11-12 | One small multimodal Qwen-VL model for cards, OCR, captions and decisions | One download, one model in memory, one thing to benchmark |
| 2026-11-12 | Videos capped: 6–8 frames → one contact sheet, frames not stored, ~30s budget | Long videos must cost the same as short ones |
| 2026-11-12 | Search built in (FTS5 + "recently moved" view) | Users need to find what the sorter moved |
| 2026-11-12 | Develop against `Desktop\SorterTest\Inbox`, never the real Inbox; SubDesktop read-only | Safety while building |
| 2026-11-12 | Separate repo from FolderBadges, light link via tray menu | Different scope and much larger download |
| 2026-11-12 | Suggest-only by default; Recycle Bin for deletes; undo log | Trust in the app depends on it |
