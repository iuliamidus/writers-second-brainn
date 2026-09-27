# A Writer's Second Brain

Point it at every blog you've ever written on. Get back a searchable
archive, an interactive graph of everything you've written, and a chat
interface that reads it back to you.

Built for the specific problem of a writer with fifteen years of scattered
posts across four platforms in two languages who wanted to see the shape
of it, but the platform-specific pieces are confined to one directory so
adding new platforms is small.

## Why

Full-text search answers *"where did I write about Norway"*. It cannot
answer *"which of these belong in the same chapter"*, *"what do I keep
coming back to"*, or *"how does the music writing connect to the travel
writing"*. Those are structural questions, and structure is what a graph
is for.

Once the archive becomes a graph, it can also become a *picture* — one
that a writer who has never seen fifteen years of their own output as a
shape tends to find useful.

## What you get

- **A searchable local archive** in SQLite (FTS5, diacritic-insensitive)
- **A graph** projecting into Neo4j: posts, tags, extracted entities, with
  every edge carrying its provenance
- **A vector index** (BGE-M3, multilingual) for semantic and cross-language
  similarity
- **An MCP server** so you can query the archive from Claude Desktop in
  plain language — Romanian, English, whatever you write in
- **An interactive HTML graph** you can share as a single file
- **A deployable web app** (static site + one Vercel Edge Function) with a
  chat interface backed by hybrid search over the whole corpus

## Provenance is not a nicety

Every edge in the graph is tagged with where it came from, and the tiers
are never blurred:

| Tier | Where it comes from | Cost | Trust |
|---|---|---|---|
| `structural` | Platform metadata: tags, dates, self-links, series numbers | Free | Fact |
| `statistical` | Embedding cosine between posts | Cheap | Inference |
| `extracted` | An LLM pass over each post | Paid | Inference |

Each tier is stored and rebuilt independently, so you can always ask the
graph a question using only the edges you actually trust.

Build and query the structural tier first. Tags, dates and self-links
carry more signal than people expect.

## Quickstart

```bash
git clone https://github.com/YOU/writers-second-brain.git
cd writers-second-brain

python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"

cp sources.example.yaml sources.yaml
cp .env.example .env
# edit both for your setup
```

Then the pipeline, in stages:

```bash
wsb detect https://someblog.wordpress.com   # which adapter handles this?
wsb fetch                                    # pull everything into SQLite
wsb stats                                    # what did we get?
wsb search "Norvegia"                        # diacritic-insensitive FTS
```

For the graph:

```bash
docker compose up -d          # Neo4j 5 with the GDS plugin
wsb graph build               # derive structural edges
wsb graph push                # project into Neo4j
```

Open <http://localhost:7474> (`neo4j` / `secondbrain`) and work through
`src/wsb/cypher/queries.cypher`. The first three sections need only
structural edges.

## The paid stages (opt-in)

Everything above is free. The following stages call APIs:

```bash
pip install -e ".[embeddings]"       # BGE-M3 for semantic edges
wsb embed                             # ~5-10 min on CPU for ~700 posts

pip install -e ".[enrich]"           # Anthropic SDK
wsb extract                           # ~$3 for ~700 posts of entity extraction

wsb entities propose-merges data/merges.yaml    # LLM proposes canonicalisations
# open merges.yaml, approve the ones you want, then:
wsb entities apply-merges data/merges.yaml

wsb graph build && wsb graph push    # refresh with the new tiers
```

Rough cost for a 1000-post corpus, one-time: embedding is free (local
model), extraction is ~$3-5 (Claude Haiku), canonicalisation is ~$0.10.

## Chatting with the archive

Install the MCP extra and point Claude Desktop at the server:

```bash
pip install -e ".[mcp]"
```

Edit `~/Library/Application Support/Claude/claude_desktop_config.json`
(create if missing):

```json
{
  "mcpServers": {
    "wsb": {
      "command": "/absolute/path/to/writers-second-brain/.venv/bin/wsb",
      "args": ["mcp"],
      "env": {"WSB_DB": "/absolute/path/to/second_brain.db"}
    }
  }
}
```

Restart Claude Desktop. The archive is queryable in plain language.

## Deploying as a web app

`wsb site build` emits a static site + one Edge Function you can push to
Vercel (or any equivalent host). The server just protects the API key;
the tool calls run in the browser against a baked-in `data.json`.

See `docs/deploy.md` for the details, or the [Sharing it: the hosted
site](#sharing-it-the-hosted-site) section below.

## Adding a platform

Write one class in `src/wsb/sources/` implementing the `Source` protocol
(`describe()` and `fetch()`), register it in `REGISTRY`, and optionally
add a probe to `detect_platform`. Nothing downstream changes.

## Design notes

**Diacritics are load-bearing.** Romanian writers routinely mix diacritics
and bare ASCII in the same corpus, and there are two incompatible Unicode
encodings of ș and ț in circulation (U+015F cedilla vs U+0219 comma
below). Every lexical index folds all of these together. Skip this and
you silently lose half the archive. The same applies to any language with
comparable spelling ambiguity.

**Short posts are the common case.** A photo with a two-word caption is
genuinely undetectable, and a corpus like this is full of them. Posts
under twelve words fall back to the blog's configured `default_lang`
rather than guessing. When embedding, we compose title + tags + caption +
media context rather than embedding the two-word body alone.

**Fetching is separate from normalising.** Raw platform JSON is archived
to `data/raw/` and never touched again. Changing a parsing rule should
never mean re-hitting someone's blog.

**Series ordering comes from the writer.** Titles ending in `- 2`, `(3)`,
`part 2` are the author's own sequencing, so `CONTINUES` edges are
near-certain in a way that inferred edges never are.

## Sharing it: the hosted site

A single-page app that combines the interactive graph and a chat
interface, deployable to Vercel as a static site plus one serverless
function. Non-technical readers get a URL they open on desktop or phone.

```bash
export WSB_SITE_PASSWORD="whatever passphrase you want to share"
wsb site build
```

Writes `index.html`, `data.json`, `api/chat.js`, `vercel.json`. Push to
Vercel; set `ANTHROPIC_API_KEY` and `WSB_SITE_PASSWORD_HASH` (the SHA-256
of the passphrase — visible in the built `index.html`) in the project's
environment variables. The chat runs client-side (tools execute against
the local `data.json`), and only the LLM call round-trips to the proxy.

## Contributing

Small, reviewable commits. When adding a pipeline stage, write the CLI
command and the test alongside it. If a change would break the
"SQLite is canonical, Neo4j is disposable" property, raise it first
rather than working around it.

## Roadmap

- [x] Source adapters: Blogger, WordPress.com, self-hosted WordPress, RSS
- [x] Normalisation, media extraction, diacritic-folded FTS5
- [x] Structural edges and the Neo4j projection
- [x] Embeddings (BGE-M3) and `SIMILAR_TO` edges
- [x] LLM entity extraction with per-kind canonicalisation
- [x] Hybrid retrieval (RRF over FTS5 and dense vectors)
- [x] MCP server for Claude Desktop
- [x] Deployable static site + streaming chat
- [ ] Quote ledger — flag quoted lyrics and passages with sources, because
      what is fine on a blog is a clearance problem in a printed book
- [ ] Community summaries — Leiden via GDS, one LLM-written summary per
      cluster, for corpus-wide questions no single post answers
- [ ] More source adapters: Substack, Ghost, Medium, static-site markdown
- [ ] Comment ingestion — reader response is signal the writer already has

## Licence

MIT.
