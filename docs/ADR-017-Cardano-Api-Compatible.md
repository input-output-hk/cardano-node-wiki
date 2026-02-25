# ADR-017: Cardano.Api.Compatible — Cross-Era Transaction and Certificate Construction

## Status

Accepted

## Context

The `cardano-api` library provides two main API layers
for constructing transactions and certificates:

1. **Legacy API** (`Cardano.Api.Tx`, `Cardano.Api.Certificate.Internal`) —
   The original transaction construction interface.
   It covers all Shelley-based eras and supports the full feature set
   including governance operations
   (DRep registration, committee certificates, proposal procedures).
   However, it uses era-specific GADT constructors
   (e.g., `StakeAddressRequirements era`, `DRepRegistrationRequirements era`)
   that carry era witnesses at construction time.
   As the number of Cardano eras has grown,
   maintaining correct GADT coverage across all eras
   has become increasingly difficult,
   and several functions do not compile for all Shelley-based eras.

2. **Experimental API** (`Cardano.Api.Experimental.Tx`,
   `Cardano.Api.Experimental.Tx.Internal.Certificate`) —
   A newer API built around the `IsEra` type class/ `Era` data type,
   which is intentionally restricted to only
   the **current mainnet era and the upcoming era**
   (currently Conway and Dijkstra).
   After a hardfork, the previous era is deprecated and eventually removed.
   This forward-looking design avoids the combinatorial complexity
   of supporting all historical eras
   but means Experimental functions cannot be used for pre-Conway eras.

Neither API fully satisfies the need for
**simple, uniform transaction and certificate construction
across all Shelley-based eras** (Shelley through Conway and beyond).
The Legacy API covers all eras
but its GADT-heavy design makes writing truly era-polymorphic code difficult.
The Experimental API has a cleaner design
but intentionally excludes historical eras.

The Compatible API was introduced specifically to support QA's
hardfork traversal tests, which need to construct valid transactions
in every Shelley-based era in order to hardfork
across all boundaries within a single test run.

A key challenge is that hardfork boundaries
introduce **breaking semantic changes**:

- **Stake registration**:
  Pre-Conway eras require only a `StakeCredential`;
  Conway and later require a `StakeCredential` *and* a deposit (`Coin`).
- **Delegation targets**:
  Pre-Conway eras delegate to stake pool key hashes (`Hash StakePoolKey`);
  Conway and later support richer delegation targets
  including DReps (`Ledger.Delegatee`).
- **Protocol governance**:
  Pre-Conway eras use `UpdateProposal` for parameter changes;
  Conway and later use `ProposalProcedures` and `VotingProcedures`.

These differences need to be captured in a type-safe way
without burdening callers with era-specific logic at every use site.

## Decision

We introduce the `Cardano.Api.Compatible` module family
as a **bridge layer** that provides uniform transaction
and certificate construction across all Shelley-based eras.
The Compatible API is organised into two sub-modules:

### Module Structure

- **`Cardano.Api.Compatible`** — Top-level re-export module.
- **`Cardano.Api.Compatible.Tx`** — Transaction construction functions.
- **`Cardano.Api.Compatible.Certificate`** —
  Certificate construction functions
  (re-exports from `Cardano.Api.Experimental.Tx.Internal.Certificate.Compatible`).

The implementation of certificate functions lives under
`Cardano.Api.Experimental.Tx.Internal.Certificate.Compatible`
because these functions build on the Experimental API's
internal certificate types.
`Cardano.Api.Compatible.Certificate` re-exports them
for discoverability under the `Compatible` namespace.

### Handling Hardfork Boundary Differences with Type Families

Type families capture era-dependent semantics at the type level:

```haskell
-- Delegation targets differ across the Conway boundary
type family Delegatee era where
  Delegatee ConwayEra   = Ledger.Delegatee        -- DRep/pool/abstain
  Delegatee BabbageEra  = Hash StakePoolKey        -- Pool only
  Delegatee AlonzoEra   = Hash StakePoolKey
  -- ... (all pre-Conway eras map to Hash StakePoolKey)

-- Stake registration requirements differ across the Conway boundary
type family StakeRegistrationRequirements era where
  StakeRegistrationRequirements ConwayEra   = StakeCredentialAndDeposit
  StakeRegistrationRequirements BabbageEra  = StakeCredential
  -- ... (all pre-Conway eras map to StakeCredential)
```

This design means callers provide the correct data for each era
without needing to pattern-match on era witnesses themselves.
The `IsShelleyBasedEra` constraint on the certificate functions
dispatches to the correct ledger operations internally.

### Handling Governance Differences with GADTs

For transaction-level governance features
that exist only in certain eras,
GADTs provide runtime era witnesses:

```haskell
data AnyProtocolUpdate era where
  ProtocolUpdate
    :: ShelleyToBabbageEra era
    -> UpdateProposal
    -> AnyProtocolUpdate era
  ProposalProcedures
    :: ConwayEraOnwards era
    -> TxProposalProcedures (...)
    -> AnyProtocolUpdate era
  NoPParamsUpdate
    :: ShelleyBasedEra era
    -> AnyProtocolUpdate era

data AnyVote era where
  VotingProcedures
    :: ConwayEraOnwards era
    -> TxVotingProcedures (...)
    -> AnyVote era
  NoVotes :: AnyVote era
```

### Core Functions

**Transaction construction:**

- `createCompatibleTx` —
  Constructs an unbalanced transaction in any Shelley-based era,
  given inputs, outputs, fees, protocol updates, votes, and certificates.
  Returns `Either ProtocolParametersConversionError (Tx era)`.
- `addWitnesses` —
  Adds key witnesses (both Shelley address and Byron bootstrap)
  to a transaction post-construction.

**Certificate construction:**

- `makeStakeAddressDelegationCertificate` —
  Era-polymorphic stake delegation
  (pool key pre-Conway, `Delegatee` post-Conway).
- `makeStakeAddressRegistrationCertificate` —
  Era-polymorphic stake registration
  (no deposit pre-Conway, deposit required post-Conway).
- `makeStakeAddressUnregistrationCertificate` —
  Stake deregistration across all eras.
- `makeStakePoolRegistrationCertificate` /
  `makeStakePoolRetirementCertificate` —
  Pool lifecycle certificates.
- `makeMIRCertificate` /
  `makeGenesisKeyDelegationCertificate` —
  QA-only certificates hardcoded to BabbageEra
  (these certificate types do not exist post-Conway).
- `selectStakeCredentialWitness` / `getTxCertWitness` —
  Extract the stake credential that must witness a given certificate.

### Relationship to Other APIs

- **Built on the Experimental API** —
  Compatible uses the Experimental API's internal types
  (`TxCertificates`, `TxProposalProcedures`,
  `TxVotingProcedures`, `AnyWitness`, `Certificate`)
  rather than duplicating ledger integration.
- **Deprecates Legacy API functions** —
  Legacy certificate functions in `Cardano.Api.Certificate.Internal`
  carry `DEPRECATED` pragmas directing users
  to the Compatible equivalents for cross-era support,
  or to the Experimental equivalents for era-specific use.
- **Not re-exported from `Cardano.Api`** —
  The Compatible module is deliberately not exposed
  through the main API entry point,
  keeping it clearly scoped as a testing/internal API.

## Consequences

### Positive

- **Uniform cross-era coverage**:
  A single set of functions works across all Shelley-based eras,
  eliminating the need for era-specific branching
  at call sites in tests and QA tooling.
- **Type-safe hardfork boundaries**:
  Type families make era-dependent differences explicit at the type level.
  Callers cannot accidentally pass a `StakeCredential`
  where a `StakeCredentialAndDeposit` is required in Conway.
- **Clear deprecation path**:
  Legacy API functions now have a concrete migration target,
  reducing API surface over time.
- **Layered on Experimental**:
  By building on the Experimental API's internals,
  Compatible avoids duplicating ledger integration logic
  and benefits from improvements to the Experimental layer.

### Negative

- **Third API surface**:
  Maintaining three API layers (Legacy, Experimental, Compatible)
  increases the overall API surface area
  and cognitive load for contributors.
- **Testing-scoped by design**:
  The Compatible API is intentionally not exposed via `Cardano.Api`,
  meaning downstream consumers must import
  `Cardano.Api.Compatible` directly if they want to use it.
  This is a deliberate trade-off to keep the main API surface clean.
- **Partial coverage of governance features**:
  The Compatible API focuses on common transaction
  and certificate operations.
  Advanced governance operations
  (DRep registration, committee certificates)
  remain exclusive to the Experimental API.
