# @pipeworx/catnap

HIV-1 broadly neutralizing antibody potency (LANL CATNAP: IC50/IC80/ID50 across antibody-virus panels) plus HIV T-cell epitope and antibody-binding-site records (LANL HIV Molecular Immunology Database).

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1863+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `catnap_search_neutralization(antibody?, virus?, limit?)` — antibody-virus neutralization measurements (IC50/IC80/ID50), the antibody's potency compared across viral strains. Censored bounds (`>150`, `<.05`) are returned as `censored: true` with `numeric_value: null` — never coerced to a number.
- `catnap_antibody_info(name, limit?)` — antibody metadata: epitope/binding region, structure (PDB), isolation paper, panel summary stats.
- `immunology_search(table, mab_name?, epitope?, protein_name?, limit?)` — T-cell epitopes (CTL/CD8+ or T-helper/CD4+) or antibody binding sites, queried live from LANL's own API.

## Auth

Keyless. `catnap_search_neutralization` and `catnap_antibody_info` return neutralization/antibody data ingested monthly from LANL's CATNAP bulk files (credentials injected by the gateway). `immunology_search` calls LANL's public JSON API directly on every call — no ingest.

## Data sources

- <https://www.hiv.lanl.gov/components/sequence/HIV/neutralization/index.html> (CATNAP) — antibody-virus neutralization panels (IC50/IC80/ID50), compiled by LANL from published papers. Monthly bulk tab-delimited files (`assay_*.txt`, `abs_*.txt`, `viruses_*.txt`), filename dated at each drop — the ingest scrapes the download page for the current links rather than hardcoding a date. Ingested by `workers/data-pipeline/src/datasets/catnap.ts` on a monthly cadence into `catnap_neutralization`, `catnap_antibodies`, `catnap_viruses` (see `supabase/migrations/170_catnap_neutralization.sql`).
- <https://www.hiv.lanl.gov/mojo/immunology/api/v2> (HIV Molecular Immunology Database JSON API) — CTL/CD8+, T-helper/CD4+ and antibody epitope/binding-site records. Keyless, queried live, no ingest.

## What to get right

- **Censored values are never coerced.** IC50/IC80/ID50 in the source are sometimes a bound (`>150` = not neutralized up to the highest concentration tested; `<.05` = fully neutralized at the lowest concentration tested), not a point estimate. Every measurement carries `_raw` (verbatim string), `_value` (numeric, `null` when censored) and `_censor` (`'<'`/`'>'`, `null` otherwise). A `>50` stored as `50` would invert a potency ranking.
- **Assay context travels with every value.** IC50/IC80 from different papers use different protocols and virus stocks — the tool always returns `reference`/`pubmed_id` and never implies cross-study comparability just because two values are both labelled "IC50".
- **Two different questions live in this one pack.** `catnap_search_neutralization`/`catnap_antibody_info` answer "how potent is this antibody" (CATNAP); `immunology_search` answers "where on the virus does this immune response target, and which HLA restricts it" (Immunology DB) — a different upstream database with its own record IDs.

## Refresh path

`workers/data-pipeline/src/datasets/catnap.ts`, registered in `workers/data-pipeline/src/datasets/index.ts`, `schedule: 'monthly'`. Re-fetches and re-parses all three CATNAP bulk files each run (the neutralization file is ~200k rows / ~16MB, chunk-upserted with a cursor across invocations); no stale-row delete — CATNAP accumulates measurements rather than retracting them.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "catnap": {
      "url": "https://gateway.pipeworx.io/catnap/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/catnap/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1863+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/catnap_search_neutralization \
  -H 'Content-Type: application/json' \
  -d '{"antibody":"10-1074","virus":"CNE8"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/catnap_search_neutralization`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-catnap` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/catnap/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "catnap": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-catnap"]
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
ask_pipeworx({ question: "your question about Catnap data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
