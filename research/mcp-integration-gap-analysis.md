# MCP Integration Gap Analysis (for State / Shock / Amplifier)

Date: 2026-02-26  
Workspace: `/Users/qinghuan/Documents/code/financial-services-plugins`

## 1. Scope and Method

This audit evaluates whether the currently integrated MCP servers in this repository can support your target capability:

- State engine (macro/cross-asset regime)
- Shock engine (24-72h event and news shock window)
- Amplifier engine (crypto leverage/liquidity tail-risk)

Evidence used:

- Static config and docs:
  - `.mcp.json` files
  - `README.md`
  - `partner-built/lseg/CONNECTORS.md` and related skills/commands
- Runtime probes:
  - HTTP `POST initialize` checks against configured endpoints (no credentials, health only)
  - `list_mcp_resources` in current Codex runtime

## 2. Current MCP Inventory

From `financial-analysis/.mcp.json` (core shared connectors):

- daloopa
- morningstar
- sp-global
- factset
- moodys
- mtnewswire
- aiera
- lseg
- pitchbook
- chronograph
- egnyte

Partner plugin MCPs:

- `partner-built/lseg/.mcp.json`: lseg
- `partner-built/spglobal/.mcp.json`: spglobal

Notable architecture point:

- Add-on plugins mostly have no own MCP servers; design assumes core `financial-analysis` is installed first.

## 3. Endpoint Health (No Credentials)

Probe summary (HTTP POST initialize):

| Endpoint | Status | Interpretation |
|---|---:|---|
| Daloopa | 401 | Reachable, auth required |
| Morningstar | 401 | Reachable, auth required |
| S&P Global | 401 | Reachable, auth required |
| FactSet | 303 | Reachable, redirects to OAuth |
| Moody's | 401 | Reachable, auth required |
| Aiera | 401 | Reachable, OAuth-protected |
| LSEG (`/lfa/mcp`) | 401 | Reachable, bearer token required |
| LSEG partner (`/server-cl`) | 401 | Reachable, bearer token required |
| PitchBook | 401 | Reachable, auth required |
| Chronograph | 401 | Reachable, auth required |
| Egnyte | 401 | Reachable, auth required |
| MT Newswires (`/mtnewswires`) | 404 | Likely wrong path in config |

Important finding:

- `https://vast-mcp.blueskyapi.com/mtnewswires` returns 404.
- `https://vast-mcp.blueskyapi.com/mcp` returns 401 (valid protected endpoint behavior).
- This strongly suggests `mtnewswire` URL is stale/misconfigured and should likely be `/mcp`.

Runtime availability finding:

- `list_mcp_resources` and `list_mcp_resource_templates` return empty in this Codex session.
- So in this runtime, connectors are not actively mounted/authenticated even if repo config exists.

## 4. Capability Coverage vs Your Target System

### 4.1 State (macro regime) coverage

Needed signals:

- Growth/inflation/labor trends
- Rates curve shape, real rates, swap spreads
- Cross-asset risk appetite proxies (equities/FX/rates volatility)

Current MCP support:

- **Partially covered to strong** via LSEG:
  - `qa_macroeconomic`
  - `interest_rate_curve`
  - `inflation_curve`
  - `ir_swap`
  - `qa_historical_equity_price`
  - vol surfaces for equity/FX

Assessment:

- **State engine is feasible** with current integration, pending credentials and production query templates.

### 4.2 Shock (24-72h) coverage

Needed signals:

- Scheduled event window (FOMC, CPI, NFP, etc.)
- Real-time unscheduled shock headlines
- Event classification into macro / cross-asset / crypto-native

Current MCP support:

- **Partial**:
  - News providers present (MT Newswires, Aiera) in config
  - Macro timeseries available (LSEG)
- **Key limitation**:
  - No explicit macro-event-calendar MCP observed in repo
  - MT Newswires endpoint appears misconfigured (404 as configured)

Assessment:

- **Shock engine is partially feasible**, but currently constrained by (a) missing explicit calendar feed and (b) MT Newswires path issue.

### 4.3 Amplifier (crypto microstructure) coverage

Needed signals:

- Perp funding, OI, liquidations
- Basis term structure (spot vs perp/futures)
- Options IV/skew for BTC
- Order book depth / liquidity stress proxies
- Stablecoin/exchange-flow/on-chain stress proxies (optional but high value)

Current MCP support:

- **Largely not covered** in current integrated connectors/tool docs.
- Existing LSEG option/vol tools are focused on FX/equity contexts in documented workflows.
- No dedicated crypto-derivatives or on-chain MCP in this repository.

Assessment:

- **Amplifier engine is not sufficiently supported** by current MCP stack.

## 5. Does Current MCP Integration Meet Your Requirement?

Short answer:

- **Not fully**.

Detailed answer:

- Can support:
  - Risk baseline (from your local BTC dataset, no MCP needed)
  - State engine (macro/cross-asset) with LSEG and related feeds
  - Part of Shock engine (news+macro context)
- Cannot robustly support:
  - Amplifier engine (crypto leverage/liquidity tail-risk)
  - Fully reliable shock pipeline while MT Newswires URL remains unresolved and no explicit macro calendar feed is integrated

## 6. Concrete Missing Pieces (Priority Ordered)

P0 (blockers for production-grade target):

1. **Crypto derivatives/microstructure MCP**
   - Required fields: funding, OI, liquidations, perp basis, options skew/term, depth
2. **MT Newswires endpoint correction**
   - Current configured URL appears invalid (`/mtnewswires` -> 404)
3. **Auth/bootstrap runbook**
   - Token setup, renewal behavior, and health checks per provider

P1 (quality and false-positive reduction):

4. **Scheduled macro calendar MCP**
   - Event time, consensus, prior, actual, surprise
5. **Cross-asset volatility/credit stress standardized schema**
   - Unified normalized fields to feed State/Shock models

P2 (operational robustness):

6. **Connector contract tests**
   - Per-MCP smoke test for `initialize` and one read tool call
7. **Data quality monitoring**
   - Missing-field rate, stale timestamps, and fallback source logic

## 7. Minimal Path to “Meets Requirement”

1. Fix MT Newswires URL in core `.mcp.json` and verify via protocol probe.
2. Add one crypto-focused MCP with derivatives + liquidity metrics.
3. Add one macro calendar MCP (or equivalent event feed).
4. Keep current LSEG stack as State backbone.
5. Build a unified output schema for `state/shock/amplifier` so strategy layer consumes stable fields.

## 8. Final Verdict

Current integrated MCP stack is strong for traditional financial analytics and partially usable for your Shock concept, but it is **insufficient for a complete State→Shock→Amplifier risk thermostat** until crypto microstructure data and event-window plumbing are added.
