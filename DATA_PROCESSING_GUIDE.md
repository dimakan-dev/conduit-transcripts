# Data Processing Guide

This guide covers the end-to-end pipeline using the existing Python scripts:

1. Pull episode audio from RSS
2. Transcribe with Whisper
3. Create OpenSearch index
4. Ingest transcript chunks + embeddings into OpenSearch

It includes both modes:
- standard mode (no timestamps)
- timestamp mode (stores chunk timestamps and YouTube metadata)

## 1) Prerequisites

- Python environment available (project currently uses `venv/`)
- FFmpeg installed (required by Whisper)
- OpenSearch cluster available
- `.env` file configured with:
  - `OPENSEARCH_SERVICE_URI`
  - `INDEX_NAME`
  - optional: `INDEX_NAME_WITH_TIMESTAMPS` (otherwise defaults to `{INDEX_NAME}_timestamps`)

## 2) Install Python dependencies

If your environment is not already prepared, install packages required by scripts in `src/`.

```bash
pip install \
  typer httpx python-slugify python-frontmatter \
  langchain-text-splitters langchain-huggingface \
  sentence-transformers opensearch-py python-dotenv \
  arrow beautifulsoup4 python-dateutil openai-whisper
```

> `openai-whisper` downloads model files on first use.

## 3) Transcribe episodes from RSS (download + transcription)

`src/transcribe.py` downloads episode audio and transcribes it in one step. You do not need a separate download step for the normal pipeline.

### 3.1 Latest episode only

```bash
python src/transcribe.py rss --url "https://media.rss.com/vector-podcast/feed.xml" --latest --show "vector-podcast"
```

### 3.2 Range of episodes

```bash
python src/transcribe.py rss --url "https://media.rss.com/vector-podcast/feed.xml" --range "1-10" --show "vector-podcast"
```

### 3.3 All episodes

```bash
python src/transcribe.py rss --url "https://media.rss.com/vector-podcast/feed.xml" --all --show "vector-podcast"
```

### 3.4 Skip already-transcribed episodes

```bash
python src/transcribe.py rss --url "https://media.rss.com/vector-podcast/feed.xml" --all --show "vector-podcast" --skip-if-exists
```

### 3.5 Optional: transcribe a local downloaded audio file

```bash
python src/transcribe.py file "audio/vector-podcast/episode-file.mp3"
```

## 4) Choose one processing mode

---

## Option A: Standard Mode (No Timestamps)

### A1) Generate transcripts

Run transcription without `--with-timestamps`. Output goes to:

- `transcripts/<show-slug>/*.md`

Example:

```bash
python src/transcribe.py rss \
  --url "https://media.rss.com/vector-podcast/feed.xml" \
  --range "1-10" \
  --show "vector-podcast" \
  --skip-if-exists
```

### A2) Create OpenSearch index

Creates/recreates index from `INDEX_NAME`.

```bash
python src/os_index.py
```

### A3) Ingest transcripts into OpenSearch

Ingest all transcripts:

```bash
python src/os_ingest.py ingest --all --show "vector-podcast"
```

Ingest one episode:

```bash
python src/os_ingest.py ingest --episode "episode-slug" --show "vector-podcast"
```

---

## Option B: Timestamp Mode (With Timestamps)

### B1) Generate timestamp-enabled transcripts

Add `--with-timestamps`. Output goes to:

- `transcripts_with_timestamps/<show-slug>/*.md`

Each file includes `whisper_segments` in frontmatter.

Example:

```bash
python src/transcribe.py rss \
  --url "https://media.rss.com/vector-podcast/feed.xml" \
  --range "1-10" \
  --show "vector-podcast" \
  --skip-if-exists \
  --with-timestamps
```

### B2) Create timestamp-enabled OpenSearch index

Creates/recreates timestamp index (default: `{INDEX_NAME}_timestamps`, or `INDEX_NAME_WITH_TIMESTAMPS` if set).

```bash
python src/os_index.py --with-timestamps
```

### B3) Ingest timestamp-enabled transcripts into OpenSearch

Reads from `transcripts_with_timestamps/` and writes to timestamp index.

```bash
python src/os_ingest.py ingest --all --show "vector-podcast" --with-timestamps
```

Single episode:

```bash
python src/os_ingest.py ingest --episode "episode-slug" --show "vector-podcast" --with-timestamps
```

## 5) Notes on timestamp behavior

- Timestamp mode attempts to:
  - extract YouTube URL/video ID from the episode description
  - map each chunk to Whisper segment start time
- Additional indexed fields in timestamp mode:
  - `youtube_url`
  - `youtube_video_id`
  - `timestamp`
  - `chunk_index`
- If a mapping cannot be found for a chunk, `timestamp` may be `null`.

## 6) Useful validation steps

Verify transcript files were created:

```bash
ls transcripts/vector-podcast | head
ls transcripts_with_timestamps/vector-podcast | head
```

Verify index exists (from Python/OpenSearch client):

```bash
python -c "import os; from dotenv import load_dotenv; from opensearchpy import OpenSearch; load_dotenv(); c=OpenSearch(os.getenv('OPENSEARCH_SERVICE_URI'), use_ssl=True, timeout=100); print(c.indices.get_alias('*'))"
```

## 7) Typical end-to-end command sets

Standard mode:

```bash
python src/transcribe.py rss --url "https://media.rss.com/vector-podcast/feed.xml" --all --show "vector-podcast" --skip-if-exists
python src/os_index.py
python src/os_ingest.py ingest --all --show "vector-podcast"
```

Timestamp mode:

```bash
python src/transcribe.py rss --url "https://media.rss.com/vector-podcast/feed.xml" --all --show "vector-podcast" --skip-if-exists --with-timestamps
python src/os_index.py --with-timestamps
python src/os_ingest.py ingest --all --show "vector-podcast" --with-timestamps
```
