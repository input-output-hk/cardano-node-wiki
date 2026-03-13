
## ADR Relationship Graph

```mermaid
graph LR
  ADR000(ADR-000)
  ADR001(ADR-001)
  ADR002(ADR-002)
  ADR003(ADR-003)
  ADR004(ADR-004)
  ADR005(ADR-005)
  ADR006(ADR-006)
  ADR007(ADR-007)
  ADR008(ADR-008)
  ADR009(ADR-009)
  ADR010(ADR-010)
  ADR011(ADR-011)
  ADR012(ADR-012)
  ADR013(ADR-013)
  ADR014(ADR-014)
  ADR015(ADR-015)
  ADR016(ADR-016)
  ADR018(ADR-018)

  ADR001 <--> ADR004
  ADR001 <--> ADR012
  ADR002 <--> ADR009
  ADR003 <--> ADR006
  ADR004 <--> ADR009
  ADR004 <--> ADR010
  ADR004 <--> ADR015
  ADR004 <--> ADR016
  ADR005 <--> ADR007
  ADR006 <--> ADR012
  ADR006 <--> ADR013
  ADR007 <--> ADR008
  ADR008 <--> ADR011
  ADR009 <--> ADR010
  ADR009 <--> ADR014
  ADR010 <--> ADR014
  ADR010 <--> ADR016
  ADR012 <--> ADR013
  ADR014 <--> ADR015
  ADR015 <--> ADR018

  classDef adopted fill:#238636,stroke:#3fb950,color:#fff
  classDef proposed fill:#9e6a03,stroke:#d29922,color:#fff
  classDef rejected fill:#da3633,stroke:#f85149,color:#fff

  class ADR000,ADR001,ADR002,ADR004,ADR007,ADR008,ADR009,ADR010,ADR011,ADR014,ADR015 adopted
  class ADR003,ADR005,ADR012,ADR013,ADR016,ADR018 proposed
  class ADR006 rejected
```

## ADR Index

* ✅ [[ADR-0 Documenting Architecture Decisions]]
* ✅ [[ADR-1 Default eras for CLI commands]]
* ✅ [[ADR-2 Module structure for generators]]
* 📜 [[ADR-3 Dependencies version constraints in cabal file]]
* ✅ [[ADR-4 Support-only-for-mainnet-and-upcoming-eras]]
* 📜 [[ADR-5 cardano-testnet-node-configuration-file]]
* ❌ [[ADR-6 Using optparse-applicative main repository]]
* ✅ [[ADR-7 CLI Output Presentation]]
* ✅ [[ADR-8 Use RIO in cardano-cli]]
* ✅ [[ADR-9 cardano-api exports convention]]
* ✅ [[ADR-10 cardano-api script witness API]]
* ✅ [[ADR-11 Better call stacks of IO exceptions]]
* 📜 [[ADR-12 Standardise CLI multiple choice flags construction]]
* 📜 [[ADR-13 Metavars must follow screaming snake case]]
* ✅ [[ADR-14 Total conversion functions conventions]]
* ✅ [[ADR-15 JavaScript API for Cardano via WASM-compiled cardano-api]]
* 📜 [[ADR-16 cardano-api new TxBodyContent]]
* 📜 [[ADR-18 gRPC Server for Cardano Node (cardano-rpc)]]

## Legend

* 📜 Proposed
* ✅ Adopted
* ❌ Rejected
* 🗑️ Deprecated
* ⬆️ Superseded

