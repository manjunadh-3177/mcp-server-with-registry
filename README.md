# MCP Server with Registry, Policy, and Audit

A stateless [Model Context Protocol](https://modelcontextprotocol.io) server implementing protocol revision `2026-07-28` from scratch — no `@modelcontextprotocol/sdk` — with registry publication metadata, scope-based authorization, action-bound approval for destructive tools, and redacted audit logging.

Two independent implementations of the same contract ship side by side:

| | Language | What it proves |
|---|---|---|
| [`code/ts/`](code/ts/) | TypeScript, hand-rolled JSON-RPC 2.0 | A **real stdio transport** you can point an MCP client at today — reads newline-delimited JSON-RPC from stdin, writes responses to stdout, exactly how Claude Desktop or Claude Code talks to a local tool server |
| [`code/main.py`](code/main.py) | Python, stdlib only | The **registry and policy model** — `server.json` validation, reverse-DNS namespace ownership, issuer/audience/scope checks, and action-bound approval records that reject a replayed call if even one argument changed |

Both are fully tested (20/20 TS tests, 21/21 Python tests) and both run with zero external dependencies beyond `npm install`.

## Why this exists

Most "MCP server" tutorials wire three tools to the official SDK and call it done. That skips the actual hard part: MCP `2026-07-28` is **stateless** — no `initialize` handshake, no session ID, no connection-scoped state — so every request has to carry its own protocol version and client capabilities, and a load balancer can legally route two consecutive requests from the same client to two different server replicas. On top of that, a destructive tool call can't be authorized by a scope alone; it needs an approval record bound to the exact actor, tool, and argument digest, or a replay with one changed field has to be rejected before the handler ever runs.

This project implements that whole contract, not just the tool-calling happy path.

## What's real vs. what's out of scope

Being direct about this because it matters for anyone reviewing the code:

**Actually implemented and tested:**
- Stateless request envelope (protocol version + client capabilities on every call, no session state)
- `server/discover` (live capability negotiation) and deterministic, cache-aware `tools/list`
- `tools/call` with JSON Schema argument validation — malformed arguments return a tool error without invoking the handler
- Scope-based authorization: issuer, audience, and expiry checked on every call
- Action-bound approval records for destructive tools — the approval is a hash of the actor, tool name, normalized arguments, and target; changing one argument invalidates it
- Redacted audit logging (emails and SSNs are scrubbed before anything is logged)
- `server.json` registry document validation, including reverse-DNS publisher namespace checks and drift detection between published metadata and live `server/discover` output
- A real stdio transport (`npm run serve`) speaking newline-delimited JSON-RPC 2.0

**Explicitly out of scope (documented in [`docs/en.md`](docs/en.md), not silently dropped):**
- Streamable HTTP transport (network listener, `Origin` validation, SSE) — the stdio transport is real; the HTTP variant described in the lesson spec is not built here
- Real OAuth token issuance/validation — the `Token` type is `is_expired`/`has_scope`, not a JWT verifier against a live authorization server
- Publishing to the actual MCP Registry, or running two horizontally-scaled replicas behind a load balancer

If you're evaluating this for a role: the protocol/policy/audit logic is genuinely correct and tested, not a simulation. The network/infra layer around it is intentionally not built, because that's cloud infrastructure a solo project can describe honestly without needing to stand up.

## Run it

**TypeScript (the real transport):**
```bash
cd code/ts
npm install
npm test              # 20 tests
npm run demo          # scripted fixture walkthrough, self-terminating
npm run serve         # real stdio server — stays alive on stdin
```

Try the live server directly:
```bash
echo '{"jsonrpc":"2.0","id":1,"method":"server/discover","params":{"_meta":{"io.modelcontextprotocol/protocolVersion":"2026-07-28","io.modelcontextprotocol/clientCapabilities":{},"io.modelcontextprotocol/clientInfo":{"name":"test-client","version":"1.0.0"}}}}' | npm run serve
```

**Python (registry + policy model):**
```bash
python3 code/main.py                       # prints registry docs, policy-gated calls, audit log
python3 -m pytest code/tests/test_main.py  # 21 tests
```

### Connecting to a real MCP client

The TS server's `npm run serve` speaks standard MCP stdio, so it can be added to Claude Desktop's or Claude Code's MCP config as a local server (command: `npx`, args: `["tsx", "<path>/code/ts/src/index.ts", "--serve"]`, working directory set to `code/ts`). Once connected, `incidents_list`, `incidents_get`, and `incidents_ack` show up as real callable tools.

## Architecture

```
Registry (server.json)  --  publication metadata: name, version, remote transport
        |
        v
MCP client  --  server/discover  -->  live protocol version + capabilities
        |
        v
   tools/list  -->  deterministic, cache-aware tool catalog (ttlMs, cacheScope)
        |
        v
   tools/call  -->  policy_decide()
                       |-- issuer/audience/expiry check
                       |-- scope check
                       |-- [destructive tools only] approval-record check
                       v
                  handler executes  -->  redact()  -->  audit log
```

## Key design decisions worth discussing in an interview

- **Stateless-by-construction, not stateless-by-convention.** Every request revalidates protocol version and capabilities from `params._meta` rather than trusting a prior handshake — this is what makes the server safely horizontally scalable without sticky sessions.
- **Approval records are content-addressed, not capability-addressed.** An approval is a SHA-256 digest of `(actor, tool, canonicalized arguments, target, expiry)`. Changing any single argument produces a different digest and the approval no longer authorizes the call — this closes the "approved the plan, agent executed something slightly different" gap that a simple boolean approval flag would miss.
- **Registry metadata and runtime discovery are validated as two independent sources of truth.** `validate_runtime_alignment()` explicitly checks that what's published in `server.json` still matches what the live process reports via `server/discover`, catching configuration drift that a single source of truth would hide.



## Implementation Notes

This project implements the Model Context Protocol (protocol revision 2026-07-28)
based on a high-level lesson specification describing the required architecture:
registry publication metadata, scope-based authorization, action-bound approval
for destructive tools, and audit logging. The specification described *what*
to build, not working code — the stateless request handling, protocol/policy/
audit engine, both language implementations (TypeScript and Python), and the
full test suite (41 tests) were designed and implemented independently.

See [`docs/en.md`](docs/en.md) for the original lesson specification this
implementation is based on.

## License

MIT — see the [original repository](https://github.com/rohitg00/ai-engineering-from-scratch/blob/main/LICENSE) for full license text.
