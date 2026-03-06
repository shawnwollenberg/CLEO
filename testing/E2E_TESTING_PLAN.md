# AgentGuardrail End-to-End Testing Plan

## Overview

AgentGuardrail is an on-chain permission and policy enforcement layer for AI agents, built on ERC-8004 standards. Every agent gets an ERC-4337 smart account where all actions are validated against policies before execution.

This plan covers comprehensive testing from unit level through full e2e scenarios, including on-chain validation.

---

## System Architecture Summary

- **Backend:** Go service with Chi router, PostgreSQL
- **Smart Accounts:** ERC-4337 with PermissionEnforcer validation
- **Policies:** JSON definitions with actions, assets, constraints, conditions
- **Validation:** Off-chain (pre-flight) + on-chain enforcement (protocol level)
- **Audit:** Dual-source (off-chain requests + on-chain events via indexer)
- **Smart Account Options:** Connected wallet or generated bot signer

---

## Testing Layers

### 1. **Unit Testing** (Policy Engine Core)
Test individual validation logic in isolation.

#### Policy Definition Validation
- [x] Valid policy (actions, assets, constraints, duration, conditions)
- [x] Nil definition rejected
- [x] Empty actions rejected
- [x] Invalid action type rejected (e.g., `invalid_action`)
- [x] Wildcard actions `*` accepted
- [x] Invalid numeric constraints rejected (e.g., `maxValuePerTx: "not_a_number"`)
- [x] Invalid duration (validUntil before validFrom) rejected
- [x] Invalid condition operators rejected
- [ ] Action type case-insensitivity (e.g., `SWAP` vs `swap`)
- [ ] Token/protocol/chain wildcards in asset filters
- [ ] Complex nested conditions

#### Action Matching
- [x] Action type exact match
- [x] Action type wildcard match
- [x] Token filter (whitelist, non-match)
- [x] Protocol filter (whitelist, non-match)
- [x] Chain filter (whitelist, non-match)
- [x] Max value per transaction constraint
- [ ] Max daily volume constraint (requires DB mock)
- [ ] Max weekly volume constraint
- [ ] Transaction count limits
- [ ] Multiple constraints together (AND logic)

#### Condition Evaluation
- [x] Equality operator (`eq`)
- [x] Inequality operator (`ne`)
- [x] Greater than (`gt`)
- [x] Greater than or equal (`gte`)
- [x] Less than (`lt`)
- [x] Less than or equal (`lte`)
- [x] In operator (`in` with array)
- [x] Not in operator (`not_in`)
- [x] Contains operator (substring match)
- [ ] Regex operator
- [ ] Custom data field conditions
- [ ] Condition combinations (multiple conditions must ALL pass)

#### Numeric Comparisons
- [x] Big integer comparisons
- [x] Type coercion (string, int64, float64 to BigInt)
- [x] Invalid type handling

---

### 2. **Integration Testing**
Test how components interact.

#### Database Integration
- [ ] Policy creation persisted to DB
- [ ] Permission grant creates active record
- [ ] Active policies loaded correctly for validation
- [ ] Expired policies ignored (validUntil check)
- [ ] Daily usage tracking across validation requests
- [ ] Concurrent validation requests don't race
- [ ] Transaction isolation (dirty reads prevented)

#### API Layer
- [ ] `/api/v1/validate` returns `enforcement_level: "enforced"`, `wallet_type: "smart_account"`, `onchain_enforced: true`
- [ ] `/api/v1/validate/batch` handles max 100 requests
- [ ] `/api/v1/validate/batch` rejects >100 requests
- [ ] `/api/v1/validate/simulate` returns recommendations
- [ ] Request ID generated and returned
- [ ] Latency captured (latency_ms)
- [ ] Unauthorized requests rejected (no JWT)

#### Audit Logging
- [ ] Off-chain validation logged to `validation_requests` table
- [ ] Audit log created for each validation event
- [ ] Event details include action, allowed/reason, permission_id, policy_id
- [ ] Sensitive data not exposed in logs

#### Permission & Policy Lifecycle
- [ ] Policy created with definition
- [ ] Permission granted for agent + policy
- [ ] Permission status transitions (active → revoked)
- [ ] Policy activation/revocation updates active permission queries
- [ ] Expired permission not returned by query
- [ ] Multiple policies for same agent (first match wins or all checked?)

#### Smart Account Interaction
- [ ] Agent registered on-chain creates account
- [ ] Signer type stored (wallet vs generated)
- [ ] Smart account address deterministic (CREATE2)
- [ ] Bot signer shown once, never stored

---

### 3. **End-to-End Scenarios**
Full flows from user action to enforcement result.

#### Scenario: Allowed Action
- [ ] User creates policy: `{actions: ["swap"], assets: {tokens: ["0xUSDC"]}, constraints: {maxValuePerTx: "1000"}}`
- [ ] User grants permission to agent with this policy
- [ ] Agent validates swap of 500 USDC
- [ ] Response: `allowed: true`, matching permission + policy IDs returned
- [ ] Off-chain audit log created
- [ ] (On-chain) Smart account execution proceeds
- [ ] On-chain enforcement event logged

#### Scenario: Blocked Action (Wrong Action Type)
- [ ] Policy allows `["swap"]` only
- [ ] Agent attempts `transfer` action
- [ ] Response: `allowed: false`, reason: "no matching policy found for this action"
- [ ] Audit log records block
- [ ] Smart account reverts on-chain

#### Scenario: Blocked Action (Token Not Whitelisted)
- [ ] Policy allows tokens `["0xUSDC"]` only
- [ ] Agent attempts swap of 0xDAI
- [ ] Response: `allowed: false`
- [ ] Reason indicates asset mismatch
- [ ] On-chain revert

#### Scenario: Blocked Action (Amount Exceeds Max)
- [ ] Policy: `maxValuePerTx: "1000"`
- [ ] Agent attempts swap of 1500
- [ ] Response: `allowed: false`, reason constraint violation
- [ ] Audit tracks violation type

#### Scenario: Conditional Approval
- [ ] Policy with condition: `{field: "slippage", operator: "lte", value: "0.5"}`
- [ ] Agent action includes `data: {slippage: "0.3"}`
- [ ] Response: `allowed: true` (condition satisfied)
- [ ] Agent action with `data: {slippage: "1.0"}`
- [ ] Response: `allowed: false` (condition failed)

#### Scenario: Daily Volume Limit
- [ ] Policy: `maxDailyVolume: "10000"`
- [ ] Morning: Agent swaps 6000 (allowed, daily total: 6000)
- [ ] Afternoon: Agent swaps 3000 (allowed, daily total: 9000)
- [ ] Evening: Agent swaps 2000 (blocked, would exceed 10000)
- [ ] Response: `allowed: false`, daily limit exceeded
- [ ] Next day: Limit resets, 2000 is allowed

#### Scenario: Policy with Expiration
- [ ] Policy: `validUntil: "2024-12-31T23:59:59Z"`
- [ ] Before expiry: Action allowed
- [ ] After expiry: Action blocked, reason "no matching policy"
- [ ] Audit shows policy not considered (expired)

#### Scenario: Wildcard Policy
- [ ] Policy: `{actions: ["*"], assets: {tokens: ["*"], chains: [*]}}`
- [ ] Any action on any token on any chain allowed (subject to constraints)
- [ ] Agent can swap, transfer, stake, etc.

#### Scenario: Multiple Permissions
- [ ] User has 2 active policies for same agent
  - Policy A: `swap` action only
  - Policy B: `transfer` with max 500
- [ ] Swap action: Policy A matches, allowed
- [ ] Transfer of 200: Policy B matches, allowed
- [ ] Transfer of 600: Both policies checked, B rejects (over limit), A doesn't allow transfer → blocked
- [ ] Behavior: First matching policy is used OR all must approve?

#### Scenario: Permission Revocation
- [ ] Active permission allows action
- [ ] User revokes permission (status → revoked)
- [ ] Next validation: Action blocked, permission not found
- [ ] Audit shows permission_id no longer returned

#### Scenario: Simulate (Pre-flight)
- [ ] User simulates action (no recording)
- [ ] Response: `would_allow`, `matching_policy`, `current_usage`, `remaining_quota`
- [ ] Current daily usage: 6000
- [ ] Limit: 10000
- [ ] Remaining quota: 4000
- [ ] Recommendations provided if no policy matches

---

### 4. **Smart Account & On-Chain Testing**

#### Account Deployment
- [ ] CREATE2 deterministic address for agent
- [ ] `AccountCreated` event emitted
- [ ] Account queryable via API
- [ ] Signer type stored (wallet/generated)
- [ ] Generated signer shown once, never returned again

#### On-Chain Enforcement
- [ ] `validateUserOp` calls PermissionEnforcer
- [ ] Enforcer checks policy validity
- [ ] Valid UserOp executes, emits `Executed`
- [ ] Invalid UserOp reverts at protocol level
- [ ] Revert reason includes constraint violation
- [ ] On-chain event (EnforcementResult) indexed within 12 seconds

#### Constraint Violations (On-Chain)
- [ ] `ConstraintViolation` event emitted with detail
- [ ] Block explorer shows revert reason
- [ ] Audit log captures violation with `tx_hash`, `block_number`

#### Event Indexing
- [ ] Indexer polls every 12 seconds
- [ ] `EnforcementResult` (allowed/blocked) events captured
- [ ] `ConstraintViolation` events logged
- [ ] `UsageRecorded` events tracked
- [ ] `Executed` events linked to smart account
- [ ] `AccountCreated` events tracked
- [ ] Indexer position tracked in DB
- [ ] Audit logs joined with on-chain data (tx_hash, block_number)

#### Dual-Source Audit
- [ ] Off-chain validation: `source='offchain'`, no tx_hash/block_number
- [ ] On-chain enforcement: `source='onchain'`, includes tx_hash, block_number
- [ ] Both visible in `/api/v1/audit` with optional source filter

---

### 5. **Security & Safety Testing**

#### Input Validation
- [ ] Null/undefined action properties handled gracefully
- [ ] Oversized policy definitions rejected
- [ ] Invalid JSON structure returns 400 error
- [ ] SQL injection in action data prevented (parameterized queries)
- [ ] Malformed UUIDs rejected
- [ ] Special characters in token addresses don't break parsing

#### Permission Model
- [ ] Users can only grant permissions to their own agents
- [ ] Users cannot revoke others' permissions
- [ ] JWT validation required for all protected endpoints
- [ ] Admin cannot bypass policy enforcement

#### Audit Log Integrity
- [ ] Audit logs immutable (append-only)
- [ ] Logs exportable (JSON/CSV)
- [ ] No audit log deletion allowed
- [ ] Sensitive data redacted (e.g., private keys never exposed)

#### Policy Tampering
- [ ] Policy definition immutable after creation
- [ ] Policy stored as JSON blob (checksummed?)
- [ ] Cannot modify policy, only create new version
- [ ] Permission points to specific policy revision

#### Rate Limiting
- [ ] Batch validation limited to 100 requests per call
- [ ] Request-level rate limiting (if configured)
- [ ] Limits applied per wallet_id
- [ ] Error response when limit exceeded

#### On-Chain Security
- [ ] Smart account cannot be drained by unauthorized transfer
- [ ] PermissionEnforcer contract prevents invalid operations
- [ ] Admin cannot add unauthorized actions to policy on-chain
- [ ] Signature validation on all on-chain calls

---

### 6. **Performance & Scalability Testing**

#### Throughput
- [ ] Single validation request: <10ms p95
- [ ] Batch validation (100 requests): <100ms p95
- [ ] Concurrent validation requests (100 parallel): no degradation
- [ ] Daily validation throughput baseline (target TBD)

#### Scaling with Data Size
- [ ] 10 policies: no latency increase
- [ ] 100 policies: <5ms latency
- [ ] 1000 policies: <10ms latency
- [ ] Policy with 100 conditions: still <10ms
- [ ] Audit log export with 1M+ entries: responsive (<5s)

#### Memory & Resource
- [ ] No memory leaks under sustained load (4h test)
- [ ] Policy engine memory usage stable
- [ ] DB connection pool doesn't exhaust (test with 100 concurrent)
- [ ] Log indexer doesn't fall behind (no event queue buildup)

#### Database Performance
- [ ] Permission query with date filters efficient (index on valid_from/until)
- [ ] Daily usage aggregate efficient (compound index on wallet + agent + created_at)
- [ ] Validation request insertion batched efficiently
- [ ] Audit log queries use appropriate indexes

---

### 7. **User-Facing & Operational Testing**

#### API Error Messages
- [ ] Unauthorized: "unauthorized" (no JWT)
- [ ] Invalid definition: "invalid action: X" (clear)
- [ ] Blocked action: "no matching policy found for this action"
- [ ] Batch too large: "max 100 requests per batch"
- [ ] Bad request: "invalid request body"
- [ ] Errors don't expose internal DB details

#### Admin Dashboard (if applicable)
- [ ] Policies list with status (active/revoked)
- [ ] Permissions per agent visible
- [ ] Audit log queryable by agent, action, date, result
- [ ] Real-time validation request metrics
- [ ] Policy effectiveness dashboard

#### Documentation
- [ ] Policy syntax documented with examples
- [ ] API endpoint examples (curl, SDK)
- [ ] Common mistakes (e.g., validUntil before validFrom)
- [ ] Troubleshooting: "Action blocked but shouldn't be" → check:
  - Is permission active?
  - Is policy active?
  - Does policy match action type, asset, chain?
  - Have constraints been exceeded?

#### Observability
- [ ] Structured logging (JSON) with request_id
- [ ] Validation latency metrics exported
- [ ] Policy match rate metrics
- [ ] On-chain enforcement rate metrics
- [ ] Alert: High block rate (>50% of validations blocked)
- [ ] Alert: Indexer lag (>5 minutes behind chain)

---

## Test Data & Fixtures

### Policy Fixtures

#### Permissive Policy (for testing happy path)
```json
{
  "actions": ["*"],
  "assets": {
    "tokens": ["*"],
    "chains": [1, 11155111]
  },
  "constraints": {
    "maxValuePerTx": "1000000000000000000000"
  }
}
```

#### Restrictive Policy (for testing blocks)
```json
{
  "actions": ["swap"],
  "assets": {
    "tokens": ["0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"],
    "chains": [1]
  },
  "constraints": {
    "maxValuePerTx": "5000000000000000000",
    "maxDailyVolume": "50000000000000000000"
  }
}
```

#### Conditional Policy
```json
{
  "actions": ["swap"],
  "constraints": {
    "maxValuePerTx": "10000000000000000000"
  },
  "conditions": [
    {
      "field": "slippage",
      "operator": "lte",
      "value": "0.5"
    }
  ]
}
```

#### Complex Policy
```json
{
  "actions": ["swap", "transfer", "lp_add"],
  "assets": {
    "tokens": ["0xUSDC", "0xETH", "0xDAI"],
    "protocols": ["uniswap-v3", "sushiswap"],
    "chains": [1, 137, 42161]
  },
  "constraints": {
    "maxValuePerTx": "1000000000000000000",
    "maxDailyVolume": "10000000000000000000",
    "maxWeeklyVolume": "50000000000000000000",
    "requireApproval": true
  },
  "duration": {
    "validFrom": "2024-01-01T00:00:00Z",
    "validUntil": "2024-12-31T23:59:59Z"
  },
  "conditions": [
    {
      "field": "protocol",
      "operator": "in",
      "value": ["uniswap-v3", "sushiswap"]
    }
  ]
}
```

### Agent Fixtures

#### Standard Agent
- Connected wallet as signer
- Single policy permission
- Well-behaved validation requests

#### Bot Agent
- Generated signer (private key)
- Multiple permissions
- High-frequency requests

#### Misbehaving Agent
- Repeatedly blocked actions
- Exceeds constraints
- Used to test audit trail

---

## Testing Environment

### Local Development
```bash
docker-compose up -d  # Postgres, etc.
go run ./cmd/server   # Backend
npm run dev          # Frontend (if testing UI)
```

### Test Database
- Migrations pre-applied
- Seeded with fixture policies, agents, permissions
- Fresh state before each test run
- Rollback after test (or separate test schema)

### Mock Smart Account
- Mock PermissionEnforcer for unit tests
- Testnet deployment (Sepolia) for integration
- Ganache/Hardhat fork for performance tests

### CI/CD Pipeline
- Run on every push to `backend/` or `contracts/`
- Unit tests (Go + Solidity)
- Integration tests against test DB
- Coverage report (target: 80%)
- Performance baseline check
- Deploy to staging on main branch

---

## Test Execution Plan

### Phase 1: Unit Tests (Existing)
- **Status:** Mostly complete (see `engine_test.go`)
- **Owner:** TBD
- **Duration:** 1 day (expand missing cases)
- **Target:** 80%+ coverage on policy engine

**New tests needed:**
- Case-insensitive action matching
- Token/protocol/chain wildcards
- Complex condition chains
- Daily/weekly volume (with DB mock)
- Daily usage reset logic

### Phase 2: Integration Tests
- **Duration:** 2-3 days
- **Owner:** TBD
- **Setup:** Docker + test DB
- **Target:** All API endpoints validated

**Core tests:**
- Policy CRUD operations
- Permission lifecycle
- Permission + policy queries
- Batch validation
- Simulation endpoint
- Audit log recording

### Phase 3: E2E Scenarios (Off-Chain)
- **Duration:** 2-3 days
- **Owner:** TBD
- **Scenarios:** All listed in section 3 above
- **Target:** 100% scenario coverage

### Phase 4: On-Chain E2E
- **Duration:** 3-5 days
- **Owner:** TBD
- **Setup:** Sepolia testnet or testnet fork
- **Target:** Account deployment, enforcement, event indexing

**Core flows:**
- Deploy smart account
- Validate on-chain
- Block invalid operation
- Emit & index events
- Dual-source audit logs

### Phase 5: Security & Performance
- **Duration:** 2-3 days
- **Owner:** TBD
- **Target:** No critical vulnerabilities, perf baselines met

**Tests:**
- Permission boundary tests
- Input validation (fuzzing)
- Load test (1000 req/s)
- Audit log integrity
- Smart account immutability

### Phase 6: User Acceptance (UAT)
- **Duration:** 1-2 days
- **Owner:** TBD
- **Stakeholders:** Product, DevOps
- **Focus:** Dashboard UX, API usability, observability

---

## Success Criteria

- [x] Unit tests: 80%+ code coverage (policy engine)
- [ ] Integration tests: All API endpoints passing
- [ ] E2E scenarios: All 12+ scenarios passing
- [ ] On-chain: Account deployment & enforcement working
- [ ] Security: No critical/high vulnerabilities
- [ ] Performance: <10ms p95 validation latency
- [ ] Audit: Off-chain + on-chain events captured correctly
- [ ] Error messages: Clear and actionable
- [ ] Documentation: API + policy syntax fully documented

---

## Open Questions / Clarifications Needed

1. **Multiple Permissions:** When an agent has multiple active permissions (policies), what's the evaluation logic?
   - First match wins?
   - All must approve (AND)?
   - Any can approve (OR)?

2. **Performance Targets:** What throughput/latency targets are critical?
   - Requests per second?
   - Acceptable p95/p99 latency?
   - Max audit log size?

3. **On-Chain Mainnet:** When will the system go live on mainnet?
   - Impacts testnet vs testnet fork testing strategy

4. **Smart Account Upgradability:** Are smart accounts upgradeable?
   - Impacts policy rollout strategy

5. **Audit Log Retention:** How long are logs kept?
   - Impacts performance testing data size

6. **Admin Capabilities:** Are there admin functions that bypass policies?
   - For emergency or testing?
   - How are they audited?

7. **Batch Validation:** Should batch requests succeed/fail atomically or independently?
   - Currently independent (each has own result)
   - Is this desired?

---

## Next Steps

1. **Review** — Shawn reviews this updated plan
2. **Prioritize** — Decide which phases/tests to execute first
3. **Assign** — Allocate owners to each test phase
4. **Implement** — Break tests into granular test cases, code them
5. **Execute** — Run tests, track results in test management system
6. **Iterate** — Fix failures, optimize performance, increase coverage

---

## Resources

- **Code:** `/home/node/8004-agent-permission-policy/`
- **Smart Contracts:** `/home/node/8004-agent-permission-policy/contracts/`
- **Backend Tests:** `/home/node/8004-agent-permission-policy/backend/internal/domain/policy/engine_test.go`
- **API Handlers:** `/home/node/8004-agent-permission-policy/backend/internal/api/handlers/validation.go`
