# @pipeworx/south-dakota-code

South Dakota Codified Laws (South Dakota state statutes) — full section text by
citation, and topic/keyword search over official catchlines.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1866+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `sd_statute(citation)` — full current text of one SDCL section (e.g.
  `"22-16-4"`, murder in the first degree), with the official catchline,
  chapter/title context, and — when the section carries one — the "Source:"
  line listing every session law that enacted or amended it, split out into
  `history`.
- `sd_search(query, limit?)` — topic/keyword search over the Legislature's
  own official section and chapter catchlines across all 71 titles (e.g.
  `"landlord"`, `"concealed pistol permit"`). Not full-text search — South
  Dakota does not publish one.

## Auth

Keyless.

## Data sources

- `https://sdlegislature.gov/api/Statutes/Statute/{citation}` — one section's
  (or chapter's, or title's) current text, as JSON with an `Html` field.
  `sd_statute` calls this live, per request, for a section-shaped citation.
- `https://sdlegislature.gov/api/Statutes/Title` — the list of all 71 titles.
  Used only by `scripts/bake-index.mjs` to drive the per-title fetch below.

## Capabilities — three available, one is not

South Dakota's own publication covers:

- **Citation lookup** — `sd_statute`.
- **Topic/keyword search** — `sd_search`, over catchlines only (see below).
- **Amendments/enactment history** — PARTIALLY AVAILABLE. An in-force
  section's text ends with a "Source: SDC 1939, § 13.2007 (1); SL 1979, ch
  160, § 2; ..." line listing every session law that created or amended it
  (verified live on `22-16-4`). `sd_statute` splits this into `history`
  rather than leaving it only embedded in `text`. A repealed/transferred
  section's one-line disposition (e.g. "22-16-9. Repealed by SL 2005, ch 120,
  § 158, eff. July 1, 2006.") comes back as `disposition_text`, with
  `repealed: true` read from the API's own flag.
- **Historical version** (the text as it read before an amendment) — NOT
  AVAILABLE. sdlegislature.gov publishes only the CURRENT online edition: no
  per-year archive, no "view as of" date picker, and no endpoint returning a
  prior version's text (checked the site's own JS bundle for one — none
  exists). The Source: line says WHEN and by WHICH session law a section was
  amended; it does not return what the text read before that.

## Source and shape — read this before touching the parser

sdlegislature.gov is a Vue single-page app. The August state-law survey
(`docs/state-law-probe.md`) correctly flagged the HTML shell as carrying no
statute — it just stopped one layer too early. The SPA itself is fed by a
**keyless, public JSON API** under `/api/Statutes/` (found by grepping the
site's own JS bundle for `/api/Statutes` — no documentation page exists for
it): `GET /api/Statutes/Statute/{citation}` accepts a section citation
(`"22-16-4"`), a chapter citation (`"22-16"`) or a title number (`"22"`), and
returns one JSON envelope whose `Html` field is that level's content —
full section text for a section, or a plain table-of-contents listing
(chapter header + every section's citation and catchline, with NO body text)
for a chapter or title. No auth, no key, no rate limit observed.

`sd_statute` only ever calls the section-shaped form, live, per request —
never baked.

`sd_search` needs a structural index (every citation + its official
catchline) to search over, and there is no search endpoint and no single
"whole code" listing page the way North Dakota publishes one. The only way to
assemble one is to call the TITLE-level form for each of the 71 titles and
parse its table-of-contents `Html` blob. `scripts/bake-index.mjs` does this
once, offline, into `src/sd-index-data.ts` — never at request time.

**Why the parser works off flattened TEXT, not HTML anchors.** Each title's
Html blob is a concatenation of chapters that were authored and HTML-exported
independently over decades, so anchor markup is NOT uniform: some chapters
link a citation as a modern anchor (`sdlegislature.gov/Statutes?Statute=X`),
others use a legacy `DisplayStatute.aspx?Type=Statute&Statute=X` anchor form,
and repealed-section entries repeat their own citation as a text label
immediately before the real catchline. Rather than chase every anchor shape,
the bake script strips ALL tags (dropping style/script element CONTENTS too —
the first cut of this script leaked raw CSS text into a catchline because
that content sits between tags, not inside one) and then scans the flattened
text for citation-SHAPED TOKENS beginning with the known title number. This
is markup-agnostic: it only depends on the citation strings the site already
prints as plain text, uniform across every era of document. See the script's
own header comment for the full detail, including how duplicate-token
artifacts and the rare genuinely-duplicated citation (observed on <5% of
sections) are handled — never silently, always resolved toward the longer
catchline, since this index drives search RANKING only and `sd_statute`
always re-fetches the real text live.

**Refresh path:** `node scripts/bake-index.mjs > src/sd-index-data.ts`
whenever South Dakota recodifies (new session laws are folded in after each
legislative session). Nothing else in the pack needs to change.

## Known gaps

- `sd_search` matches official catchlines (section and chapter names), not
  full statutory prose — South Dakota does not publish full-text search, and
  this pack does not claim to either. It says so in both the tool
  description and every "no match" response.
- A citation present in the baked index that the LIVE API 404s on (e.g. a
  section renumbered or repealed since the index was last baked) returns a
  loud `reason: "index_live_mismatch"`, never a plain `found: false` with no
  explanation.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "south-dakota-code": {
      "url": "https://gateway.pipeworx.io/south-dakota-code/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/south-dakota-code/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1866+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/sd_statute \
  -H 'Content-Type: application/json' \
  -d '{"citation":"22-16-4"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/sd_statute`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-south-dakota-code` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/south-dakota-code/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "south-dakota-code": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-south-dakota-code"]
    }
  }
}
```

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about South Dakota Code data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
