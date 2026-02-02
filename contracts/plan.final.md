# LateDAO Final Plan (Production-Grade)

## Goal
Ship a friend-group enforcement app that is:
- Safe (caps + predictable behavior)
- Hard to grief (rate limits + clear exits)
- Smooth UX (signature approvals, clear timers, auto-refresh state)
- Extensible (group config, upgrades via safe patterns or new deployments)

Still based on Option B (allowances), but with better UX and stronger operational tooling.

## Final Architecture Overview
- Solidity contracts:
  1) LateGroup (core policy + voting + execution)
  2) LateGroupFactory (optional): creates groups with standardized parameters
  3) Config / Governance module (optional): allow 6/6 unanimous changes to config
- TypeScript dapp:
  - group creation (if using Factory)
  - member onboarding (approve/permit)
  - policy creation / veto / vote / execute
  - audit view + history
- Indexer / storage:
  - Use events + a lightweight indexer (e.g., The Graph or a simple server) to power UI history and analytics.

## Key Improvements vs MVP
### 1) Better approvals UX (reduce “approve” friction)
Support one or more:
- EIP-2612 permit (if token supports)
- Permit2 (Uniswap Permit2) for broad token support
Fallback: standard approve()

User experience:
- “Sign once to authorize 150 USDC cap” (permit) rather than a full approve tx where possible.

### 2) Stronger anti-griefing
- Rate limit policy creation per (eventId) and/or per member:
  - e.g., max 1 active policy per event
  - or max N policies per week (optional)
- Require a minimum lead time for policy creation AND store eventStartTime:
  - already required: eventStartTime >= now + 36h
- Minutes late cannot exceed MAX_LATE_MINUTES
- Penalty cannot exceed MAX_PENALTY_USDC

### 3) Configurable group parameters with unanimous consent
Allow 6/6 vote to change:
- treasury address
- penalty schedule parameters (or select among predefined schedules)
- caps (MAX_PENALTY_USDC, MAX_LATE_MINUTES)
- time windows (VETO_WINDOW, VOTE_DELAY, VOTE_WINDOW)
Constraints:
- changes only apply to future policies (not retroactive)

### 4) Membership lifecycle
Option B cannot prevent someone from revoking approval. So formalize it:
- Member status:
  - ACTIVE: allowance >= requiredCap and not “paused”
  - INACTIVE: allowance too low or member toggled “inactive”
Rules:
- Only ACTIVE members can vote/create/veto (configurable).
- If accused is INACTIVE at execution time, execution fails, but UI flags it.

Optional: “leaveGroup()” with a cooldown:
- leaving sets INACTIVE, removes ability to propose/vote.
- but membership list remains for historical consistency.

### 5) More robust policy identity
Replace (eventId, accused) with a single policyId:
- policyId = keccak256(abi.encode(groupId, eventId, accused))
Or add a nonce:
- policyId = keccak256(abi.encode(groupId, eventId, accused, nonce))
This makes UI + indexing easier.

### 6) Voting on minutesLate (optional, but better)
Instead of creator choosing minutesLate, do:
- proposer submits minutesLate
- voters vote for/against the proposal
OR
- voters vote among a small set of minute buckets (e.g., 5, 10, 15, 30, 60)
This reduces “minutes manipulation” disputes.

### 7) Execution flexibility
Allow execution:
- after voteEnd, OR
- as soon as votesFor reaches threshold after voteStart (early execution)
MVP uses after voteEnd only; final can support early execution to reduce waiting.

### 8) Observability + monitoring
- Emit events for every state change (already)
- Add view functions:
  - getPolicy(policyId)
  - getMemberIndex(address)
  - getVoteState(policyId)
- Provide a simple script to:
  - read policies from chain
  - compute “current stage” for each policy
  - output status for debugging

### 9) Gas + storage optimizations
- Pack structs tightly
- Keep voting bitmaps
- Avoid unbounded loops (MEMBER_COUNT is constant 6)
- Use custom errors

### 10) Upgrade / iteration strategy
Avoid in-place upgrades if possible:
- Use a factory that deploys new groups
- If you need upgrades later:
  - deploy a new version and migrate
  - or use a proxy only if you are comfortable with admin risk and have strong controls

## Swap / ETH funding (optional module)
If you later want ETH funding:
- Use WETH allowances (ERC-20) instead of raw ETH.
- If treasury must be USDC:
  - swap WETH -> USDC during execute via a known router
  - include slippage limits, deadlines, and safe failure modes

But the recommended production default remains USDC-native.

## Final Dapp Features
- Onboarding:
  - connect wallet
  - verify membership
  - approve/permit authorization cap
- Policies:
  - list policies (active / veto window / voting window / ended)
  - create policy (with validations + countdown)
  - veto button (with time remaining)
  - vote button (with time remaining)
  - execute button (with error explanation if allowance missing)
- Transparency:
  - show how penalty was calculated
  - show who voted for/against
  - show treasury totals and per-member penalties (derived from events)
- UX:
  - automatic timer updates
  - notifications (optional)

## Final Deliverables
- contracts:
  - LateGroup.sol (core)
  - LateGroupFactory.sol (optional)
  - full test suite + fuzz tests
  - deploy scripts for Base Sepolia + Base mainnet
- dapp:
  - Next.js UI
  - wallet connect + Base network switch
  - indexer or event-based history view
- ops:
  - “status” CLI script
  - docs: configuration, deploying, adding members, troubleshooting
