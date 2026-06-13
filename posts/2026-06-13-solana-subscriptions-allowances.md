---
layout: post
title: "Solana Subscriptions & Allowances: one onchain primitive for recurring billing, delegated spend, and agent budgets"
date: 2026-06-13
categories: solana payments technical-deep-dive
---

# Solana Subscriptions & Allowances: one onchain primitive for recurring billing, delegated spend, and agent budgets

Solana's new Subscriptions & Allowances program turns a pattern that every payment product used to rebuild separately — recurring pulls, capped delegated spending, and subscription tiers — into a shared onchain primitive.

The important part is not simply that Solana can now do “subscriptions.” The important part is the shape of the abstraction:

> A user gives one program-controlled authority the right to move tokens from a token account, but every actual transfer must pass through a separate onchain authorization account that encodes the user's constraints.

That design creates a reusable middle layer between two extremes:

- **Manual push payments**, where the user signs every invoice, payroll run, or renewal.
- **Broad custody / blanket approvals**, where a third party can move funds with too much discretion.

Subscriptions & Allowances is a middle path: reusable, pull-based, but bounded by explicit onchain terms.

This article explains the architecture, the three models inside the program, the tradeoffs for builders, and why the primitive is particularly relevant for SaaS billing, stablecoin invoices, embedded wallets, payroll, and AI-agent commerce.

---

## 1. What launched

Solana Foundation announced that Subscriptions & Allowances are live on mainnet as an audited, open-source shared program for recurring billing and delegated spending. The reference implementation is the `solana-foundation/subscriptions` repository, described as “Solana program and clients for managed token delegations on SPL Token and Token-2022.”

Key facts from the public repository and announcement:

- Program ID: `De1egAFMkMWZSN5rYXRj9CAdheBamobVNubTsi9avR44`
- Supported token programs: SPL Token and Token-2022
- Core models: fixed delegations, recurring delegations, and subscription plans
- SDK: TypeScript client package `@solana/subscriptions`, plus a generated Rust client
- Framework/tooling: Pinocchio for the Solana program, Codama for IDL and client generation
- Events: self-CPI event emission for indexers
- Security baseline: audited by Cantina, with audit status tracked in the repository

The program is not a hosted billing company. It is closer to a payments rail: a standard onchain contract that wallets, merchants, payroll tools, SaaS companies, payment gateways, and agent platforms can integrate.

---

## 2. The problem: Solana token delegation is single-delegate by default

The base SPL Token delegation model lets a token-account owner approve a delegate to spend from a token account. That is useful, but it has a major limitation for real products: a token account can only have one delegate.

That is awkward if the same wallet wants to authorize several independent relationships at once:

- a SaaS subscription,
- a card program,
- a payroll tool,
- an AI agent budget,
- a recurring invoice collector,
- and a one-off capped allowance.

If every product tries to become the delegate directly, they conflict with each other. If a wallet grants broad authority to a single custodian or processor, the trust surface becomes too large.

Subscriptions & Allowances solves this by introducing a program-controlled intermediate authority.

---

## 3. Core architecture: Subscription Authority + Delegation PDAs

For each `(user, mint)` pair, the program creates a **Subscription Authority** PDA. The user then approves this Subscription Authority as the single delegate on the user's token account with `u64::MAX` approval.

That sounds scary until you add the second half of the design:

> The Subscription Authority cannot move funds just because it has token approval. It can only execute a transfer when a matching Delegation PDA authorizes that transfer under its stored constraints.

So the system has two layers:

1. **Subscription Authority PDA**
   - One per user and token mint.
   - Receives the token-account delegate approval.
   - Acts as the common transfer authority for all delegated payment flows.

2. **Delegation PDA**
   - One per authorization relationship.
   - Stores the rules: who can pull, how much, how often, expiry, plan reference, billing state, and versioning data.
   - Must validate before the Subscription Authority signs the token transfer.

This is the key idea: the base token account still only has one delegate, but the program multiplexes many logical delegations behind that single delegate.

### Simplified transfer flow

A delegated transfer looks like this:

1. User initializes a Subscription Authority for a token mint.
2. User creates a fixed delegation, recurring delegation, or subscription delegation.
3. Delegate or authorized puller calls the relevant transfer instruction.
4. Program loads the delegation account.
5. Program checks amount, period, expiry, plan terms, caller authorization, and stale-authority protection.
6. If valid, Subscription Authority signs a CPI to the token program.
7. Token program transfers funds from the user's token account.
8. Program updates the delegation's accounting state and emits an event.

This gives builders pull-based payments without forcing every integrator to maintain its own bespoke billing contract.

---

## 4. The three models

The program supports three related, but distinct, authorization patterns.

### Model 1: Fixed delegations — capped one-time allowance

A fixed delegation lets a user authorize a delegatee to spend up to a total cap, optionally before an expiry timestamp.

Example:

> “Allow this agent to spend up to 100 USDC before Friday.”

Good fits:

- AI-agent shopping or task budgets
- One-time procurement
- Capped user-approved wallet automation
- Expense-card-style flows
- Temporary operational allowances

The delegate can call `transfer_fixed`, but only until the remaining allowance is exhausted or the delegation expires. The user can revoke the delegation by closing the PDA.

### Model 2: Recurring delegations — user-defined periodic pull rights

A recurring delegation lets a user authorize a delegatee to spend up to a maximum amount per period.

Example:

> “Allow this contractor to pull up to 500 USDC every two weeks until this date.”

Good fits:

- Payroll-like relationships
- Contractor retainers
- Repeated payouts
- Recurring invoices where terms are custom per relationship
- Budgeted agent operations that reset periodically

The program tracks the current period and amount already pulled. Once a new period begins, the allowance resets according to the configured cadence.

### Model 3: Subscription plans — merchant-published billing terms

Subscription plans are the merchant-facing model. A merchant creates a Plan PDA with reusable billing terms: amount, period, mint, destinations, metadata, and authorized pullers.

Users then subscribe to the plan, creating a Subscription Delegation PDA that references the plan and snapshots its terms.

Example:

> “Subscribe to a 49 USDC/month API tier.”

Good fits:

- SaaS subscriptions
- API billing
- stablecoin invoice collection
- membership products
- payment-gateway integrations
- embedded-wallet checkout flows

The plan model adds two important properties.

First, the merchant does not need a fresh bespoke contract or delegation design for every subscriber. One plan can serve many subscribers.

Second, core billing terms are immutable. A merchant cannot silently change a live subscriber from 49 USDC/month to 199 USDC/month. If the merchant wants a new price, they sunset the old plan and create a new one.

---

## 5. Why the plan snapshot matters

The plan architecture is subtle and important.

When a subscriber subscribes, the program snapshots the plan's terms into the subscription delegation. At transfer time, the program validates the subscription's stored terms against the live plan.

This matters because onchain account addresses can be reused or recreated in some designs. If a merchant could delete and recreate a plan at the same address with different terms, old subscribers could be exposed to a “ghost plan” problem. The snapshot-and-validate pattern reduces that risk.

The plan is therefore not just a metadata object. It is a reusable commitment surface:

- The merchant publishes terms.
- The subscriber accepts those terms.
- The subscription stores the accepted terms.
- Transfer logic checks both subscription state and plan authorization.

That structure is closer to a signed service agreement than a simple recurring-transfer cron job.

---

## 6. Token-2022 support and transfer hooks

The repository states support for both SPL Token and Token-2022, including mints configured with Transfer Hooks.

That is significant because recurring payments are often not just about moving money. Regulated stablecoins, permissioned assets, loyalty assets, and enterprise payment tokens may need additional policy checks.

With Token-2022 transfer hooks, a mint can require extra validation during transfers. The subscriptions program forwards the required hook accounts into the Token-2022 `TransferChecked` CPI, so the mint's active policy context still applies.

In practical terms: using Subscriptions & Allowances should not be a bypass around token-level compliance or transfer policy. If the token has transfer-hook rules, delegated subscription transfers must still satisfy them.

---

## 7. Developer surface: SDK, PDAs, and events

The TypeScript client exposes a high-level `SubscriptionsClient` with methods for the main lifecycle:

- `initSubscriptionAuthority` and `closeSubscriptionAuthority`
- `createFixedDelegation` and `transferFixed`
- `createRecurringDelegation` and `transferRecurring`
- `createPlan`, `updatePlan`, and `deletePlan`
- `subscribe`, `cancelSubscription`, `resumeSubscription`, and `transferSubscription`
- `revokeDelegation`
- read helpers such as `getDelegationsForWallet`, `getPlansForOwner`, and `isSubscriptionAuthorityInitialized`

The PDA helper layer includes functions for deriving:

- Subscription Authority PDA
- Delegation PDA
- Plan PDA
- Subscription PDA
- Event Authority PDA

For indexers and product analytics, the program emits onchain events via self-CPI. That gives downstream systems a cleaner way to track subscription created/cancelled events and payment transfers without inventing their own parsing scheme from scratch.

---

## 8. Cost model: rent is real, but recoverable

The repository publishes rent estimates for the main account flows:

- Enable authority for a mint: 0.00162864 SOL
- Merchant creates a plan: 0.00430824 SOL
- Subscribe to a plan: 0.00196968 SOL
- Grant fixed delegation: 0.00219240 SOL
- Grant recurring delegation: 0.00235944 SOL

The rent is recoverable when the relevant account is closed and rent is returned to the original payer. Still, builders should not ignore it.

For consumer-scale subscription apps, the account-creation step is part of UX and unit economics. Someone must pay rent up front: the user, the merchant, a sponsor, or a wallet/payments provider. The good news is that the plan model amortizes merchant setup: one Plan PDA can support many subscribers.

---

## 9. Benefits: what becomes easier

### Fewer custom billing contracts

Before this primitive, each team that wanted recurring payments needed to build authorization, period accounting, pull rights, cancellation semantics, event indexing, and token-program compatibility itself. That means more duplicated engineering and more audits.

A shared program lets teams integrate a standard, reviewed primitive instead.

### Onchain terms instead of private billing state

Subscription terms can be visible and verifiable. Users and wallets can inspect what they are authorizing before subscribing. Indexers can track state directly.

### Better UX for stablecoin SaaS billing

A SaaS product can charge in USDC-like assets without sending invoices every month or asking the user to sign each renewal manually.

### Safer agent commerce

AI agents are the clearest “new” use case. Agents need budgets, not unlimited wallet access. A fixed or recurring delegation gives an agent a bounded spend authority with explicit caps and expiry.

### Compatibility with wallets and smart accounts

The program has been tested with Squads multisig and Swig smart-wallet flows, according to external reporting and repository context. That matters because many real business payments involve shared custody, policy wallets, or smart-account authorization.

---

## 10. Tradeoffs and risks

### The `u64::MAX` approval is conceptually uncomfortable

The Subscription Authority receives maximum token approval at the token-program layer. Security depends on the subscriptions program correctly enforcing Delegation PDA constraints.

That is a reasonable architecture, but wallets should explain it clearly. The user's mental model should not be “I gave this merchant unlimited access.” It should be “I enabled a shared authority, and this specific delegation constrains what can be pulled.”

### Program risk becomes shared infrastructure risk

A shared billing primitive reduces duplicated audits, but it also concentrates risk. If many merchants use the same program, a bug in that program has ecosystem-level impact.

Mitigations include audits, conservative upgrades, transparent audit-status tracking, formal verification where possible, bounty programs, and wallet-side risk disclosures.

### Indexing and UX still matter

The program emits events, but production apps still need reliable indexers, retry logic, customer support workflows, and clear subscriber dashboards.

A user must be able to answer:

- What did I authorize?
- Who can pull funds?
- How much can they pull?
- When does it reset?
- How do I cancel?
- What happens if my wallet has insufficient funds?

### Failed payments are a product problem

Onchain programs can enforce terms, but they do not magically solve insufficient balances, retries, dunning emails, grace periods, or service suspension logic. SaaS merchants still need product logic around failed pulls.

### Plan immutability is good for trust but adds migration work

Immutable core terms protect subscribers. But merchants need clean flows for sunsetting old plans, migrating customers, communicating new terms, and supporting grandfathered pricing.

### Compliance does not disappear

A subscription rail is not a legal wrapper. Stablecoin billing, payroll, contractor payments, and merchant services may still involve tax, payroll, sanctions, consumer-protection, accounting, and money-transmission questions depending on jurisdiction.

---

## 11. Business opportunities enabled by the primitive

### 1. Stablecoin-native SaaS billing

A developer-tool company can publish API tiers onchain. Users subscribe once, and the merchant pulls stablecoins each billing period. Helius is already cited by Solana as an integration path for API tier billing.

### 2. Embedded-wallet checkout

Wallet infrastructure providers can let users authorize recurring checkout flows in one wallet interaction, instead of forcing merchants to manage card rails or custom token approvals.

### 3. Onchain payroll and contractor retainers

Recurring delegations are useful when the payer defines the budget: “this contractor can pull up to X every two weeks.” This is not exactly traditional payroll automation, but it is a strong primitive for retainers, contributor payments, and recurring operational disbursements.

### 4. Agent-controlled budgets

An AI agent can be given a fixed or recurring allowance. The user can revoke it, cap it, and set expiry. This is much safer than funding a hot wallet with unlimited discretion.

### 5. Subscription marketplaces

Because plans are onchain and discoverable, wallets or marketplaces can display active subscription offers, compare plans, and help users manage recurring commitments across merchants.

---

## 12. Canadian examples: who could use this

The bounty asks for Canadian projects or Canadian Web2 companies that could benefit from the primitive. Here are several realistic fits.

### Shopify — merchant billing and app subscriptions

Shopify is one of Canada's most important commerce platforms. A Solana subscription rail is not a replacement for Shopify's existing payments stack, but it is relevant for crypto-native merchants, app-store subscriptions, and stablecoin settlement experiments.

A Shopify-adjacent integration could let merchants publish stablecoin-denominated plans for digital goods, memberships, wholesale relationships, or recurring B2B services, while leaving customer-facing UX inside a familiar commerce interface.

### Lightspeed — recurring merchant software and B2B payments

Lightspeed serves retail and hospitality merchants. Many of those merchants pay recurring SaaS fees, buy supplies, and manage vendor relationships. A stablecoin subscription primitive could support merchant software billing, recurring supplier invoices, or cross-border B2B subscription flows where card fees and international settlement are painful.

### Wealthsimple — account automation with user-controlled limits

Wealthsimple is a Canadian fintech brand with a large consumer trust surface. If a regulated fintech were to experiment with onchain money movement, capped allowances are a safer mental model than broad third-party access. Users could authorize bounded recurring transfers, app budgets, or automated payment relationships while retaining revocation rights.

### Figment — infrastructure subscriptions and staking operations

Figment is a Canadian blockchain infrastructure company. Developer and institutional infrastructure products often have recurring billing, usage tiers, and operational retainers. Subscription Plans could fit API/data/infrastructure subscriptions; recurring delegations could fit recurring operational or service-provider payments.

### Shakepay — consumer crypto payments and recurring buys

Shakepay is a Canadian crypto platform. A subscriptions-and-allowances pattern is relevant to recurring crypto purchases, merchant rewards, or capped spending relationships where users want automation without granting unlimited authority.

These examples are not claims that the companies are integrating the program today. They are product-fit examples showing where the primitive maps to real Canadian commerce and fintech workflows.

---

## 13. How I would integrate it as a builder

For a SaaS/API product, I would start with the Subscription Plan track:

1. Choose settlement mint, e.g. a stablecoin mint supported by the product.
2. Initialize merchant configuration and create one Plan PDA per public tier.
3. Publish plan metadata for frontends and wallets.
4. In the checkout flow, ask the user to initialize a Subscription Authority if needed.
5. Have the user subscribe to the selected plan.
6. Run a billing worker that calls `transferSubscription` for due accounts.
7. Listen to program events for successful pulls, cancellations, and failed attempts.
8. Mirror state into the app database for customer support, access control, and analytics.

For an AI-agent product, I would usually start with fixed or recurring delegations:

1. User chooses task budget and expiry.
2. App creates a fixed delegation for the agent or policy wallet.
3. Agent performs purchases or payments through `transferFixed`.
4. User sees remaining allowance and can revoke at any time.
5. For ongoing agents, switch to recurring delegation with a period cap.

The design question is: **who should control the terms?**

- If the user defines the budget, use fixed or recurring delegation.
- If the merchant publishes standard terms, use subscription plans.

---

## 14. What would make the ecosystem stronger

The program itself is only the base layer. The ecosystem now needs product-grade surrounding pieces:

- Wallet UI that explains Subscription Authorities and active delegations clearly.
- Indexers that normalize plans, subscriptions, renewals, cancellations, and failed transfers.
- Merchant dashboards for plan creation, customer status, retries, and revenue recognition.
- Open-source examples for stablecoin SaaS billing and agent budget flows.
- Wallet notification standards for upcoming renewals and failed pulls.
- Security dashboards showing program audit status, upgrade authority, and active integrations.

The winning products may not feel like “crypto subscriptions.” They may feel like normal SaaS checkout, normal API billing, or normal agent budgets — except settlement and authorization happen onchain.

---

## 15. Final view

Solana Subscriptions & Allowances is best understood as a shared authorization layer for pull-based token payments.

Its main contribution is architectural:

- one Subscription Authority per user/mint,
- many Delegation PDAs behind it,
- three payment models,
- immutable merchant plans,
- recurring-period accounting,
- Token-2022 compatibility,
- and a client surface that developers can integrate without rebuilding the whole stack.

The tradeoff is that a shared primitive needs excellent audits, wallet UX, indexing, and operational tooling. But if those pieces mature, this can become a foundational rail for stablecoin SaaS billing, recurring B2B payments, embedded-wallet subscriptions, payroll-like flows, and AI-agent budgets.

For builders, the practical takeaway is simple:

> Use subscription plans when a merchant publishes terms. Use fixed or recurring delegations when a user grants a bounded budget. In both cases, keep the authorization understandable, revocable, and observable.

---

## Sources

- Solana Foundation, “Three models, one program — Solana Subscriptions & Allowances”: https://solana.com/news/subscriptions-and-allowances
- GitHub repository, `solana-foundation/subscriptions`: https://github.com/solana-foundation/subscriptions
- Repository README and architecture docs inspected at commit `81a52ec`
- Program architecture docs: `docs/001-program-architecture.md`, `docs/002-subscriptions-architecture.md`
- The Defiant, “Solana Ships Native Payments Rail for Subscriptions and Allowances”: https://thedefiant.io/converge/blockchains/solana-ships-native-payments-rail-for-subscriptions-and-allowances
- Solana Compass, “Solana Subscriptions & Allowances Program Launches on Mainnet”: https://solanacompass.com/news/solana-launches-native-subscription-billing-and-recurring-payments-on-mainnet

*Disclosure: this writeup was produced by Leonardo Research Labs as an agent-assisted public technical artifact for an agent-eligible Superteam Canada bounty. It is informational only and not financial, investment, legal, or tax advice.*
