# Design Proposal

## High-Level Design

### Solution 1: Introduce a Dedicated On-chain Price Service **[PREFERRED]**

With on-chain pricing supporting multiple price sources, the system needs to retrieve additional pricing data and transaction references from existing indexers such as the **Minswap Indexer**, **Liqwid Indexer**, **Orcfax Indexer**, and other future integrations.

To make the architecture easier to extend and to avoid coupling the Fixed Lending BFF directly with multiple pricing sources, we propose introducing an **On-chain Price Service**.

The On-chain Price Service acts as a unified aggregation layer responsible for:

- Retrieving price information from multiple indexers.
- Retrieving the UTxO outRefs required for transaction construction.
- Normalizing price-source responses behind a common interface.
- Centralizing configuration related to supported price sources.
- Providing a single integration point for services that require on-chain pricing information.
- Supporting future price-source integrations with minimal changes to downstream services.

The high-level flow is:

```text
Cardano Network
      |
      +-- Orcfax Transactions
      |       |
      |       v
      |   Orcfax Indexer
      |       |
      |       v
      |   Orcfax Consumer
      |
      +-- Liqwid Transactions
      |       |
      |       v
      |   Liqwid Indexer
      |       |
      |       v
      |   Liqwid Consumer
      |
      +-- Minswap Transactions
      |       |
      |       v
      |   Minswap Indexer
      |       |
      |       v
      |   Minswap Consumer
      |
      +-- Other supported sources
              |
              v
       On-chain Price Service
              |
              v
       Fixed Lending BFF
              |
              v
       Fixed Lending Client / FE
```

Depending on the selected price source, the service can retrieve information such as:

- Orcfax price and outRefs of FSP / FS.
- Liqwid price and outRefs of Market State / Oracle.
- Minswap price and outRefs of LP UTxOs.
- Additional price information from future integrations.

---

### Solution 2: Query Individual Indexers Directly

In this solution, the Fixed Lending BFF communicates directly with each indexer to retrieve pricing information.

Each indexer owns the source-specific indexing and price calculation logic. Because different sources have different data structures and mechanisms for determining prices, the BFF calls the corresponding indexer through its source-specific interface.

The high-level flow is:

```text
                         +-- Orcfax Indexer
                         |
Fixed Lending BFF -------+-- Liqwid Indexer
                         |
                         +-- Minswap Indexer
                         |
                         +-- Other Indexers
```

Each indexer remains responsible for interpreting its indexed data and exposing the appropriate pricing information.

---

## Solution Comparison

| Area | Solution 1 – On-chain Price Service **[PREFERRED]** | Solution 2 – Direct Indexer Integration |
| --- | --- | --- |
| Integration model | A dedicated service aggregates pricing information from all supported indexers | Fixed Lending BFF integrates directly with each indexer |
| Consumer interface | Downstream services use a single interface | Different price sources may require different interfaces |
| Configuration | Price-source configuration is centralized | Configuration and integration logic may be distributed across consumers |
| Extensibility | New price sources can be added with minimal impact on consumers | Each new source may require additional BFF integration |
| Coupling | Fixed Lending BFF is decoupled from individual source implementations | Fixed Lending BFF is coupled to individual indexers |
| Source-specific logic | Encapsulated by indexers and coordinated through the aggregation service | BFF must understand how to communicate with each indexer |

### Solution 1 – Advantages

- The **On-chain Price Service** provides a centralized aggregation point for all supported on-chain price sources.
- Other services only need to integrate with one service to retrieve pricing information and transaction-related outRefs.
- Price-source configuration is centralized in one place.
- Adding or replacing a price source has limited impact on downstream consumers.
- Source-specific implementation details are hidden from the Fixed Lending BFF.
- The architecture is better suited for supporting additional oracle and liquidity-based pricing sources in the future.

### Solution 2 – Considerations

- Each indexer already understands the logic and data structure of its corresponding price source.
- The BFF can retrieve price information directly through source-specific indexer APIs.
- However, the BFF must integrate with multiple indexers independently.
- Adding new price sources increases the number of dependencies and integration paths maintained by the BFF.

> Different indexers may use different indexing models and price validation mechanisms. This makes it difficult to define a fully generic pricing interface without preserving some source-specific logic.
>
> Detailed analysis: https://confluence.teko.vn/x/USBQGw

---

## Operational Flow

Based on the proposed HLD, integrating additional price sources mainly affects the **price retrieval flow** of the Fixed Lending BFF.

The existing flows between the BFF and the Fixed Lending Indexer for retrieving protocol information, calculating display data, and building transactions remain unchanged.

Therefore:

- Existing Fixed Lending BFF APIs remain unchanged where possible.
- Existing Fixed Lending Indexer APIs remain unchanged where possible.
- Legacy price-retrieval APIs will be deprecated.
- New APIs will be introduced specifically for retrieving prices from multiple sources.
- The Fixed Lending BFF will retrieve normalized pricing information through the On-chain Price Service.

Updated flow diagram:

https://confluence.teko.vn/download/attachments/458227789/image2025-1-18_10-23-48.png?version=1&modificationDate=1737170629175&api=v2

---

## API

Existing APIs provided by the **Fixed Lending BFF** and **Fixed Lending Indexer** will be reused where possible.

Legacy price retrieval APIs will be deprecated and replaced by the following APIs.

### On-chain Price Service

#### `POST /api/v1/prices`

Retrieves pricing information for token pairs from one or more supported price sources.

The service is responsible for selecting or aggregating the appropriate underlying source and returning the required pricing information and transaction references.

---

### Orcfax Indexer

#### `POST /api/v1/orcfax/prices`

Retrieves oracle prices from indexed Orcfax data.

---

### Liqwid Indexer

#### `POST /api/v1/liqwid/prices`

Retrieves oracle prices from indexed Liqwid data.

#### `POST /api/v1/liqwid/qtoken-prices`

Retrieves qToken prices from indexed Liqwid data.

---

###  Float Indexer

#### `POST /api/v1/dtoken-prices`

Retrieves dToken prices from indexed Float data.

---

## ERD

Based on the updated Fixed Lending datum structure, the **Fixed Lending Indexer database schema** needs to be updated to index all required protocol information.

ERD:

https://dbdiagram.io/d/CNIO-5074-Multiple-Oracle-sources-67873ef06b7fa355c3e9b541

ERD diagram:

https://confluence.teko.vn/download/attachments/458227789/image2025-1-20_14-21-25.png?version=2&modificationDate=1742360456942&api=v2

### Database Changes

#### `protocol_script_utxos`

Add:

- `oracle_script_hash`

#### `protocol_config_tokens`

Remove:

- `liqwid_qtoken`
- `liqwid_market_param_pid`
- `liqwid_market_state_pid`
- `compound_rate_idx`
- `interest_rate_idx`

Add:

- `reference_borrow_rate_type`
- `borrow_rate_type`
- `yield_tokens`

#### Removed Tables

The following tables are no longer required:

- `protocol_config_oracle_prices`
- `protocol_config_oracle_feeds`
