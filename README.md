# Burbit

**An order-book prediction market for bonding-curve tokens on Solana.**

Burbit runs YES/NO markets on the lifecycle of freshly launched bonding-curve tokens: graduation, curve milestones, creator behavior, post-graduation crashes, races, and launchpad-wide aggregates. Trading happens on a real central limit order book with fully-collateralized $1.00 share pairs, and markets settle trustlessly by reading the launchpad's own on-chain accounts.

Burbit builds no bonding curve and no AMM and is not a launchpad. It is a read-only prediction layer that can integrate **any** bonding-curve launchpad: supporting a new venue means adding one strict, read-only account parser, and nothing else changes.

## Documentation

**Start with the [full product walkthrough](docs/WALKTHROUGH.md)**: a plain-language, end-to-end tour of what people trade, where the money sits at every moment, how the YES/NO books work, who creates the first orders, and what happens to every participant when a market resolves. Developers then read the [PRD](docs/PRD.md), the complete build document. The formal specifications follow:

| Area | File |
| --- | --- |
| **Product walkthrough (start here)** | [WALKTHROUGH.md](docs/WALKTHROUGH.md) |
| **Developer PRD (build document)** | [PRD.md](docs/PRD.md) |
| Overview and glossary | [00-OVERVIEW.md](docs/00-OVERVIEW.md) |
| System architecture | [01-ARCHITECTURE.md](docs/01-ARCHITECTURE.md) |
| Core concepts and invariants | [02-CORE-CONCEPTS.md](docs/02-CORE-CONCEPTS.md) |
| Order book specification | [03-ORDER-BOOK-SPEC.md](docs/03-ORDER-BOOK-SPEC.md) |
| Market lifecycle | [04-MARKET-LIFECYCLE.md](docs/04-MARKET-LIFECYCLE.md) |
| Markets and settlement | [05-MARKETS-AND-SETTLEMENT.md](docs/05-MARKETS-AND-SETTLEMENT.md) |
| On-chain program spec | [06-ONCHAIN-PROGRAM-SPEC.md](docs/06-ONCHAIN-PROGRAM-SPEC.md) |
| Keepers | [07-KEEPERS.md](docs/07-KEEPERS.md) |
| Indexer and API spec | [08-INDEXER-AND-API-SPEC.md](docs/08-INDEXER-AND-API-SPEC.md) |
| Fees and economics | [09-FEES-AND-ECONOMICS.md](docs/09-FEES-AND-ECONOMICS.md) |
| Frontend spec | [10-FRONTEND-SPEC.md](docs/10-FRONTEND-SPEC.md) |
| Security | [11-SECURITY.md](docs/11-SECURITY.md) |
| Testing | [12-TESTING.md](docs/12-TESTING.md) |
| Roadmap | [13-ROADMAP.md](docs/13-ROADMAP.md) |
| Bonding-curve behavior study | [14-BONDING-CURVE-BEHAVIOR.md](docs/14-BONDING-CURVE-BEHAVIOR.md) |
| Question catalog | [15-QUESTION-CATALOG.md](docs/15-QUESTION-CATALOG.md) |
