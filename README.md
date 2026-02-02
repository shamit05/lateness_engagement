# LateDAO (Base) — Friend-Group Lateness Penalties

This repo contains:
- Solidity contracts that enforce timing + voting rules on-chain
- A TypeScript dapp for your 6-person group to create policies, veto, vote, and execute penalties
- Two implementation plans: MVP and production-grade final version

---

## Docs / Plans

### Contracts
- `contracts/plan.md` — MVP scope, rules, storage model, tests
- `contracts/plan.final.md` — production-grade feature set and hardening roadmap

### (Optional) Dapp plan
If you want a dedicated UI plan, add:
- `dapp/plan.md` — screens, flows, state model, indexing strategy

---

## Architecture Overview

### Solidity (contracts/)
The on-chain system implements:
- Fixed 6-member group
- Policy creation constraint: must be created >= 36h before event start
- Veto window: first 24h after creation
- Vote window: starts 36h after creation, lasts 12h
- Threshold: 5/6 votes
- Execution: if threshold met, pull penalty from accused via ERC-20 allowance and send to treasury

Token:
- Native USDC on Base (recommended)

Important constraint:
- This design uses allowance-based charging (Option B). Any member can revoke allowance; the app should treat revocation as leaving/pausing participation.

### TypeScript Dapp (dapp/)
The UI handles:
- wallet connect + Base network setup
- “approve/permit” flow for spending cap
- creating policies (with client-side validations)
- veto/vote/execute interactions
- policy timeline display
- history views from events

---

## Getting Started

### 1) Choose your dev stack
Contracts:
- Foundry (fastest solidity iteration), or
- Hardhat (best if you want TS tests + tight dapp integration)

Dapp:
- Next.js + wagmi/viem (typical stack)

### 2) Local development flow
1. Run a local chain (Anvil or Hardhat Network)
2. Deploy the contract locally
3. Use a mock USDC token in tests
4. Run unit tests (time-warp to simulate veto/voting windows)
5. Run the dapp against localhost contracts

### 3) Testnet flow (Base Sepolia)
1. Deploy contracts to Base Sepolia
2. Fund test wallets with test ETH
3. Use a mock stablecoin (or test USDC if available/desired)
4. Run a full end-to-end cycle:
   approve -> create -> veto/vote -> execute

### 4) Mainnet flow (Base)
1. Deploy to Base mainnet
2. Use native USDC on Base
3. Have members approve the contract up to the cap (e.g., 150 USDC)
4. Run your first real policy cycle

---

## Repo Layout (suggested)

- `contracts/`
  - `src/` Solidity contracts
  - `test/` unit tests
  - `script/` deploy scripts
  - `plan.md`
  - `plan.final.md`

- `dapp/`
  - `src/` Next.js app
  - `components/`
  - `lib/` chain + contract config
  - `public/`

---

## Where to start
If you want the fastest path to a working demo:
1. Read `contracts/plan.md`
2. Implement LateGroup.sol + tests with mock USDC
3. Add a minimal dapp:
   - connect wallet
   - approve cap
   - create policy
   - vote
   - execute

Then iterate toward `contracts/plan.final.md`.

---

## Notes / Limitations
- “Minutes late” must be human-entered and agreed by vote (blockchain cannot observe real-world arrival time).
- Option B enforcement depends on allowance + wallet balance. If a member revokes allowance, execution will fail until they restore it.

---

## License
TBD
