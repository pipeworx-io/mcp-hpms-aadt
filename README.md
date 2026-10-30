# @pipeworx/hpms-aadt

US road traffic volume (AADT, annual average daily traffic) per roadway section, by state and data year, from the Federal Highway Administration's Highway Performance Monitoring System.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1863+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `aadt_near_point(lat, lon, state, radius_m?, year?, limit?)` - road segments within a radius of a point, busiest first.
- `aadt_by_route(state, route, county?, min_aadt?, year?, limit?)` - segments whose route_id starts with a prefix, e.g. "I089" (Vermont), "IH0035" (Texas), "SHS_101" (California).
- `aadt_top_segments(state, county?, f_system?, year?, limit?)` - the busiest segments in a state, optionally by county or functional class (1 Interstate to 7 Local). Also shows a state's route_id naming style.
- `aadt_layers(state)` - which data years exist for a state and which columns each carries.

Every response carries `data_year`, `data_as_of`, `source`, a caveat, `total_matching`, `returned` and `truncated`. A query that matches no segment with a reported AADT fails with a narrowing hint rather than returning an empty list.

## Auth

Keyless.

## Data sources

- FHWA (U.S. Department of Transportation), Highway Performance Monitoring System (HPMS) - public-release state datasets, one per state and data year (50 states, DC, PR; data years 2020 and 2022-2025 for nearly all). US federal data, public domain. Answered live on every request; nothing is copied.

## Things to know

- AADT is the value for a roadway section as reported by the state to FHWA, partly sampled and expanded. It is not a raw counter reading, and rows are segments (begin and end milepost), not count stations.
- `route_id` is state-specific free text. There is no national route key, so "I-95 across states" is not one query. A highway can carry several ids in one state (for example a mainline and a frontage designation).
- `state` is required: county codes and route ids only mean something within a state, and the pack does not guess a state from a point.
- The newest data years drop the route number and route name columns, and one state drops section length. The pack reads each year's column list and asks only for columns that exist, because requesting a missing column is a hard error upstream.
- Many segments have no reported AADT; every query filters to segments that do. Sorting without that filter puts the empty ones first.
- Large states can be slow on a cold row count, so `total_matching` is best-effort; when it does not arrive in time it is null and `truncated` is inferred from the page size.
- Duplicate-looking rows can occur (the same section reported more than once); they are returned as published.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "hpms-aadt": {
      "url": "https://gateway.pipeworx.io/hpms-aadt/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/hpms-aadt/mcp` returns the tools in the table
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
curl -X POST https://gateway.pipeworx.io/v1/tools/aadt_near_point \
  -H 'Content-Type: application/json' \
  -d '{"lat":34.0522,"lon":-118.2437,"state":"CA","radius_m":1000,"limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/aadt_near_point`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-hpms-aadt` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/hpms-aadt/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "hpms-aadt": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-hpms-aadt"]
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
ask_pipeworx({ question: "your question about Hpms Aadt data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
