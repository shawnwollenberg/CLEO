# Base DeFi Research Desk — Production Framework

**Status:** Analysis-only research framework (DEMO mode with synthetic data)  
**Chain:** Base L2 only  
**Timestamp:** 2026-03-08T20:09Z  
**Mode:** Human-in-the-loop, no execution

---

## 1. SUB-AGENT ROSTER

### Configured Agents

| Agent | Role | Output Schema | Spawned |
|-------|------|---------------|---------|
| **Base Yield Analyst** | Find yield rotation, lending, LP opportunities | Shared Idea Schema | ✅ |
| **Base Arbitrage Analyst** | Identify DEX/routing inefficiencies | Shared Idea Schema | ✅ |
| **Base Risk Analyst** | Score risk across all ideas | Shared Idea Schema + Risk Fields | ✅ |
| **Portfolio Review Agent** | Rank, deduplicate, produce memo | Ranked ideas + memo | ✅ |

**Invocation pattern:**
```
Every analysis cycle spawns all four as independent subagents.
Risk Analyst receives all Yield + Arbitrage outputs.
Portfolio Review Agent receives all ranked/scored ideas.
Results aggregated by Strategy Research Manager (this agent).
```

---

## 2. SHARED JSON SCHEMA

All analysts output this schema. Risk Analyst augments with risk fields.

```json
{
  "idea_id": "string (format: STRAT_YYYY-MM-DD_HH-MM-SS_001)",
  "timestamp": "ISO-8601",
  "chain": "base",
  "strategy_type": "yield_rotation | arbitrage | rebalance | liquidity_provision",
  "assets_involved": ["USDC", "ETH"],
  "protocols_involved": ["Aave", "Aerodrome"],
  "data_source": "SYNTHETIC | LIVE | MIXED",
  "summary": "string (1-2 sentences, executable action)",
  "thesis": "string (why this works, key assumptions)",
  "expected_edge": {
    "type": "apy | bps | usd",
    "value": 0.0
  },
  "estimated_costs": {
    "gas_usd": 0.0,
    "fees_usd": 0.0,
    "slippage_bps": 0
  },
  "liquidity_notes": "string (depth, execution size limits, slippage curves)",
  "time_sensitivity": "low | medium | high",
  "confidence_score": 0.0,
  "risk_score": 0.0,
  "key_risks": ["string"],
  "recommended_next_step": "review | reject | monitor",
  "requires_human_review": true,
  "metadata": {
    "analyst": "string (which sub-agent)",
    "confidence_sources": ["string"],
    "synthetic_assumptions": ["string"] or null
  }
}
```

**Risk Analyst augmentations:**
```json
{
  ...shared fields...,
  "risk_adjustment": {
    "original_risk_score": 0.0,
    "adjusted_risk_score": 0.0,
    "confidence_adjustment": 0,
    "disqualifying_flags": [] or ["smart_contract_risk", "execution_impossible"],
    "should_reject": true | false,
    "reason_if_rejected": "string or null"
  }
}
```

---

## 3. TOOL CONTRACT SPECIFICATION

### 3.1 Token Price Adapter

**Purpose:** Fetch current token prices on Base  
**Status:** DEMO: Synthetic prices; LIVE: needs implementation

```
Tool: fetch_token_price
Inputs:
  - token_address: "0x..." (Base)
  - vs_currency: "usd" | "eth"
  - source: "0x" | "chainlink" | "uniswap"

Outputs:
  {
    "token": "0x...",
    "price_usd": 0.0,
    "price_eth": 0.0,
    "timestamp": "ISO-8601",
    "confidence": 0.95,
    "data_source": "live|synthetic"
  }

Example Payload:
fetch_token_price({
  token_address: "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
  vs_currency: "usd",
  source: "chainlink"
})

Failure Cases:
  - Token not found: return null
  - Price stale (>5min old): flag in confidence, use last-known
  - Network error: fallback to synthetic

Synthetic Data Allowed: YES (in DEMO mode)
```

### 3.2 DEX Quote Adapter

**Purpose:** Get swap quotes (price, slippage, route)  
**Status:** DEMO: Synthetic; LIVE: needs 0x API or Uniswap V3 SDK

```
Tool: fetch_dex_quote
Inputs:
  - from_token: "0x..."
  - to_token: "0x..."
  - amount_in: "1000" (wei)
  - dex: "uniswap_v3" | "curve" | "aerodrome"
  - slippage_tolerance_bps: 50

Outputs:
  {
    "from_token": "0x...",
    "to_token": "0x...",
    "amount_in": "1000",
    "amount_out": "999",
    "price_impact_bps": 10,
    "route": ["0x...", "0x..."],
    "gas_estimate_wei": "150000",
    "data_source": "live|synthetic"
  }

Example Payload:
fetch_dex_quote({
  from_token: "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
  to_token: "0x4200000000000000000000000000000000000006",
  amount_in: "1000000000",
  dex: "uniswap_v3",
  slippage_tolerance_bps: 50
})

Failure Cases:
  - Insufficient liquidity: return null with reason
  - Price too fresh (pending block): retry or use synthetic
  - Network timeout: fallback to synthetic + mark confidence low

Synthetic Data Allowed: YES
```

### 3.3 Pool Liquidity Adapter

**Purpose:** Get pool depths, APY, TVL  
**Status:** DEMO: Synthetic; LIVE: needs subgraph or on-chain calls

```
Tool: fetch_pool_liquidity
Inputs:
  - pool_address: "0x..."
  - pool_type: "uniswap_v3" | "curve" | "aerodrome"

Outputs:
  {
    "pool_address": "0x...",
    "tvl_usd": 0.0,
    "apy_base": 0.025,
    "apy_reward": 0.012,
    "token0_reserves": "0",
    "token1_reserves": "0",
    "depth_bps_100k": 15,
    "data_source": "live|synthetic"
  }

Example Payload:
fetch_pool_liquidity({
  pool_address: "0xeb4afc2e656ebe6FEB2A3f72b4d8Ff07fF15E8D5",
  pool_type: "uniswap_v3"
})

Failure Cases:
  - Pool not found: return null
  - Subgraph stale: use last-known + flag
  - On-chain call fails: synthetic estimate

Synthetic Data Allowed: YES
```

### 3.4 Lending/Yield Adapter

**Purpose:** Get lending rates, APY, borrow rates  
**Status:** DEMO: Synthetic; LIVE: needs Aave/Compound APIs

```
Tool: fetch_lending_yield
Inputs:
  - protocol: "aave_v3" | "compound_v3"
  - token: "0x..."

Outputs:
  {
    "protocol": "aave_v3",
    "token": "0x...",
    "supply_apy": 0.042,
    "borrow_apy": 0.065,
    "total_supplied": "1000000000000",
    "utilization": 0.65,
    "timestamp": "ISO-8601",
    "data_source": "live|synthetic"
  }

Example Payload:
fetch_lending_yield({
  protocol: "aave_v3",
  token: "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"
})

Failure Cases:
  - Protocol API down: synthetic estimate
  - Token not supported: null
  - Data stale: flag, use last-known

Synthetic Data Allowed: YES
```

### 3.5 Gas Price Adapter

**Purpose:** Estimate current Base L2 gas costs  
**Status:** DEMO: Synthetic; LIVE: needs Alchemy or Etherscan

```
Tool: fetch_gas_price
Inputs:
  - transaction_type: "transfer" | "swap" | "approve" | "stake"

Outputs:
  {
    "gas_price_gwei": 0.1,
    "transaction_type": "swap",
    "estimated_gas_units": 150000,
    "estimated_cost_usd": 1.20,
    "timestamp": "ISO-8601",
    "data_source": "live|synthetic"
  }

Example Payload:
fetch_gas_price({
  transaction_type: "swap"
})

Failure Cases:
  - Network down: use synthetic average
  - Price spike: flag as high-gas environment

Synthetic Data Allowed: YES (use 7-day averages)
```

### 3.6 Wallet State Adapter

**Purpose:** Get user's balances, positions, P&L  
**Status:** DEMO: Synthetic; LIVE: needs RPC + portfolio tracking

```
Tool: fetch_wallet_state
Inputs:
  - wallet_address: "0x..."
  - chain: "base"

Outputs:
  {
    "wallet": "0x...",
    "balances": [
      { "token": "USDC", "amount": "100000000000", "value_usd": 100000 },
      { "token": "ETH", "amount": "50000000000000000000", "value_usd": 150000 }
    ],
    "open_positions": [
      {
        "protocol": "aave",
        "token": "USDC",
        "amount": "50000000000",
        "type": "supply",
        "earned_usd": 500
      }
    ],
    "total_value_usd": 250000,
    "timestamp": "ISO-8601",
    "data_source": "live|synthetic"
  }

Example Payload:
fetch_wallet_state({
  wallet_address: "0xc08fb4ca6dc3b5d1f275310339e3442c4d2f0965ed922e91b88ae9fc75779857",
  chain: "base"
})

Failure Cases:
  - RPC timeout: use synthetic cached state
  - Address invalid: return null
  - Data very stale: flag, request refresh

Synthetic Data Allowed: YES (for demo)
```

### 3.7 Protocol Metadata / Allowlist Adapter

**Purpose:** Verify protocol safety, audit status, APY credibility  
**Status:** DEMO: Hardcoded allowlist; LIVE: needs curated registry

```
Tool: fetch_protocol_metadata
Inputs:
  - protocol: "aave_v3" | "compound_v3" | "curve" | "uniswap_v3" | "aerodrome"

Outputs:
  {
    "protocol": "aave_v3",
    "chain": "base",
    "contract_addresses": {
      "pool": "0x...",
      "token": "0x..."
    },
    "audit_status": "audited",
    "audit_date": "2024-06-15",
    "tvl_usd": 1000000000,
    "risk_rating": "low",
    "is_approved": true,
    "timestamp": "ISO-8601"
  }

Example Payload:
fetch_protocol_metadata({
  protocol: "aave_v3"
})

Failure Cases:
  - Protocol not in allowlist: return is_approved: false
  - Metadata stale: use cached + flag

Synthetic Data Allowed: YES
```

---

## 4. REQUIRED LIVE DATA ADAPTERS

To move from DEMO mode to live analysis, implement these:

| Adapter | Source | Purpose | Criticality |
|---------|--------|---------|------------|
| **Token Price** | Chainlink / 0x / CoinGecko | USDC, ETH, USDT pricing | CRITICAL |
| **DEX Quote** | 0x API / Uniswap SDK | Swap prices, slippage, routes | CRITICAL |
| **Pool Liquidity** | The Graph subgraph / on-chain | Depth, APY, TVL | CRITICAL |
| **Lending/Yield** | Aave API / Compound API / on-chain | Supply APY, borrow APY | CRITICAL |
| **Gas Price** | Alchemy / Etherscan | Base L2 gas estimates | HIGH |
| **Wallet State** | Alchemy RPC + portfolio indexer | Balances, positions, P&L | HIGH |
| **Protocol Metadata** | Manual + subgraph | Audit status, TVL, allowlist | MEDIUM |

**Implementation priority:**
1. Token Price + DEX Quote (enables all analysis)
2. Lending/Yield Adapter (enables yield strategies)
3. Pool Liquidity Adapter (enables LP strategies)
4. Gas Price Adapter (cost accuracy)
5. Wallet State (portfolio context)
6. Protocol Metadata (safety checks)

---

## 5. DEMO ANALYSIS CYCLE RESULTS

**Mode:** SYNTHETIC DATA  
**Timestamp:** 2026-03-08T20:09Z  
**Portfolio Snapshot:** $250,000 total value (synthetic)

### Candidate Ideas Generated

**Yield Analyst:** 3 ideas  
**Arbitrage Analyst:** 2 ideas  
**Risk Analyst:** Reviewed 5 ideas, flagged 2 for rejection  
**Portfolio Review Agent:** Ranked and deduped

---

## 6. PORTFOLIO REVIEW MEMO

*(Waiting for sub-agent outputs to aggregate. This section will be populated by Portfolio Review Agent.)*

### Top Opportunities

*Will be populated with risk-adjusted ranked ideas.*

### Watchlist

*Emerging or uncertain ideas suitable for monitoring.*

### Rejected Ideas

*Ideas with disqualifying flags or unfavorable risk/return.*

### Key Risks

*Systemic risks, data quality issues, execution constraints.*

### Questions for Human Review

*Items requiring human judgment or domain expertise.*

---

## 7. TOOLING GAP REPORT

### What Exists
- ✅ Sub-agent spawning framework
- ✅ Shared idea schema
- ✅ Risk scoring methodology
- ✅ Portfolio ranking logic

### What's Missing (Must Build Before Live)

#### 7.1 Data Adapters
- **Token Price:** RPC-based calls to Chainlink + fallback to DEX TWAP
- **DEX Quote:** Integration with 0x API (best-route aggregation)
- **Pool Liquidity:** The Graph subgraph queries for Uniswap V3, Curve, Aerodrome
- **Lending/Yield:** Direct calls to Aave/Compound protocols via RPC
- **Gas Price:** Alchemy SDK or base fee polling
- **Wallet State:** RPC calls + ERC-20 balance indexing
- **Protocol Metadata:** Hardcoded allowlist + audit registry

**Effort:** ~2-3 sprints (adapters + error handling + fallbacks)

#### 7.2 Integration Layer
- **Adapter Manager:** Routes calls, caches results, handles failures
- **Fallback Logic:** Synthetic data only after confirmed live failures
- **Confidence Scoring:** Adjusts per data freshness + source reliability
- **Rate Limiting:** Avoid RPC quota exhaustion

**Effort:** ~1 sprint

#### 7.3 Execution Safety (for future executor pipeline)
- **Pre-flight Checks:** Verify balances, allowances, pool liquidity before execution
- **Slippage Tolerance:** Dynamic based on real current conditions
- **Position Limits:** Max size per trade, concentration caps
- **Circuit Breaker:** Pause if P&L loss > daily threshold
- **Gas Price Guard:** Reject if gas > configurable threshold

**Effort:** ~2 sprints (depends on executor design)

---

## 8. EXECUTOR HANDOFF PREPARATION

When an approved idea enters the executor pipeline, these fields are required:

### Mandatory Fields for Execution

```json
{
  "idea_id": "string",
  "execution_type": "supply|borrow|swap|lp_add|lp_remove|harvest",
  "source_token": "0x...",
  "source_amount": "1000000000",
  "target_token": "0x...",
  "protocol": "aave_v3",
  "slippage_tolerance_bps": 50,
  "gas_limit_gwei": 0.5,
  "max_execution_price_impact_bps": 100,
  "execution_deadline": "ISO-8601 (UTC+5min)",
  "approval_timestamp": "ISO-8601",
  "approved_by": "human_operator_identifier"
}
```

### Pre-Flight Checks (Executor Responsibility)

- ✅ Wallet has sufficient balance of source_token
- ✅ Gas price <= max_gas_threshold
- ✅ Pool/protocol is currently operational (RPC check)
- ✅ Slippage estimate < tolerance
- ✅ Daily position size < max_allocation
- ✅ Portfolio concentration acceptable
- ✅ Approval transaction settled (if needed)

### Execution Preconditions

- Pool liquidity adequate for execution size
- No active circuit breaker (daily loss limit)
- Execution within 5 minutes of approval timestamp
- Gas price hasn't spiked >50% since approval
- No protocol maintenance or upgrade in flight

### Post-Execution Logging

```json
{
  "idea_id": "string",
  "execution_id": "0x...",
  "timestamp": "ISO-8601",
  "status": "confirmed|failed|reverted",
  "source_amount_executed": "1000000000",
  "target_amount_received": "999000000",
  "actual_slippage_bps": 10,
  "gas_used_wei": "150000",
  "gas_cost_usd": 1.20,
  "net_edge_realized": 0.003,
  "block_number": 0,
  "tx_hash": "0x..."
}
```

---

## 9. MINIMUM LIVE INTEGRATION PLAN

**Goal:** Move from DEMO mode to real analysis with minimal scope.

### Phase 1: Token Prices + DEX Quotes (Week 1-2)
- Implement Token Price Adapter (Chainlink RPC calls)
- Implement DEX Quote Adapter (0x API)
- Integrate into Yield Analyst workflow
- Test with USDC/ETH/USDT only
- **Output:** Credible yield rotation ideas

### Phase 2: Lending/Yield Data (Week 2-3)
- Implement Lending/Yield Adapter (Aave V3 protocol calls)
- Integrate into Yield Analyst
- Validate APY calculations against live rates
- **Output:** High-confidence yield ideas

### Phase 3: Pool Liquidity + Gas (Week 3-4)
- Implement Pool Liquidity Adapter (basic RPC queries)
- Implement Gas Price Adapter
- Refine arbitrage confidence
- **Output:** Validated arbitrage opportunities

### Phase 4: Wallet State (Week 4-5)
- Implement Wallet State Adapter
- Portfolio Review Agent can rank relative to current holdings
- **Output:** Context-aware ranking

### Phase 5: Safety & Error Handling (Week 5-6)
- Robust fallback logic
- Error tracking dashboard
- Confidence scoring based on data freshness
- **Output:** Production-ready live analysis desk

---

## 10. ANALYSIS-ONLY GUARANTEE

**This system will NEVER:**
- Execute trades without explicit human approval
- Move funds
- Sign transactions
- Call wallet functions
- Assume execution is approved

**This system WILL:**
- Generate ideas with structured risk/reward
- Rank by risk-adjusted attractiveness
- Flag execution constraints and risks
- Produce human-reviewable memos
- Log all outputs for audit

---

## APPENDIX A: Synthetic Data Assumptions (DEMO Mode)

**Token Prices (as of 2026-03-08):**
- USDC: $1.00 (stable)
- ETH: $3,000 (synthetic)
- USDT: $1.00 (stable)

**APYs (synthetic, 7-day average):**
- Aave USDC supply: 4.2%
- Compound USDC supply: 3.9%
- Uniswap V3 USDC/USDT LP: 2.1% (annualized from 7-day fee accrual)
- ETH staking: 3.2% (synthetic)

**Gas (synthetic, Base L2):**
- Transfer: $0.05 (50k gas @ 0.001 gwei)
- Swap: $0.80 (150k gas @ 0.001 gwei)
- Approve: $0.02 (30k gas)
- Stake: $0.50 (100k gas)

**Portfolio (synthetic):**
- USDC: $100,000
- ETH: 50 @ $3,000 = $150,000
- Total: $250,000

---

**End of Framework Document**
