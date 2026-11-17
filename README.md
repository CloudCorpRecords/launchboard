# launchboard

Keep a finger on what is launching in AI right now, without scraping launch sites or maintaining a catalogue yourself. launchboard is a small forwarding client for the hosted Lanzamientos IA MCP service, a daily catalogue of newly launched AI tools: the tool list is answered from your own machine using captured definitions, and each call travels upstream, with the service's answer handed back exactly as it arrived.

Four of the service's six captured tools are exposed. Search the catalogue by free text, open one listing's full public detail — description, launch date, revenue signals, for-sale status — pull live same-category alternatives to a well-known product, and read the published list of optional paid add-ons a maker can buy to promote a listing. Two captured tools are deliberately left out. `submit_tool` submits a new listing on behalf of its maker and needs the maker's email address to do it; `create_checkout` creates a Stripe checkout URL for a paid add-on, which originates a purchase. This rebuild carries nobody's email upstream and starts no checkouts. `list_offers` stays because it only reads a published price list — it buys nothing itself.

## The tools

- **`search_tools`** — Search the catalogue of live, published AI tools with a free-text `query`. Matching is a case-insensitive substring over tool name, tagline and category name, and at most 20 results come back. The natural first call when someone wants to find, discover or compare tools by keyword, use-case or category.
- **`get_tool`** — The full public detail for one listing, addressed by its `slug` (the tail of the `toolUrl` that `search_tools` returns, so `/tools/acme-writer` means slug `acme-writer`): full description, launch date, revenue signals whether verified or self-reported, and for-sale status. A slug that does not resolve to a live listing comes back as null.
- **`list_alternatives`** — Live, same-category alternatives to a well-known AI tool named by `target` — its slug, such as `chatgpt`, `notion-ai` or `figma`, not a slug from this catalogue. For "what are the alternatives to X" questions.
- **`list_offers`** — The optional paid add-ons a maker can buy to promote a listing: name, a one-sentence statement of the effect, and the price in USD. Listing itself is always free and the add-ons are always labelled "Sponsored"; during free early access the offers are not yet purchasable and the result says so. No arguments.

## Install and run

As an MCP server via npm:

```bash
npx -y @reneturcios/launchboard mcp
```

MCP client configuration:

```json
{
  "mcpServers": {
    "launchboard": {
      "command": "npx",
      "args": ["-y", "@reneturcios/launchboard", "mcp"]
    }
  }
}
```

As a container:

```bash
docker run -i ghcr.io/cloudcorprecords/launchboard
docker run --rm ghcr.io/cloudcorprecords/launchboard search_tools '{"query":"agents"}'
```

## CLI

A single binary covers both the command line and the MCP server. Every tool is a subcommand whose one argument is a JSON string of that tool's inputs:

```bash
launchboard search_tools '{"query":"writing assistants"}'
launchboard get_tool '{"slug":"acme-writer"}'
launchboard list_alternatives '{"target":"chatgpt"}'
launchboard list_offers '{}'
launchboard --version
```

Exit codes: `0` for a successful forwarded call, `1` when the result that comes back is an error, and `2` for usage mistakes like an unknown subcommand or a JSON argument that does not parse. `submit_tool` and `create_checkout` are not subcommands and exit `2` like any other unknown name.

## Configuration

- `LANZAMIENTOSIA_MCP_URL` — Replaces the upstream endpoint. Defaults to the pinned hosted endpoint, `https://lanzamientosia.com/api/mcp/mcp`. (This override variable is a reconstruction introduced by this rebuild.)

The Lanzamientos IA registry record lists no headers for its remote, so launchboard never attaches an authentication header — every call goes upstream keyless.

When the upstream cannot answer — it returns a JSON-RPC error, answers with a non-2xx status, sends a body that cannot be read, or the request itself fails — the caller receives an error result whose text begins with `launchboard error:`.

## How forwarding works

The first call of a process opens a session against the upstream: an `initialize` request (protocol version `2025-03-26`), a `notifications/initialized` notification, and then the `tools/call`. If the service issues an `Mcp-Session-Id` header during initialize, launchboard repeats it on subsequent requests and keeps reusing that one session until the process ends. Replies are understood in both plain JSON and server-sent-events framing. Successful results are passed through untouched, `isError` flag and any extra fields included.

## Development

```bash
npm install
npm run build   # tsc -> dist/ (dist/cli.js is the bin)
npm test        # tsx --test test/*.test.ts — stubbed upstream, no network
npm run dev     # run the CLI from source
```

The suite compares the served tool list against the captured surface tool for tool, asserts the omitted `submit_tool` and `create_checkout` tools are absent at the registry, MCP and CLI levels, walks the session handshake and its reuse, proves no `Authorization` header is sent under any environment, and confirms that arguments travel upstream unchanged and results return unchanged.

## License

MIT — see `LICENSE`. Copyright (c) 2026 Rene Turcios.
