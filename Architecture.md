# Rift on Stellar: Netting and Smart Accounts



## 1. Summary

- Businesses pay and get paid in local currency.
- Rift nets obligations every cycle.
- Only the residual moves, as USDC on Stellar.
- Graph: Kenya ↔ Tanzania, plus Chinese suppliers.
- Business funds sit in passkey smart accounts. Rift holds no signing key.

## 2. Architecture

Component diagram. Subgraphs are trust boundaries, as the Stellar threat-model guide requires.

```mermaid
flowchart TB
  subgraph EXT["External entities"]
    KEB[KE businesses]
    TZB[TZ businesses]
    CNS[Chinese suppliers]
    LP[Liquidity providers]
  end

  subgraph PARTNERS["Trust boundary: licensed partners"]
    KEPSP["KE PSP / bank<br>KES float"]
    TZPSP["TZ PSP<br>TZS float"]
    PAY["Payout partner<br>Bridge / BlindPay"]
  end

  subgraph RIFT["Trust boundary: Rift-operated"]
    API[Rift backend + app]
    NODES["Country node accounts<br>multisig"]
    LEDGER[("Off-chain ledger<br>trade graph")]
  end

  subgraph STELLAR["Trust boundary: Stellar / Soroban"]
    SA["Business smart accounts<br>passkey signers"]
    REG[Registry]
    NET[Netting contract]
    POOL[("Liquidity pool<br>USDC")]
  end

  KEB -- pay KES --> KEPSP
  TZB -- pay TZS --> TZPSP
  KEB & TZB -- signed obligation --> SA
  SA --> NET
  REG --> NET
  KEPSP & TZPSP -- pre-funding confirmed --> API
  API --> NODES
  API --> LEDGER
  NODES -- repay + fee --> POOL
  NET -- advance --> POOL
  LP -- deposit --> POOL
  POOL -- USDC --> PAY
  PAY -- USD / HKD / USDC --> CNS
```

## 3. Participants

| Participant | Role | Stellar account |
| --- | --- | --- |
| KE / TZ businesses | Pre-fund locally, sign obligations | Passkey smart account |
| Chinese suppliers | Get paid | Smart account or payout partner |
| Country nodes | Hold local float, repay pool | Classic multisig |
| Liquidity providers | Fund the residual | Any Stellar wallet |

## 4. Netting cycle

### Example (illustrative)

| Payer | Payee | USD |
| --- | --- | --- |
| KE importers | TZ suppliers | 250,000 |
| TZ importers | KE suppliers | 240,000 |
| KE importers | CN suppliers | 100,000 |
| TZ importers | CN suppliers | 60,000 |
| **Total** | | **650,000** |

| Result | USD |
| --- | --- |
| Trade settled | 650,000 |
| KE ↔ TZ net | 10,000 |
| Dollars needed | 160,000 (= China imports) |

### Sequence

```mermaid
sequenceDiagram
  participant B as KE / TZ business
  participant N as Country node
  participant NC as Netting contract
  participant P as Liquidity pool
  participant CN as CN supplier

  B->>N: Pay local currency (pre-fund)
  N->>NC: Confirm pre-funding
  B->>NC: submit_obligation (passkey-signed)
  NC->>NC: Update net per node
  N->>B: Pay local recipients locally
  Note over NC: Cycle ends
  NC->>NC: close_cycle()
  NC->>P: advance(160K)
  P->>CN: 160K USDC
  N->>P: repay(110K + 50K + fee)
```

### Cycle states

```mermaid
stateDiagram-v2
  [*] --> Open
  Open --> Closed: close_cycle()
  Closed --> Advanced: pool.advance()
  Advanced --> Repaid: nodes repay + fee
  Advanced --> Overdue: repay late
  Overdue --> Repaid: recovered
  Repaid --> [*]
  note right of Overdue: failure handling TBD
```

## 5. Soroban contracts

```mermaid
classDiagram
  class Registry {
    +register(business, node, limits)
    +set_kyb(business, status)
  }
  class NettingContract {
    -net_per_node
    +submit_obligation(payer, payee, usd_amount, fx_rate, ref_hash)
    +close_cycle()
    +settle(cycle)
  }
  class LiquidityPool {
    +deposit(amount)
    +withdraw(shares)
    +advance(cycle, amount)
    +repay(cycle, amount, fee)
  }
  class SmartAccount {
    -signers
    -policies
    +check_auth()
  }
  NettingContract --> Registry : checks KYB
  NettingContract --> LiquidityPool : only caller of advance
  SmartAccount --> NettingContract : authorizes obligations
```

Rules:

- Running net per node, so `close_cycle()` is constant cost.
- Settlement is one atomic transaction.
- Pool caps per node and per cycle.

## 6. Smart accounts

| Policy | Value (placeholder) |
| --- | --- |
| Signers | Director, finance officer, recovery key |
| Under $10K | 1 signer |
| Over $10K | 2 signers (maker-checker) |
| Daily limit | $50K |
| Allowlist | Netting contract, approved suppliers |

### Signing

```mermaid
sequenceDiagram
  participant D as Director
  participant F as Finance officer
  participant App as Rift app
  participant SA as Smart account
  participant NC as Netting contract

  D->>App: Pay supplier 20K
  App->>D: Show decoded intent
  D->>App: Passkey signature
  App->>F: Approval request (over 10K)
  F->>App: Passkey signature
  App->>SA: Fee-bumped tx (Rift pays fee)
  SA->>SA: check_auth: signers + policies
  SA->>NC: submit_obligation
```

### Recovery

```mermaid
stateDiagram-v2
  [*] --> Active
  Active --> RecoveryPending: recovery key requests new signer
  RecoveryPending --> Active: any passkey cancels
  RecoveryPending --> Active: 72h passes, signer added
```

### Wallet OS vs Stellar

| | Wallet OS (EVM) | Stellar smart account |
| --- | --- | --- |
| Verifies consent | Rift enclave | Stellar network |
| Rift signing key | Yes, in enclave | None |
| Re-pin gap | Yes | No |
| Fees | Paymaster | Fee-bump, CAP-33 sponsorship |

## 7. Integrations

| Need | Integration List item | Time |
| --- | --- | --- |
| Liquidity from Base | CCTP | 1–5 days |
| Pay Chinese suppliers | Bridge or BlindPay | 1–5 days / 1–2 weeks |
| KES/TZS anchor (SEP-31) | Anchor Platform | 1+ month |
| Wallets | Stellar Wallets Kit | < 1 day |

Local ramps: Pretium, Paycrest, SasaPay, TEMBO.

## 8. Open decisions

| Topic | Recommendation |
| --- | --- |
| Privacy | Pseudonymous IDs + invoice hash on-chain; graph off-chain |
| Cycle length | Daily |
| China leg | Payout only for now |
| Licensing | Partner PSPs + bank; confirm before 4 Nov 2026 |
| Pool risk | Caps + stress buffer |
| Failures | Design next |

## 9. SCF fit (Integration Track)

| Requirement | How |
| --- | --- |
| Stellar is core | USDC settles on Stellar every cycle |
| Open source | All contracts public |
| Integration List | CCTP, Bridge/BlindPay, Anchor Platform, Wallets Kit |
| Tranche 2 threat model | Diagram in section 2 + STRIDE |
| Final-tranche metric | Cumulative USDC through registered contracts |
| Traction | $600K on Base, wallet addresses + Dune |
| Content rules | No "yield", "returns", "earn" |
