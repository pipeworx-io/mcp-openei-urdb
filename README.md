# @pipeworx/openei-urdb

NREL's Utility Rate Database (URDB) — US electricity tariffs: residential,
commercial and industrial rate schedules with their energy charges by tier and
time-of-use period, demand charges, fixed monthly charges, net-metering
treatment and effective dates, keyed to the utility's EIA id.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1683+ live data sources.

## Tools

- `urdb_search_rates(address? | latitude+longitude | utility | eia_id, sector?, detail?, approved_only?, limit?, offset?, _apiKey?)` —
  search tariffs.
- `urdb_rate_detail(label, _apiKey?)` — the full structure of one tariff,
  including the 12x24 month-by-hour period schedules a bill calculation needs.
- `urdb_utilities_for_location(address? | latitude+longitude | filter, _apiKey?)` —
  which utilities serve a location, with tariff counts and how many are current.

## Auth

The OpenEI API takes an `api_key`. NREL's public **`DEMO_KEY` is accepted and
returns real rows** (verified 2026-09-17), so this pack works with no
configuration at a low rate limit — it is not a gated pack.

Callers who exceed the DEMO_KEY limit pass their own free NREL key as
`_apiKey`. Free key: <https://developer.nrel.gov/signup/>

A platform key would be read from the injected `_platformKey` argument
(`PLATFORM_NREL_KEY`). It is **not** declared as `platformKeyEnv` in the pack
manifest and must not be until the secret actually exists: the gateway is at
Cloudflare's 250-text-binding cap, and declaring the flag without the secret
would route rate questions to a tool that could only refuse.

## Data sources

- <https://api.openei.org/utility_rates?version=latest> — rate search and detail.
- <https://api.openei.org/utility_companies?version=latest> — utility name directory.

`developer.nrel.gov/api/utility_rates/v3.json` was not reachable from our
egress on 2026-09-17 (connection failure, not an HTTP error) and is not used.

## Traps

Every one of these returns HTTP 200.

- **`radius` silently zeroes a lat/lon search.** `lat=37.5202&lon=-122.2758`
  alone returns PG&E rates; the identical call plus `&radius=25` returns
  nothing. This pack never sends it.
- **Geographic lookup has real coverage holes.** Houston returns zero rates by
  lat/lon *and* by ZIP 77002 — Texas retail-choice tariffs are largely absent
  from URDB. That is a coverage gap, not an error, and the tools say so instead
  of handing back a bare empty array.
- `address` is geocoded upstream and works for some ZIPs and not others. When
  it returns nothing, lat/lon or `utility` usually works.
- `sector` combined with `address` narrowed several queries to zero.
- `startdate`/`enddate` are **UNIX seconds**, and URDB keeps **superseded**
  tariffs. A rate whose end date has passed is historical; `is_active` says so,
  and the tools warn when every result is superseded.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "openei-urdb": {
      "url": "https://gateway.pipeworx.io/openei-urdb/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/openei-urdb/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1683+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/urdb_search_rates \
  -H 'Content-Type: application/json' \
  -d '{"utility":"Pacific Gas & Electric Co","sector":"Residential","limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/urdb_search_rates`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "openei-urdb": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-openei-urdb"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-openei-urdb
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Openei Urdb data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
