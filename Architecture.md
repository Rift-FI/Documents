# Rift: Stellar Integration Plan

## 1. Overview

- Rift settles East African trade payments on Stellar.
- Businesses pay in local currency (KES, TZS). Rift nets obligations daily in a Soroban contract.
- Only the residual moves, as USDC on Stellar, to Tanzania or to international suppliers.
- International suppliers = suppliers outside East Africa. We start with China, where most of our businesses' suppliers are.

## 2. Architecture

```mermaid
flowchart TB
  subgraph EXT["Users"]
    KEB[KE / TZ businesses]
    INTL[International suppliers]
    LP[Liquidity providers]
    BASE["USDC on Base"]
  end

  subgraph BB["SCF building blocks"]
    AP["Anchor Platform<br>SEP-10/45/12/31/38"]
    CCTP["CCTP<br>native USDC"]
    SWK["Stellar Wallets Kit"]
  end

  subgraph STELLAR["Stellar / Soroban"]
    SA["Business smart accounts<br>passkey signers"]
    NET[Netting contract]
    POOL[("Liquidity pool<br>USDC")]
  end

  KEB -- KES / TZS via SasaPay, TEMBO --> AP
  AP -- confirm pre-funding --> NET
  KEB -- signed obligation --> SA --> NET
  NET -- advance residual --> POOL
  POOL -- USDC to Stellar address --> INTL
  BASE --> CCTP -- mint USDC --> POOL
  LP --> SWK -- deposit / withdraw --> POOL
  AP -- repay USDC --> POOL
```

## 3. End-to-end flow

```mermaid
sequenceDiagram
  participant B as Business
  participant AP as Anchor Platform
  participant NC as Netting contract
  participant P as Liquidity pool
  participant S as Intl supplier

  B->>AP: SEP-10/45 auth, SEP-12 KYB
  B->>AP: SEP-38 quote (rate locked)
  B->>AP: Pay KES / TZS
  AP->>NC: confirm_prefund
  B->>NC: submit_obligation (passkey-signed)
  Note over NC: Daily cycle close: net per country
  NC->>P: advance(residual)
  P->>S: USDC to supplier's Stellar address
  AP->>P: repay(residual + fee)
```

## 4. Building block integration plan

### 4.1 Anchor Platform

| | |
| --- | --- |
| Role | Kenya and Tanzania nodes for local-currency in and out |
| SEP-1 | `stellar.toml` for each node |
| SEP-10 / SEP-45 | Partner anchors authenticate with Stellar accounts (SEP-10); business smart accounts authenticate with SEP-45 |
| SEP-12 | KYB for each business |
| SEP-38 | Firm quote; locks the FX rate when the obligation is submitted |
| SEP-31 | Receive cross-border payments; pay out KES/TZS via SasaPay and TEMBO |
| Rift builds | Business-server callbacks linking Anchor Platform to SasaPay, TEMBO and the netting contract |
| Done when | A SEP-31 payment from a test sender pays out in KES and TZS |

### 4.2 CCTP

| | |
| --- | --- |
| Role | Bring USDC held on Base (and other chains) onto Stellar, 1:1 |
| Flow | `depositForBurn` on Base with Stellar as destination; native USDC minted on Stellar to the pool |
| Who uses it | Rift treasury, liquidity providers, businesses moving existing Base balances |
| Done when | A Base → Stellar transfer lands in the pool |

### 4.3 Stellar Wallets Kit

| | |
| --- | --- |
| Role | Liquidity provider wallet connection |
| Wallets | Freighter, LOBSTR, xBull and others |
| Flow | LP dashboard signs `deposit` and `withdraw` calls to the pool contract |
| Done when | An LP deposits and withdraws USDC on mainnet |

## 5. Stellar components Rift builds

```mermaid
classDiagram
  class NettingContract {
    +confirm_prefund(node, ref, amount)
    +submit_obligation(payer, payee, amount, fx_rate, ref)
    +close_cycle()
  }
  class LiquidityPool {
    +deposit(amount)
    +withdraw(shares)
    +advance(cycle, amount)
    +repay(cycle, amount, fee)
  }
  class SmartAccount {
    +__check_auth()
    +add_signer(signer)
    +request_recovery(new_signer)
  }
  SmartAccount --> NettingContract : signs obligations
  NettingContract --> LiquidityPool : advances residual
```

| Stellar feature | Use |
| --- | --- |
| Soroban (Rust, soroban-sdk) | Netting contract, liquidity pool, smart accounts; all open source |
| USDC Stellar Asset Contract | Every USDC transfer in and out of the pool |
| secp256r1 (CAP-51) | Business passkeys verified on-chain in `__check_auth` |
| Fee-bump (CAP-15) | Rift pays network fees for businesses |
| Fee sponsorship | Rift covers smart account deployment, storage rent and network fees |
| Stellar RPC + contract events | Submission, simulation, monitoring |
