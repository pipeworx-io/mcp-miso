# @pipeworx/miso

MISO (Midcontinent ISO) real-time electricity data — fuel mix, load, five-minute
LMPs, wind/solar forecast-vs-actual and transmission binding constraints for the
15-state footprint from Louisiana to Minnesota plus Manitoba.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1683+ live data sources.

## Tools

- `miso_fuel_mix(period?)` — MW generated per fuel category (coal, gas, nuclear,
  wind, solar, hydro, battery storage, other) plus the footprint total.
  `current` (default) = the latest 5-minute interval; `today` / `yesterday` = the
  whole day.
- `miso_lmp(region?, node?, limit?)` — five-minute LMP $/MWh per pricing node with
  loss and congestion components and every ancillary-service clearing price
  (regulation, mileage, spinning, supplemental, short-term reserve, ramp up/down).
  ~1,600 nodes across the Central, North and South regions.
- `miso_binding_constraints(kind?)` — the flowgates currently binding, with their
  shadow price. `transmission` (default), `subregional` or `reserve`.
- `miso_load()` — today's actual load hour by hour against the medium-term load
  forecast for the same hours, plus the five-minute total-load series.
- `miso_wind_solar()` — hourly wind and solar forecast vs actual MW.

## Auth

Keyless. No registration, no key, no rate-limit documented.

## Data sources

- <https://public-api.misoenergy.org/> — the live public API. Endpoint index is
  rendered as HTML at the root of that host.
- <https://www.misoenergy.org/markets-and-operations/rtdataapis/> — MISO's own
  page for these feeds.

## Traps

**The endpoint everyone still cites is a silent zero.** MISO's long-documented
Real-Time Data Broker —

```
https://api.misoenergy.org/MISORTWDDataBroker/DataBrokerServices.asmx
  ?messageType=getfuelmix&returnType=json
```

— still resolves, still answers **HTTP 200**, and returns

```json
{"error": "no data", "See": "https://www.misoenergy.org/markets-and-operations/rtdataapis"}
```

for every data `messageType` (`getfuelmix`, `getlmpconsolidatedtable`,
`gettotalload`, …). Only `getapiversion` still carries a payload. Any client that
checks `res.ok` reports success forever while returning nothing — the failure
class in `docs/silent-zero-policy.md`. Verified dead 2026-09-17. This pack talks
only to `public-api.misoenergy.org`, and `misoGet` throws on a body carrying a
string `error` so a future retirement fails loudly instead of silently.

Other things worth knowing:

- **Every number is a string.** `"34168"`, and in the `/api/Snapshot` display
  feed comma-grouped as `"99,978"`. `num()` coerces and returns `null` — never
  `0` — for a blank, so "no reading" never reads as "zero megawatts".
- **Timestamps are EST year-round.** MISO does not shift the market clock for
  daylight saving; the payload labels this explicitly.
- **`/api/FuelMix` is one interval, not a series.** Use `/Today` or `/Yesterday`
  for the full day.
- **`MediumTermLoadForecast` uses `HourEnding`, not `Hour`** — unlike `ClearedMW`
  right beside it, which uses `Hour`. Keying both off `Hour` silently produces an
  all-null forecast column.
- **`/api/WindSolar/GetCombined` returns future hours too**, with actuals that
  are absent rather than meaningful. This pack emits `null` for an actual whose
  hour has not happened, rather than reporting 0 MW of wind at 11pm.
- **`getapiversion` is gzipped without negotiation**, so a raw `curl` without
  `--compressed` prints binary. Workers' `fetch` decompresses automatically.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "miso": {
      "url": "https://gateway.pipeworx.io/miso/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/miso/mcp` returns the tools in the table
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
curl -X POST https://gateway.pipeworx.io/v1/tools/miso_fuel_mix \
  -H 'Content-Type: application/json' \
  -d '{}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/miso_fuel_mix`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "miso": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-miso"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-miso
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Miso data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
