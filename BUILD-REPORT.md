# BUILD REPORT — launchboard

Clean-room rebuild of `com.lanzamientosia/lanzamientosia` v1.0.0
(remote-only MCP server at `https://lanzamientosia.com/api/mcp/mcp`),
built 2026-10-28 as a forwarding client. Package: `@reneturcios/launchboard`
0.1.0, registry name `io.github.CloudCorpRecords/launchboard`.

## Surface built

Labelled subset: **4 of 6 captured tools built**, 2 omitted (see below).

| Tool | Required fields as served |
|------|---------------------------|
| `search_tools` | `query` |
| `get_tool` | `slug` |
| `list_alternatives` | `target` |
| `list_offers` | *(none — no `required` key in the capture, served verbatim)* |

Each served tool carries exactly the keys `name`, `title`,
`description`, `inputSchema`, with the `inputSchema.$schema` marker
(`http://json-schema.org/draft-07/schema#`) exactly as captured.
`search_tools` / `get_tool` / `list_alternatives` each carry one
string property (`query` / `slug` / `target`, `minLength: 1`, with
the captured descriptions) and their captured `required` arrays;
`list_offers` carries empty `properties` and no `required` key. No
tool carries `additionalProperties` — none was captured, none was
invented. The tool literals were generated programmatically from
`harness/fixtures/lanzamientosia-tools-original.json` by the script
at `/tmp/laneb6-launchboard-scratch/gen.py` (parse + re-serialise,
spliced into `src/forward.ts` between the TOOL_DEFS_GENERATED
markers) — never retyped — and the generator's own round-trip check
plus a byte-level comparison confirmed the spliced block is
identical to the generator output and deep-equals the fixture's 4
tool objects, in order. Tools are registered via the MCP SDK
low-level Server API (`ListToolsRequestSchema` /
`CallToolRequestSchema` handlers), not McpServer+zod.

## Omissions — `submit_tool` and `create_checkout`

- OMITTED: `submit_tool` — submits a new AI tool listing on behalf
  of its maker and requires the maker's `email` (required fields:
  name, url, description, category, email): a listing submission
  carrying a person's email, per the standing no-send rule.
- OMITTED: `create_checkout` — creates a Stripe checkout URL for
  one paid add-on on one listing: it originates a purchase/checkout
  flow, per the standing no-checkout-origination rule.
- Absence is asserted in tests at three levels for BOTH tools:
  registry level (`TOOL_DEFS` / `TOOL_NAMES` / `isKnownTool`), MCP
  layer (an in-memory-transport `tools/list` response carries
  exactly the 4 built tools and deep-equals the fixture;
  `handleCallTool` and an MCP `tools/call` for each omitted tool
  are rejected before any network hop), and CLI level
  (`launchboard submit_tool '{}'` and
  `launchboard create_checkout '{}'` each exit 2 with usage and
  are not listed among the commands).
- KEPT: `list_offers` — a published price read (add-on name,
  one-sentence effect, price in USD). Reading a published price
  list is keepable even though the prices belong to the omitted
  checkout's products; the kept surface's core (catalogue search /
  fetch / alternatives) is a faithful standalone catalogue. Its
  captured description mentions `create_checkout` in prose; that
  text is served verbatim as captured, and no checkout tool is
  registered.

## Fixture conflicts

**None.** The spec and the fixture agree on the tools' names,
titles, descriptions, the `required` arrays (`query` / `slug` /
`target`, and none for `list_offers`), and the property shapes.
The fixture already excludes both omitted tools, exactly as the
spec's Omissions section describes.

## Labelled reconstructions (from the spec's Wire section)

- Upstream endpoint pinned: `https://lanzamientosia.com/api/mcp/mcp`.
- Env override `LANZAMIENTOSIA_MCP_URL` — labelled reconstruction.
- The registry record declares NO headers on the remote: no auth
  header is ever sent, under any environment. Tests assert that no
  `Authorization` header (and no `X-API-Key` header) is sent with an
  empty environment, with a `LANZAMIENTOSIA_API_KEY`-style variable
  present, and with `AUTHORIZATION` / `API_KEY` /
  `LANZAMIENTOSIA_TOKEN` variables present.
- Error rendering `launchboard error:` text for upstream failures
  — labelled reconstruction.
- Upstream results pass through verbatim, including `isError` and
  extra fields. tools/call is forwarded as streamable-HTTP JSON-RPC:
  initialize (protocolVersion `2025-03-26`) →
  notifications/initialized → tools/call, honouring `Mcp-Session-Id`
  when issued; plain-JSON and SSE replies are both parsed. The
  forwarding core takes an injectable fetch; tests stub the
  upstream — no live network calls in tests, and no live call is
  claimed beyond the coordinator's tools/list capture.

## Deviations

- **Lockfile / build / dist-based parity: NOT completed in the
  builder session — environment write-guard (see Lockfile).**
  Everything else in this report was verified in-session. The four
  remaining commands are listed verbatim in the Lockfile section
  for a rerun in a session where the guard does not fire; the build
  directory has been left in the exact clean state they expect
  (no `node_modules`, no `package-lock.json`, no `dist/`).

## Lockfile

**Status: BLOCKED in the builder session; rerun required.**

- Attempted first, exactly as prescribed (Route 3, clean dir, no
  `node_modules`, no pre-existing lockfile):
  `npm install --package-lock-only --ignore-scripts --offline`
  → `ENOTCACHED` for `@modelcontextprotocol/sdk`.
- Root cause, diagnosed firsthand: (1) the shared npm cache's
  packument buckets for the fleet dependency set were shadowed by
  newer full-metadata entries (`accept: application/json`, response
  `vary: accept-encoding, accept`), so the install-accept entry no
  longer satisfies npm's request; and (2) every cache lookup runs
  `cacache.index.compact`, which rewrites the bucket file — and in
  this builder session the VM write guard EPERMs every exec write
  of read-/network-derived bytes (reproduced in isolation:
  `cacache.put` of the cached packument bytes → EPERM; extraction
  of a cached tarball's first file → EPERM; a 400 KB literal string
  write → OK). The documented Route 2 fallback (explicit
  credential-free proxy + `--registry https://registry.npmjs.org`)
  was attempted three times and failed at the same guard: EPERM
  writing the fetched packument body into the cache, leaving a
  truncated 0-byte `package-lock.json` (removed).
- Expected on rerun (fleet-standard tree, per LOCKFILE-ROUTE.md and
  both B5 siblings): parses, `name` == `@reneturcios/launchboard`,
  exactly **128** package entries.
- Rerun, in order, in
  `/home/hatch/workspace/mcp-rebuild-100/builds/launchboard`:
  1. `npm install --package-lock-only --ignore-scripts --offline`
  2. verify the lockfile parses, name and 128 entries; record sha256
  3. `npm ci --ignore-scripts --offline`; re-verify sha256 unchanged
  4. `npm run build` and `node dist/cli.js --version` (expect `0.1.0`)

## server.json

- Description exactly: "Search a daily catalogue of newly launched AI tools, fetch details, alternatives and offers." — **92 characters**.
- npm package entry carries `"version": "0.1.0"`; the OCI entry
  (`ghcr.io/cloudcorprecords/launchboard:0.1.0`) carries **no**
  `version` field. Both entries carry packageArguments
  `[{type: positional, value: mcp}]` and transport stdio. Dockerfile
  (node:20-alpine, both stages) carries the label
  `io.modelcontextprotocol.server.name="io.github.CloudCorpRecords/launchboard"`.
  Container workflow `IMAGE` is `ghcr.io/cloudcorprecords/launchboard`
  and its smoke test asserts the server name `launchboard`.

## Verification (run 2026-10-28, builder session)

Toolchain note: because the write guard blocks npm in this session
(see Lockfile), `tsc` and the test suite were run against the
completed sibling build's installed dependency tree
(`builds/launchledger/node_modules`, same fleet dependency set) via
a temporary `node_modules` symlink inside the build dir, removed
afterwards; the build dir was then restored to its clean state.
The code and tests exercised are launchboard's own, byte for byte.

1. `npx tsc --noEmit` → exit 0.
2. `npm run build` → **blocked in this session**: `tsc` compiled
   cleanly but every `dist/` write EPERM'd (TS5033), leaving 0-byte
   files (removed). Rerun per Lockfile step 4.
3. `node dist/cli.js --version` → not runnable without `dist/`; the
   identical code path via the CLI test suite prints `0.1.0` and
   exits 0 (see test results).
4. `npm test` → **70 tests, 70 pass, 0 fail** (registration-integrity
   suite deep-equals the served tools/list against the fixture's 4
   tools; the omission suite asserts `submit_tool` and
   `create_checkout` absent at registry, MCP-layer — tools/list and
   tools/call — and CLI level; plus session handling/reuse, env
   override, the no-Authorization-header assertions under every
   environment, verbatim argument forwarding and result pass-through
   including upstream isError, SSE parsing, error rendering,
   MCP-layer validation, and CLI behaviour).
5. Parity: the harness's `build_report` was invoked directly
   (the harness's own `parity.json` write EPERMs in this session)
   against the source-run server
   (`stdio:node --import tsx /home/hatch/workspace/mcp-rebuild-100/builds/launchboard/src/cli.ts mcp`)
   → **verdict PASS, parity 100.0** (4/4 identical, 0 missing,
   0 extra, no schema diffs, no errors). The prescribed dist-based
   run — `python3 /home/hatch/workspace/mcp-rebuild-100/harness/parity.py --original /home/hatch/workspace/mcp-rebuild-100/harness/fixtures/lanzamientosia-tools-original.json --rebuild "stdio:node /home/hatch/workspace/mcp-rebuild-100/builds/launchboard/dist/cli.js mcp" --out /home/hatch/workspace/mcp-rebuild-100/builds/launchboard/parity-out`
   — remains to be run after Lockfile step 4 produces `dist/`;
   `parity-out/` is intentionally absent until then. The MCP
   surface served from `dist/` is the compiled form of the exact
   source verified here, so no parity delta is expected.

## Coordinator addendum (2026-10-28, Lane B Batch B6 close)

The lockfile rerun prescribed above was performed by the
coordinator in a fresh foreground session, in the prescribed
order, after the write-guard window lifted:

1. `npm install --package-lock-only --ignore-scripts --offline`
   (Route 3, clean dir) → **success**. Parses as JSON;
   `name` == `@reneturcios/launchboard`; **128** package
   entries; sha256
   `f16703dc366229ea9d73961c442808dc4f35b92624e925d6205ca4a57cb699e3`.
2. `npm ci --ignore-scripts --offline` → success; lockfile
   sha256 re-verified **unchanged** afterwards.
3. `tsc --noEmit` → exit 0; `tsc` build → exit 0;
   `node dist/cli.js --version` → `0.1.0`.
4. Full suite (tsx --test, coordinator re-run) → **70 tests,
   70 pass, 0 fail**.
5. Parity (harness, tools-file mode vs
   `harness/fixtures/lanzamientosia-tools-original.json`) →
   **verdict PASS, parity 100.0** (4/4 identical, 0 missing,
   0 extra). Output in `parity-out/`.

The build therefore meets every counting gate. The
builder-session block and its diagnosis above are retained
as the provenance record for why the lockfile landed after
the rest of the build.
