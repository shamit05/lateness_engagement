# LateDAO MVP Plan (Base + USDC, Option B Allowance)

## Goal
Build a 7-person “late policy” app where:
- Each member keeps funds in their own wallet (no escrow).
- Members grant the contract an allowance to pull up to a cap (e.g., 50 USDC).
- Members can call for a vote and input two fields: (1) event: brief description of what happened, (2) accused: the person who violated the rules, and (3) minutes late (this will be implemented in the UI). All 7 people will receive a notification to vote with two options: "late" and "on time". If >= 6/7 cast their vote "late", the contract pulls the penalty from the accused’s wallet and sends to a treasury.
- A vote stays open until all 7 people vote (no expiry time). All events currently being voted on should be displayed on the dashboard. Once all 7 people vote or when at least 2 people vote "on time", the event is taken off the dashboard.

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
- Members: fixed list of 7 addresses set at deploy time.
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
- voteEnds (uint64) // initially infinity, set when voting ends (all 7 voted OR >=2 voted "on time")
- accused (address)
- eventType (uint8) // 0 = minor, 1 = major
- minutesLate (uint16)
- penaltyUSDC (uint32 or uint64 in 6-decimal units)
- executed (bool)
- votesFor (uint8)
- votedBitmap (uint8)          // bit i: member i has voted
- forBitmap (uint8)            // bit i: member i voted "for late"

## Rules
### Create Policy
Function: createPolicy(eventId, eventDescription, accused, eventType, minutesLate, voteStarts)
- caller must be member
- accused must be member
- must not already exist: (eventId, accused) unique
- eventType must be "minor" (0) or "major" (1)
- minutesLate must be <= MAX_LATE_MINUTES (e.g., 180)
- compute penaltyUSDC = f(minutesLate, eventType); cap at MAX_PENALTY_USDC_MINOR or MAX_PENALTY_USDC_MAJOR

### Vote
Function: vote(eventId, accused, support)
- caller must be member
- now in [createdAt, voteEnds]
- caller must not have voted
- if support: votesFor++

### Execute
Function: execute(eventId, accused)
- now > voteEnds
- votesFor >= 6
- transferFrom(accused, treasury, penaltyUSDC)
- mark executed = true

## Penalty Function f(minutesLate, eventType)
MVP deterministic tiered schedules:

### Major Events (eventType = 1):
- 0-4  => 0
- 5-9  => 5 USDC
- 10-14 => 10 USDC
- 15-19 => 15 USDC
- 20+ => 20 USDC
Cap: MAX_PENALTY_USDC_MAJOR = 20 USDC

### Minor Events (eventType = 0):
- 0-4  => 0
- 5-9  => 1 USDC
- 10-14 => 2 USDC
- 15-19 => 3 USDC
- 20+ => 4 USDC
Cap: MAX_PENALTY_USDC_MINOR = 4 USDC

Store penalty in 6-decimal USDC units.

## Constants
- MEMBER_COUNT = 7
- THRESHOLD = 6

## Events (Solidity)
- PolicyCreated(eventId, eventDescription, accused, eventType, minutesLate, penaltyUSDC, createdAt)
- VoteCast(eventId, accused, voter, support)
- VotingEnded(eventId, accused, voteEnds, reason) // reason: "all_voted" or "threshold_reached"
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
- Double voting reverts
- Non-member cannot create/veto/vote/execute
- Execute requires votesFor >= 6
- Execute before voteEnds (all 7 people have voted OR >=2 people voted "on time") reverts
- Execute transfers correct amount
- Execute cannot be called twice

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
- Policy detail view with all casted votes (person + decision) and an execute button
- Treasury balance display (read-only)

## Deliverables
- LateGroup.sol
- test suite (Foundry or Hardhat)
- deploy script for Base Sepolia + Base mainnet
- basic React/Next.js UI to drive the flow
