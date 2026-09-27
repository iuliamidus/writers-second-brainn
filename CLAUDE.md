# CLAUDE.md

Context for Claude Code working in this repository.

## What this is

A pipeline that ingests a writer's blog archive from multiple platforms and
turns it into a searchable corpus plus a navigable graph. Typical audience
is a writer with several blogs, possibly in multiple languages, wanting to
see patterns and connections across years of scattered posts.

Two audiences the design serves:

1. **The writer.** Non-technical. Interacts through `wsb search`, Neo4j
   Browser, the exported HTML graph, or Claude Desktop via MCP. Everything
   they touch should work without explanation.
2. **The developer.** Building this as a portfolio piece or extending it.
   Prefers understanding the mechanism over hiding it.

When those conflict, keep the mechanism visible but the defaults sane.

## Architecture

```
blogs ──▶ Source adapters ──▶ normalise ──▶ SQLite ──▶ edge builders ──▶ Neo4j
          (per platform)      (once)        (canonical)                  (disposable)
```

**SQLite is the source of truth.** Neo4j is a projection, rebuilt with
`wsb graph push --reset`. Never write state to Neo4j that does not exist
in SQLite, and never make the pipeline depend on Neo4j being up. Someone
should be able to use the archive and full-text search with no graph
database at all.

**Edges carry provenance and are never blurred.** Every edge has a `source`
of `structural` (platform metadata — a fact), `statistical` (embedding
similarity — an inference) or `extracted` (LLM output — an inference).
`Store.replace_edges` swaps one tier without touching the others. Preserve
this. Being able to query using only trusted edges is a core design
property, not an implementation detail.

## Layout

| Path | Role |
|---|---|
| `src/wsb/models.py` | Dataclasses every stage speaks. Change carefully. |
| `src/wsb/sources/` | The only place platform-specific code belongs. |
| `src/wsb/normalize.py` | HTML → text, media extraction, diacritic folding. |
| `src/wsb/db.py` | SQLite schema and storage. Schema lives in `SCHEMA`. |
| `src/wsb/graph/edges.py` | Edge derivation from stored data. |
| `src/wsb/graph/neo4j_loader.py` | SQLite → Neo4j projection. |
| `src/wsb/graph/export.py` | Standalone HTML export of the graph. |
| `src/wsb/site/` | Deployable web app: template, JS proxy, build. |
| `src/wsb/mcp_server.py` | MCP tools for Claude Desktop integration. |
| `src/wsb/extract.py` | LLM entity extraction. |
| `src/wsb/canonicalize.py` | LLM-driven entity merge suggestions. |
| `src/wsb/embeddings.py` | BGE-M3 dense embeddings for the statistical tier. |
| `src/wsb/search.py` | FTS, semantic, and RRF hybrid retrieval. |
| `src/wsb/cypher/queries.cypher` | Annotated query book, ordered by pipeline stage. |
| `sources.yaml` | Which blogs to ingest (gitignored — copy from example). |

## Non-obvious constraints

**Diacritic folding is load-bearing, not a nicety.** Romanian mixes diacritics
and bare ASCII freely — `carti` and `cărți` in the same corpus — and there are
two incompatible Unicode encodings of ș/ț in circulation (U+015F cedilla,
U+0219 comma-below). `normalize.fold()` collapses all of them. Any new lexical
index, entity normaliser or dedupe key must run through it. Getting this wrong
loses half the archive silently, which is the worst kind of bug here. The
same principle applies to any language with comparable spelling ambiguity.

**Short posts are the common case, not an edge case.** Many posts are one
photo and a caption. Language detection is unreliable below ~12 words and
falls back to the blog's `default_lang`. Embeddings compose title + tags +
body + media captions rather than embedding the two-word body alone.

**Never re-fetch to fix a parsing bug.** Raw platform JSON is archived to
`data/raw/`. Normalisation must be re-runnable from disk. Be polite to
these servers; they are someone's blog, not an API product.

**Series ordering comes from the author.** `CONTINUES` edges are derived
from titles the writer numbered themselves, which makes them near-certain
in a way inferred edges are not. `_series_key` is deliberately
conservative — it rejects short stems so `Life 2` is not treated as a
series.

**LLM outputs need human review.** `wsb entities propose-merges` writes
to a YAML file the user edits; `apply-merges` only touches approved rows.
Never bypass this with a "just apply everything" shortcut — LLM
canonicalisation confuses parent/child places, common first names, and
same-artist different-works often enough to matter.

## Conventions

- Python 3.11+, `from __future__ import annotations`, type hints throughout.
- Comments explain *why*, never *what*. If a line needs a comment
  describing what it does, rewrite the line.
- Pipeline stages are separate CLI commands and independently re-runnable.
  Do not collapse them into one `run` command for convenience.
- New dependencies for the embedding, LLM, MCP and site stages go in the
  relevant `optional-dependencies` group, never in the base install. The
  fetch + FTS core must stay cheap to install.
- `ruff` for lint, line length 100. `pytest` with `pythonpath = ["src"]`.

## Commands

```bash
wsb detect <url>                       # which adapter handles this blog
wsb fetch                              # pull everything into SQLite (slow, hits network)
wsb stats                              # corpus summary
wsb search "<query>"                   # diacritic-insensitive full-text search
wsb embed                              # BGE-M3 vectors (requires [embeddings])
wsb extract                            # LLM entity extraction (requires [enrich])
wsb entities propose-merges FILE       # write cross-language merge proposals
wsb entities apply-merges FILE         # apply reviewed merges
wsb graph build                        # derive all edge tiers into SQLite
wsb graph push                         # project into Neo4j
wsb graph export                       # standalone interactive HTML
wsb site build                         # emit deployable site to repo root
wsb mcp                                # run the MCP server over stdio

docker compose up -d                   # Neo4j 5 + GDS on :7474 (neo4j / secondbrain)
pytest -q
```

## Working style

Prefer small, reviewable commits over large ones. When adding a pipeline
stage, write the CLI command and the test alongside it. If a change would
break the "SQLite is canonical, Neo4j is disposable" property, stop and
raise it rather than working around it.

The corpus scale is ~1000-5000 documents and runs comfortably on a laptop.
Do not add a task queue, distributed anything, or a hosted database
dependency. Complexity here is a cost with no matching benefit.
