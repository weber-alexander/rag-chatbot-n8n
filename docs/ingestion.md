# 01 · Ingestion Pipeline

Uploads a PDF, parses it page by page, splits it into chunks, embeds them and stores them in MongoDB Atlas.
Every chunk carries the **filename** and **page number** of its source, which is what allows the chatbot to cite exact pages.

← [Back to overview](../README.md)

![Ingestion workflow](../images/ingestion-workflow.png)

---

## Flow

```
Form: Upload File
  → LlamaParse: Submit Job
  → Wait 5s ⇄ LlamaParse: Poll Status ⇄ Job Status?
        ├─ SUCCESS  → LlamaParse: Fetch Result
        ├─ PENDING  → back to Wait 5s
        └─ ERROR / unknown → Stop and Error
  → Mongo: Delete Old Chunks
  → Split Pages + Add Metadata
  → Mongo: Insert Chunks  (Loader + Splitter + Mistral Embeddings)
```

---

## Steps

### 1 · Upload
An n8n form trigger accepts a single PDF. The file is available as the binary property `File`.

### 2 · Parse via LlamaParse API
The PDF is sent to LlamaParse (`parse_page_with_agent`, language `de`) via plain HTTP requests.

**Why not the LlamaParse community node?**
The community node returns the entire document as one concatenated text. Pages are separated by `---`, but markdown tables and horizontal rules use the same pattern, so splitting on it produces wrong page numbers.
The API endpoint `/api/v1/parsing/job/{id}/result/json` instead returns a clean `pages[]` array with an explicit page number per entry.

### 3 · Async polling
Parsing runs as an asynchronous job. The workflow waits 5 seconds, checks the job status and loops until it is finished. Agentic parsing of larger documents typically takes 1–3 minutes.

| Status | Action |
|---|---|
| `SUCCESS` | fetch result |
| `PENDING` | wait and poll again |
| `ERROR` | stop with error message incl. job ID |
| anything else | stop with error (fallback output) |

The fallback output matters: without it, an unexpected status would end the workflow silently as "successful" with nothing stored.

### 4 · Idempotent re-upload
Before inserting, all existing chunks with the same `filename` are deleted:

```json
{ "filename": "<uploaded file name>" }
```

Re-uploading an updated document therefore replaces its content instead of duplicating it.
The node runs with **Execute Once**, so the delete happens a single time per upload.

> Order matters: the delete node replaces the item stream with `{ deletedCount: n }`.
> That's why the next node reads the pages explicitly via `$('LlamaParse: Fetch Result')` instead of from its input.

### 5 · One item per page
A Code node turns the `pages[]` array into one n8n item per page:

```javascript
const filename = $('Form: Upload File').first().json["File"][0].filename;
const pages = $('LlamaParse: Fetch Result').first().json.pages ?? [];
const out = [];

for (const p of pages) {
  const text = (p.md ?? p.text ?? '').trim();
  if (!text) continue;
  out.push({ json: { text, page: p.page, filename } });
}

return out;
```

This is the core idea of the pipeline: the Data Loader runs once per item, so **every chunk created from a page automatically inherits that page's number**. No offset calculations needed.

### 6 · Chunk, embed & store

| Setting | Value |
|---|---|
| Splitter | Recursive Character Text Splitter |
| Chunk size / overlap | 1500 / 200 characters |
| Metadata | `filename`, `page` |
| Embedding model | Mistral `mistral-embed` (1024 dimensions) |
| Collection / index | `knowledge_base` / `vector_index` |

---

## Stored document

The MongoDB Atlas vector store saves metadata as top-level fields:

```json
{
  "_id": "...",
  "text": "**Original Gebrauchsanweisung** ...",
  "embedding": [0.012, -0.034, ...],
  "filename": "Manual.pdf",
  "page": 7,
  "loc": { "lines": { "from": 1, "to": 24 } }
}
```

`loc.lines` is added automatically by the splitter and could be used later for paragraph-level citations.

---

## Configuration notes

- **Parse mode:** `parse_page_with_agent` gives the best results for tables and complex layouts but costs the most credits. For plain text documents a cheaper mode is usually sufficient.
- **Language:** set to `de`. Adjust for documents in other languages.
- **Chunk size:** 1500/200 works well for manuals. For slide decks with little text per page, smaller chunks or one chunk per page can be better.

---

## Known limitations

- Content spanning a page break is split into two chunks.
- The polling loop has no retry limit for jobs stuck in `PENDING`.
- Documents are identified by filename only. Two different files with the same name overwrite each other.
- One file per upload.
