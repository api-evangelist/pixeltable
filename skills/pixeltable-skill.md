---
name: Pixeltable
description: Use when building multimodal AI applications, processing images/video/audio/documents, creating RAG pipelines, building agents with tool calling, or exposing data operations as HTTP APIs. Agents should reach for this skill when working with declarative data pipelines, computed columns, embedding indexes, or serving tables over HTTP.
metadata:
    mintlify-proj: pixeltable
    version: "1.0"
---

# Pixeltable Skill

## Product Summary

Pixeltable is an open-source Python library providing declarative data infrastructure for multimodal AI applications. It combines persistent storage, incremental computation, and automatic orchestration in a single system. Agents use Pixeltable to define data pipelines as tables with computed columns, avoiding manual orchestration code.

**Key files and commands:**
- Configuration: `~/.pixeltable/config.toml` (API keys, cache size, time zone)
- CLI: `pxt` (inspect tables, serve endpoints, manage schema)
- Python SDK: `import pixeltable as pxt`
- Primary docs: https://docs.pixeltable.com

## When to Use

Reach for this skill when:

- **Building multimodal pipelines**: Processing images, video, audio, documents alongside structured data
- **Creating computed columns**: Defining transformations (LLM calls, model inference, embeddings) that run automatically on new data
- **Implementing RAG**: Building retrieval-augmented generation with embedding indexes and semantic search
- **Building agents**: Tool calling, persistent memory, MCP server integration
- **Exposing APIs**: Serving table operations and queries as HTTP endpoints without web framework code
- **Iterating on ML workflows**: Testing transformations on sample rows before committing to the full table
- **Managing versions**: Tracking data lineage, rolling back changes, querying historical versions

Do not use Pixeltable for: pure analytics (use a data warehouse), real-time streaming (use Kafka), or applications that don't need persistent storage.

## Quick Reference

### Core Concepts

| Concept | Purpose | Example |
|---------|---------|---------|
| **Table** | Persistent multimodal storage | `pxt.create_table('app/images', {'image': pxt.Image, 'text': pxt.String})` |
| **Computed Column** | Declarative transformation (LLM, model, UDF) | `t.add_computed_column(caption=openai.chat_completions(...))` |
| **View** | Virtual table from iterator (frames, chunks, tiles) | `pxt.create_view('app/frames', t, iterator=frame_iterator(t.video, fps=1))` |
| **UDF** | Custom Python function | `@pxt.udf def my_func(x: str) -> str: ...` |
| **Embedding Index** | Vector search on a column | `t.add_embedding_index('text', embedding=openai.embeddings())` |
| **Query** | Reusable SQL-like function | `@pxt.query def search(q: str): return t.where(...)` |

### Essential CLI Commands

```bash
# Inspect
pxt ls -l                              # list tables with metadata
pxt describe my_dir/my_table           # show schema
pxt columns --computed                 # list all computed columns
pxt rows my_dir/my_table -n 5          # first 5 rows
pxt errors my_dir/my_table             # rows where computed columns failed

# Manage
pxt drop my_dir/my_table -f            # delete table
pxt revert my_dir/my_table --steps 3   # undo last 3 operations
pxt mv my_dir/old_name other_dir       # move/rename

# Serve
pxt serve my-service --config service.toml  # start HTTP service
pxt serve insert --table my_dir.my_table --path /generate --inputs prompt

# Schema
pxt schema diff schema.py my_app       # review changes
pxt schema update schema.py my_app     # apply changes
```

### Configuration (config.toml)

```toml
[pixeltable]
file_cache_size_g = 250
time_zone = "America/Los_Angeles"
verbosity = 1

[openai]
api_key = "sk-..."

[openai.rate_limits]
gpt-4o = 500
gpt-4o-mini = 1000
```

### Python SDK Essentials

```python
import pixeltable as pxt

# Create table
t = pxt.create_table('app/docs', {
    'text': pxt.String,
    'metadata': pxt.Json
})

# Insert data
t.insert([{'text': 'hello', 'metadata': {'source': 'web'}}])

# Add computed column
t.add_computed_column(
    embedding=openai.embeddings(t.text, model='text-embedding-3-small')
)

# Query
results = t.where(t.text.similarity('find docs') > 0.8).limit(10).collect()

# Create view (iterator)
frames = pxt.create_view('app/frames', t, 
    iterator=frame_iterator(t.video, fps=1))

# Add embedding index
t.add_embedding_index('text', embedding=t.embedding)

# Serve
from pixeltable.serving import FastAPIRouter
router = FastAPIRouter()
router.add_insert_route(t, path='/ingest', inputs=['text'], outputs=['text', 'embedding'])
```

## Decision Guidance

### When to Use X vs Y

| Decision | Use This | When | Use That | When |
|----------|----------|------|----------|------|
| **Store data** | Table | Persistent, queryable | View | Virtual, derived from iterator |
| **Transform data** | Computed column | Automatic on new rows | UDF in query | One-off, not stored |
| **Extract frames** | `frame_iterator` | Video → frames | `tile_iterator` | Image → tiles |
| **Split documents** | `document_splitter` | PDFs → chunks | Custom `@pxt.iterator` | Non-standard splitting |
| **Search** | Embedding index | Semantic similarity | SQL `where` | Exact match, filtering |
| **Serve** | `pxt serve` (TOML) | Multiple endpoints, declarative | `FastAPIRouter` (Python) | Custom middleware, auth |
| **Test transform** | `.select().head(3)` | Preview before commit | `.add_computed_column()` | Commit to all rows |
| **Handle errors** | `on_error='ignore'` | Production (skip failures) | `on_error='raise'` | Development (fail fast) |

## Workflow

### Typical Task: Build a RAG Pipeline

1. **Create a documents table**
   ```python
   docs = pxt.create_table('app/documents', {
       'document': pxt.Document,
       'source': pxt.String
   })
   ```

2. **Insert documents**
   ```python
   docs.insert([{'document': 'path/to/file.pdf', 'source': 'web'}])
   ```

3. **Create chunks view** (split documents)
   ```python
   chunks = pxt.create_view('app/chunks', docs,
       iterator=document_splitter(docs.document, chunk_size=512))
   ```

4. **Add embeddings** (computed column on chunks)
   ```python
   chunks.add_computed_column(
       embedding=openai.embeddings(chunks.text, model='text-embedding-3-small')
   )
   ```

5. **Add embedding index** (for semantic search)
   ```python
   chunks.add_embedding_index('text', embedding=chunks.embedding)
   ```

6. **Define search query**
   ```python
   @pxt.query
   def search_docs(query_text: str):
       sim = chunks.text.similarity(query_text)
       return chunks.order_by(sim, asc=False).limit(10)
   ```

7. **Serve as HTTP endpoint**
   ```toml
   [[service.routes]]
   type = "query"
   path = "/search"
   query = "myapp.queries.search_docs"
   ```

8. **Test before deploying**
   ```bash
   pxt serve my-service --config service.toml --dry-run
   pxt serve my-service --config service.toml
   ```

### Typical Task: Add a Computed Column

1. **Test on sample rows first** (don't commit yet)
   ```python
   t.select(t.text, summary=openai.chat_completions(
       messages=[{'role': 'user', 'content': 'Summarize: ' + t.text}]
   )).head(3)
   ```

2. **Review output** — if good, commit to all rows
   ```python
   t.add_computed_column(summary=openai.chat_completions(...))
   ```

3. **Check for errors**
   ```bash
   pxt errors my_dir/my_table --col summary
   ```

4. **Recompute on errors only** (if you fix the UDF)
   ```python
   t.recompute_column('summary', errors_only=True)
   ```

## Common Gotchas

- **Computed columns are stored by default**: Every computed column value is persisted. Use `.select()` without `.add_computed_column()` for one-off transforms.
- **Local UDFs are serialized**: Changes to local `@pxt.udf` functions don't affect existing columns. Only new columns use the updated code. Use module UDFs for live updates.
- **Type hints are required**: All UDF parameters and return values must have explicit type hints. Pixeltable is a database and needs types upfront.
- **Iterators require TypedDict**: Custom `@pxt.iterator` functions must return a TypedDict-annotated dict, not a plain dict.
- **API keys in config.toml**: Store sensitive keys in `~/.pixeltable/config.toml`, not in code. Use environment variables for CI/CD.
- **Batch size matters**: For GPU-heavy UDFs (embeddings, vision), use `@pxt.udf(batch_size=32)` to process multiple rows at once.
- **Embedding indexes are automatic**: Once added, they update incrementally. Don't manually manage them.
- **Views are read-only**: You cannot insert into a view. Insert into the base table; the view updates automatically.
- **Cascade deletes**: Dropping a table with dependent views requires `--cascade`. Check dependencies first with `pxt ls -l`.
- **File cache size**: Set `file_cache_size_g` in config.toml based on available RAM. Default is too small for large media.

## Verification Checklist

Before submitting work with Pixeltable:

- [ ] **Schema is correct**: Run `pxt describe my_dir/my_table` and verify column types and computed column definitions
- [ ] **No errors in computed columns**: Run `pxt errors my_dir/my_table` and confirm the list is empty (or acceptable)
- [ ] **Sample data looks good**: Run `pxt rows my_dir/my_table -n 5` and spot-check values
- [ ] **Embedding index is built**: Run `pxt idxs --embedding` and confirm the index exists and is not stale
- [ ] **API keys are configured**: Run `pxt config` and verify all required API keys are set (show `<redacted>`)
- [ ] **Service config is valid**: Run `pxt serve my-service --config service.toml --dry-run` and confirm no errors
- [ ] **Queries return expected results**: Run `@pxt.query` functions locally and verify output shape and content
- [ ] **Version history is clean**: Run `pxt history my_dir/my_table` and confirm no accidental operations
- [ ] **No hardcoded secrets**: Search code for API keys, passwords, or credentials; move to config.toml or env vars
- [ ] **Documentation is updated**: If adding new tables or queries, update schema.py or README

## Resources

**Comprehensive navigation:** https://docs.pixeltable.com/llms.txt

**Critical documentation pages:**
1. [Quick Start](https://docs.pixeltable.com/overview/quick-start) — 5-minute walkthrough with object detection
2. [Computed Columns](https://docs.pixeltable.com/tutorials/computed-columns) — core pattern for LLM inference and transformations
3. [HTTP Serving](https://docs.pixeltable.com/howto/deployment/serving) — expose tables and queries as REST endpoints
4. [UDFs in Pixeltable](https://docs.pixeltable.com/platform/udfs-in-pixeltable) — write custom Python functions
5. [CLI Reference](https://docs.pixeltable.com/platform/cli) — all pxt commands and flags

---

> For additional documentation and navigation, see: https://docs.pixeltable.com/llms.txt