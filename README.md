# @pipeworx/epa-aqs

EPA's Air Quality System — the quality-assured, regulatory-grade archive of
ambient air measurements from ~10,000 US monitoring sites. Distinct from the
`airnow` pack, which serves provisional real-time AQI for the last few days.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1866+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `aqs_list_parameters(list, pc?, state?, county?)` — parameter codes, classes, states, counties, sites, agencies.
- `aqs_sample_data(param, bdate, edate, by?, state?, county?, site?, cbsa?, bbox?, limit?)` — raw measured sample values.
- `aqs_annual_summary(param, bdate, edate, ...)` — annual means, maxima, percentiles, exceedance counts, completeness.
- `aqs_monitors(param, bdate, edate, ...)` — monitor metadata, networks, methods, open/close dates.

## Auth

**BYO only today.** EPA AQS authenticates with **two** values, an email and a
key, issued together. A pack gets one credential slot, so both halves travel in
`_apiKey`:

- `?_apiKey=<email>:<key>` — email and key together, or
- `?_apiKey=<key>` with `email=<your-email>`.

Register free and instantly: `GET https://aqs.epa.gov/data/api/signup?email=<your-email>`.
EPA emails the key to that address.

The platform fallback env var is **`PLATFORM_EPA_AQS_KEY`**, and it must be
stored in the `<email>:<key>` form. It is **not provisioned** as of 2026-09-17
(fleet #2212) — the gateway is at Cloudflare's 250-text-binding cap, so it goes
in through the platform key store rather than `wrangler secret put`.

There is **no keyless or demo path**. EPA's own documentation names a shared
test account (`test@aqs.api` / `test`); it is permanently over its daily quota —
verified 2026-09-17, it answers
`{"error":"Daily limit for account use exceeded. Retry later."}` — so it is
deliberately not wired in. Serving that as data would read as an outage of ours.

## Data source

`https://aqs.epa.gov/data/api` — docs at https://aqs.epa.gov/aqsweb/documents/data_api.html

Things the next person would otherwise rediscover:

- **Dates are `YYYYMMDD`** with no separators. The tools take `YYYY-MM-DD`.
- **`bdate` and `edate` must be in the SAME calendar year.** EPA's rejection does
  not say so plainly; this pack checks first and says it.
- **Trouble arrives as HTTP 200.** Authentication and quota problems come back in
  `Header[0].status` or as a bare `{"error": ...}`, never as a 4xx. Both are
  raised as errors here rather than as an empty `Data` array.
- **`param` takes at most 5 codes per request.**

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "epa-aqs": {
      "url": "https://gateway.pipeworx.io/epa-aqs/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/epa-aqs/mcp` returns the tools in the table
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
curl -X POST https://gateway.pipeworx.io/v1/tools/aqs_list_parameters \
  -H 'Content-Type: application/json' \
  -d '{"list":"classes"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/aqs_list_parameters`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "epa-aqs": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-epa-aqs"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-epa-aqs
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Epa Aqs data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
