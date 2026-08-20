# On-chain Price Service

This repository contains the design proposal for introducing an **On-chain Price Service** to support multiple on-chain pricing sources.

The proposed architecture centralizes price retrieval and transaction-reference lookup across multiple indexers, including:

- Orcfax
- Liqwid
- Minswap
- Future oracle and liquidity-based price sources

## Why This Service


The preferred solution introduces a dedicated **On-chain Price Service** that:

- Provides a single pricing interface for downstream services.
- Aggregates data from multiple source-specific indexers.
- Returns pricing information and UTxO outRefs required for transaction building.
- Centralizes price-source configuration.

## Architecture

```text
Orcfax Indexer ----+
Liqwid Indexer ----+
Minswap Indexer ---+--> On-chain Price Service --> Fixed Lending BFF --> Client / FE
Other Indexers ----+
```

Each indexer remains responsible for indexing and interpreting its own source-specific on-chain data.

The On-chain Price Service provides the aggregation layer consumed by Fixed Lending and other services.

## Main API

```http
POST /api/v1/prices
```

Returns pricing information for supported token pairs from one or more configured price sources.

Source-specific indexer APIs include:

```http
POST /api/v1/orcfax/prices
POST /api/v1/liqwid/prices
POST /api/v1/liqwid/qtoken-prices
POST /api/v1/dtoken-prices
```

## Documentation

See the full technical proposal:

[Design Proposal](./design-proposal.md)

The proposal covers:

- High-level architecture
- Alternative solution comparison
- Operational flow
- API changes
- Database and ERD changes

## Preferred Approach

**Solution 1: Dedicated On-chain Price Service**

This approach is preferred because it reduces coupling between Fixed Lending and individual price sources, centralizes configuration, and provides a cleaner path for integrating additional price sources in the future.
