# LateDAO MVP Plan (Base + USDC, Option B Allowance)

## Goal
Build a 6-person “late policy” app where:
- Each member keeps funds in their own wallet (no escrow).
- Members grant the contract an allowance to pull up to a cap (e.g., 150 USDC).
- Policies must be created >= 36 hours before the event start time.
- Anyone can veto within the first 24 hours after policy creation.
- Voting opens at +36 hours after creation and lasts 12 hours.
- If >= 5/6 vote "late", the contract pulls the penalty from the accused’s wallet and sends to a treasury.

## Non-goals (MVP)
- No automatic “real-world lateness detection.” Minutes late are human-input.
- No dynamic group membership / join/leave flows.
- No fancy swaps / ETH funding. Use USDC directly.
- No governance tokens, NFTs, reputation, etc.

## Chain & Token
- Chain: Base mainnet (deploy after Base Sepolia testing).
- Token: native USDC on Base (6 decimals).

## Repository Structure (suggested)
- contracts/
  - src/
    - LateGroup.sol
  - test/
    - LateGroup.t.sol (Foundry) OR LateGroup.test.ts (Hardhat)
  - plan.md
- dapp/
  - src/...
  - plan.md (optional for UI)
- README.md

## Roles / Entities
- Members: fixed list of 6 addresses set at deploy time.
- Treasury: a fixed address set at deploy time.
- USDC: an IERC20 reference set at deploy time.

## Core Concepts
### Event ID
Represent each event as:
- eventId = keccak256(abi.encode(eventStartTime, eventSalt))
In MVP, eventSalt can be user-provided bytes32 to avoid collisions.

### Policy
A Policy targets one accused member for one eventId.
Fields:
- eventId (bytes32)
- eventStartTime (uint64)
- createdAt (uint64)
- vetoUntil = createdAt + 24h
- voteStarts = createdAt + 36h
- voteEnds = voteStarts + 12h
- accused (address)
- minutesLate (uint16)
- penaltyUSDC (uint32 or uint64 in 6-decimal units)
- vetoed (bool)
- executed (bool)
- votesFor (uint8)
- votedBitmap (uint8)          // bit i: member i has voted
- forBitmap (uint8)            // bit i: member i voted "for late"

## Rules
### Create Policy
Function: createPolicy(eventId, eventStartTime, accused, minutesLate)
- caller must be member
- accused must be member
- must not already exist: (eventId, accused) unique
- require eventStartTime >= now + 36 hours
- minutesLate must be <= MAX_LATE_MINUTES (e.g., 180)
- compute penaltyUSDC = f(minutesLate); cap at MAX_PENALTY_USDC
- sets timing windows based on createdAt

### Veto
Function: veto(eventId, accused)
- caller must be member
- now <= vetoUntil
- sets vetoed = true

### Vote
Function: vote(eventId, accused, support)
- caller must be member
- now in [voteStarts, voteEnds]
- not vetoed, not executed
- caller must not have voted
- if support: votesFor++

### Execute
Function: execute(eventId, accused)
- not vetoed, not executed
- now > voteEnds
- votesFor >= 5
- transferFrom(accused, treasury, penaltyUSDC)
- mark executed = true

## Penalty Function f(minutesLate)
MVP deterministic tiered schedule (example):
- 0-4  => 0
- 5-9  => 2 USDC
- 10-14 => 4 USDC
- 15-29 => 8 USDC
- 30-59 => 15 USDC
- 60+ => 25 USDC
Cap: MAX_PENALTY_USDC = 25 USDC

Store penalty in 6-decimal USDC units.

## Constants
- MEMBER_COUNT = 6
- THRESHOLD = 5
- MIN_CREATE_AHEAD = 36 hours
- VETO_WINDOW = 24 hours
- VOTE_DELAY = 36 hours
- VOTE_WINDOW = 12 hours

## Events (Solidity)
- PolicyCreated(eventId, accused, minutesLate, penaltyUSDC, createdAt, vetoUntil, voteStarts, voteEnds)
- PolicyVetoed(eventId, accused, vetoer)
- VoteCast(eventId, accused, voter, support)
- PenaltyExecuted(eventId, accused, penaltyUSDC, treasury)

## Security / Safety (MVP)
- Membership is immutable (hardcoded at deploy).
- No owner-only "drain" function.
- All state-changing functions require msg.sender is member.
- Disallow duplicate policies for same (eventId, accused).
- Cap minutesLate and penalty.
- Use bitmap voting to avoid storage-heavy mappings.
- Reverts on failed transferFrom; UI surfaces “allowance missing or insufficient funds.”

## Testing Checklist
### Unit tests (local)
- Create policy fails if eventStartTime < now + 36h
- Veto only within first 24h
- Vote only between voteStarts and voteEnds
- Double voting reverts
- Non-member cannot create/veto/vote/execute
- Execute requires votesFor >= 5
- Execute before voteEnds reverts
- Execute transfers correct amount
- Execute cannot be called twice
- Veto prevents voting/execution

### E2E (Base Sepolia)
- Deploy contract
- Use a mock USDC or test USDC
- Each member approves 150 USDC
- Create policy, warp time windows, vote, execute

## Dapp MVP Screens (TypeScript)
- Connect wallet
- Show members + status (approved? balance? optional)
- Approve USDC spend (150)
- Create policy form
- Policy detail view with timers + veto/vote/execute buttons
- Treasury balance display (read-only)

## Deliverables
- LateGroup.sol
- test suite (Foundry or Hardhat)
- deploy script for Base Sepolia + Base mainnet
- basic React/Next.js UI to drive the flow
