# Status

📜 Proposed 2023-07-22

# Context

## Background
 
In the cardano blockchain, available functionality is determined by the era (more precisely the protocol version). To capture this we defined the following types<sup>1, 2</sup>:
```haskell
data ShelleyBasedEra era where
  ShelleyBasedEraShelley :: ShelleyBasedEra ShelleyEra
  ShelleyBasedEraAllegra :: ShelleyBasedEra AllegraEra
  ShelleyBasedEraMary :: ShelleyBasedEra MaryEra
  ShelleyBasedEraAlonzo :: ShelleyBasedEra AlonzoEra
  ShelleyBasedEraBabbage :: ShelleyBasedEra BabbageEra
  ShelleyBasedEraConway :: ShelleyBasedEra ConwayEra

data CardanoEra era where
  ByronEra :: CardanoEra ByronEra
  ShelleyEra :: CardanoEra ShelleyEra
  AllegraEra :: CardanoEra AllegraEra
  MaryEra :: CardanoEra MaryEra
  AlonzoEra :: CardanoEra AlonzoEra
  BabbageEra :: CardanoEra BabbageEra
  ConwayEra :: CardanoEra ConwayEra

```
Early on the idea was to parameterize functions on the era so they can have the era specific behaviour but also to capture this idea of backwards compatibility. In the Shelley era days, there was an (incorrect) assumption that later eras would be backwards compatible with previous eras, especially where transactions were concerned. This would have allowed users to in theory never update their `cardano-cli` version however this is not the case as hardforks introduce backwards incompatible changes (see Conway breaking changes as an example).

The other justification for enumerating the eras like this was to allow QA to perform testing across all eras. However, the only test QA performs across the eras are hardforks. Therefore we don't need to expose all the era specific functionality for each era, rather we need to maintain the ability to submit hardfork update proposals across the eras i.e Byron -> latest era. 

Users consuming `cardano-api` are only interested in functionality relevant to mainnet and the upcoming era if they want to experiment with new features on a testnet. Therefore rather than maintaining a potentially never ending enumeration of eras, I propose to maintain only the era on mainnet and the upcoming era that will be hardforked to in the future.

## Code 

```haskell
data ConwayEra

data BabbageEra 

-- Let's us constrain the era possibilities to mainnet's era
-- and a potential upcoming era
type family ToConstrainedEra era = (r :: Type) | r -> era where
  ToConstrainedEra BabbageEra = Ledger.Babbage
  ToConstrainedEra ConwayEra = Ledger.Conway

-- Allows us to gradually change the api without breaking things.
-- This will eventually be removed.
type family AvailableErasToSbe era = (r :: Type) | r -> era where
  AvailableErasToSbe BabbageEra = Api.BabbageEra
  AvailableErasToSbe ConwayEra = Api.ConwayEra


data Era era where
  -- | The era currently active on Cardano's mainnet.
  CurrentEra ::  Era BabbageEra
  -- | The era planned for the next hardfork on Cardano's mainnet.
  UpcomingEra :: Era ConwayEra

-- | After a hardfork there is usually no planned upcoming era
-- that we are able to experiment with. We force a type era
-- in this instance. See docs above.
data EraCurrentlyNonExistent

type family UninhabitableType a where
  UninhabitableType EraCurrentlyNonExistent =
    TypeError ('Text "There is currently no planned upcoming era. Use CurrentEra constructor instead.")
```

### Deprecation as seen by users 

The following is a two stage example of how a user would consume the api and how a user would see a deprecation of an era

```haskell
-- TODO: These constraints can be hidden
makeUnsignedTx
  :: Ledger.EraCrypto (ToConstrainedEra era) ~ L.StandardCrypto
  => L.AlonzoEraTx (ToConstrainedEra era)
  => L.BabbageEraTxBody (ToConstrainedEra era)
  => ShelleyLedgerEra (AvailableErasToSbe era) ~ ToConstrainedEra era
  => Era era
  -> TxBodyContent BuildTx (AvailableErasToSbe era)
  -> Either UnsignedTxError (UnsignedTx era)
makeUnsignedTx era bc = ...


testBabbageUnsignedTx :: Either UnsignedTxError (UnsignedTx BabbageEra)
testBabbageUnsignedTx = makeUnsignedTx CurrentEra $ defaultTxBodyContent ShelleyBasedEraBabbage

testConwayUnsignedTx :: Either UnsignedTxError (UnsignedTx ConwayEra)
testConwayUnsignedTx = makeUnsignedTx UpcomingEra $ defaultTxBodyContent ShelleyBasedEraConway
```

In the current state of affairs both `testBabbageUnsignedTx` and `testConwayUnsignedTx` type check without issue. However let's update the types to show how a deprecation (or a successful hardfork to the upcoming era) will show up in a user's code.

Changes:

```haskell
{-# DEPRECATED BabbageEra "We are currently in the Conway era. Please update your type Signature to ConwayEra" #-}
data BabbageEra
...
...
data Era era where
  -- | The era currently active on Cardano's mainnet.
  CurrentEra ::  Era ConwayEra
  -- | The era planned for the next hardfork on Cardano's mainnet.
  UpcomingEra :: Era (UninhabitableType EraCurrentlyNonExistent)
```
Type errors:

```haskell
In the use of type constructor or class ‘BabbageEra’
(imported from Cardano.Api.Experimental.Eras, but defined in Cardano.Api.Protocol.AvailableEras):
Deprecated: "We are currently in the Conway era. Please update your type Signature to ConwayEra"

and

internal/Cardano/Api/Experimental/Tx.hs:34:39: error: [GHC-64725]
    • There is currently no planned upcoming era. Use CurrentEra instead.
    • In the first argument of ‘makeUnsignedTx’, namely ‘UpcomingEra’
      In the first argument of ‘($)’, namely ‘makeUnsignedTx UpcomingEra’
      In the expression:
        makeUnsignedTx UpcomingEra
          $ defaultTxBodyContent ShelleyBasedEraConway
   |
34 | testConwayUnsignedTx = makeUnsignedTx UpcomingEra $ defaultTxBodyContent ShelleyBasedEraConway
```

This deprecation will happen only after the hardfork has been successful. There should be no need for consumers of the api to need the prior era's specific functionality. If users for some reason want this, we can direct them to use the ledger's api directly.

## Downsides

Because we are using GADTs our library functions will have unreachable code paths that will compile, however trying to use the unreachable values will result in a type error. E.g

### Deprecation as seen by cardano-api maintainers

Before deprecation:

```haskell

testBabbageEra :: Either UnsignedTxError (Ledger.TxBody (ToConstrainedEra BabbageEra))
testBabbageEra = eraSpecificLedgerTxBody CurrentEra undefined $ defaultTxBodyContent ShelleyBasedEraBabbage

testConwayEra :: Either UnsignedTxError (Ledger.TxBody (ToConstrainedEra ConwayEra))
testConwayEra = eraSpecificLedgerTxBody UpcomingEra undefined $ defaultTxBodyContent ShelleyBasedEraConway


eraSpecificLedgerTxBody
  :: Era era
  -> Ledger.TxBody (ToConstrainedEra era)
  -> TxBodyContent BuildTx (AvailableErasToSbe era)
  -> Either UnsignedTxError (Ledger.TxBody (ToConstrainedEra era))
eraSpecificLedgerTxBody CurrentEra ledgerbody bc = do
  sbe <- maybe (Left $ error "eraSpecificLedgerTxBody: TODO") Right $ protocolVersionToSbe CurrentEra

  setUpdateProposal <- first UnsignedTxError $ convTxUpdateProposal sbe (txUpdateProposal bc)

  return $ ledgerbody  & L.updateTxBodyL .~ setUpdateProposal

eraSpecificLedgerTxBody UpcomingEra ledgerbody bc =
  let propProcedures = txProposalProcedures bc
      voteProcedures = txVotingProcedures bc
      treasuryDonation = txTreasuryDonation bc
      currentTresuryValue = txCurrentTreasuryValue bc
  in return $
       ledgerbody
          & L.proposalProceduresTxBodyL .~ convProposalProcedures (maybe TxProposalProceduresNone unFeatured propProcedures)
          & L.votingProceduresTxBodyL .~ convVotingProcedures (maybe TxVotingProceduresNone unFeatured voteProcedures)
          & L.treasuryDonationTxBodyL .~ maybe (L.Coin 0) unFeatured treasuryDonation
          & L.currentTreasuryValueTxBodyL .~ L.maybeToStrictMaybe (unFeatured <$> currentTresuryValue)
```
After deprecation:

```haskell
internal/Cardano/Api/Experimental/Tx.hs:123:26: error: [GHC-39999]
    • Could not deduce ‘L.ShelleyEraTxBody
                          Ouroboros.Consensus.Shelley.Eras.StandardConway’
        arising from a use of ‘L.updateTxBodyL’
      from the context: era ~ ConwayEra
        bound by a pattern with constructor: CurrentEra :: Era ConwayEra,
                 in an equation for ‘eraSpecificLedgerTxBody’
        at internal/Cardano/Api/Experimental/Tx.hs:118:25-34
    • In the first argument of ‘(.~)’, namely ‘L.updateTxBodyL’
      In the second argument of ‘(&)’, namely
        ‘L.updateTxBodyL .~ setUpdateProposal’
      In the second argument of ‘($)’, namely
        ‘ledgerbody & L.updateTxBodyL .~ setUpdateProposal’
    |
123 |   return $ ledgerbody  & L.updateTxBodyL .~ setUpdateProposal
    |                          ^^^^^^^^^^^^^^^

internal/Cardano/Api/Experimental/Tx.hs:132:13: error: [GHC-64725]
    • Cannot satisfy: Cardano.Ledger.Binary.Version.MinVersion <= Ledger.ProtVerHigh
                                                                    (ToConstrainedEra era)
    • In the first argument of ‘(.~)’, namely
        ‘L.proposalProceduresTxBodyL’
      In the second argument of ‘(&)’, namely
        ‘L.proposalProceduresTxBodyL
           .~
             convProposalProcedures
               (maybe TxProposalProceduresNone unFeatured propProcedures)’
      In the first argument of ‘(&)’, namely
        ‘ledgerbody
           & L.proposalProceduresTxBodyL
               .~
                 convProposalProcedures
                   (maybe TxProposalProceduresNone unFeatured propProcedures)’
    |
132 |           & L.proposalProceduresTxBodyL .~ convProposalProcedures (maybe TxProposalProceduresNone unFeatured propProcedures)
    |             ^^^^^^^^^^^^^^^^^^^^^^^^^^^

internal/Cardano/Api/Experimental/Tx.hs:132:13: error: [GHC-64725]
    • Cannot satisfy: Cardano.Ledger.Binary.Version.MinVersion <= Ledger.ProtVerLow
                                                                    (ToConstrainedEra era)
    • In the first argument of ‘(.~)’, namely
        ‘L.proposalProceduresTxBodyL’
      In the second argument of ‘(&)’, namely
        ‘L.proposalProceduresTxBodyL
           .~
             convProposalProcedures
               (maybe TxProposalProceduresNone unFeatured propProcedures)’
      In the first argument of ‘(&)’, namely
        ‘ledgerbody
           & L.proposalProceduresTxBodyL
               .~
                 convProposalProcedures
                   (maybe TxProposalProceduresNone unFeatured propProcedures)’
    |
132 |           & L.proposalProceduresTxBodyL .~ convProposalProcedures (maybe TxProposalProceduresNone unFeatured propProcedures)
    |             ^^^^^^^^^^^^^^^^^^^^^^^^^^^

internal/Cardano/Api/Experimental/Tx.hs:132:13: error: [GHC-64725]
    • Cannot satisfy: Ledger.ProtVerLow
                        (ToConstrainedEra era) <= Ledger.ProtVerHigh (ToConstrainedEra era)
    • In the first argument of ‘(.~)’, namely
        ‘L.proposalProceduresTxBodyL’
      In the second argument of ‘(&)’, namely
        ‘L.proposalProceduresTxBodyL
           .~
             convProposalProcedures
               (maybe TxProposalProceduresNone unFeatured propProcedures)’
      In the first argument of ‘(&)’, namely
        ‘ledgerbody
           & L.proposalProceduresTxBodyL
               .~
                 convProposalProcedures
                   (maybe TxProposalProceduresNone unFeatured propProcedures)’
    |
132 |           & L.proposalProceduresTxBodyL .~ convProposalProcedures (maybe TxProposalProceduresNone unFeatured propProcedures)
```

We get somewhat obscure type errors but this should not be an issue as this is a direct result of deprecating an era in cardano-api. We know we have to update our functions because of this, so it comes as no surprise

However downstream users get better type eras:

```haskell
internal/Cardano/Api/Experimental/Tx.hs:125:41: error: [GHC-64725]
    • There is currently no planned upcoming era. Use CurrentEra instead.
    • In the first argument of ‘eraSpecificLedgerTxBody’, namely
        ‘UpcomingEra’
      In the first argument of ‘($)’, namely
        ‘eraSpecificLedgerTxBody UpcomingEra undefined’
      In the expression:
        eraSpecificLedgerTxBody UpcomingEra undefined
          $ defaultTxBodyContent ShelleyBasedEraConway
    |
125 | testConwayEra = eraSpecificLedgerTxBody UpcomingEra undefined $ defaultTxBodyContent ShelleyBasedEraConway
    |                              
```
and for `testBabbageEra`:
```
internal/Cardano/Api/Experimental/Tx.hs:122:18: error: [GHC-83865]
    • Couldn't match type: L.ConwayTxBody
                             (L.ConwayEra L.StandardCrypto)
                     with: Cardano.Ledger.Babbage.TxBody.BabbageTxBody
                             (L.BabbageEra L.StandardCrypto)
      Expected: Either
                  UnsignedTxError (Ledger.TxBody (ToConstrainedEra BabbageEra))
        Actual: Either
                  UnsignedTxError (Ledger.TxBody (ToConstrainedEra ConwayEra))
    • In the expression:
        eraSpecificLedgerTxBody CurrentEra undefined
          $ defaultTxBodyContent ShelleyBasedEraBabbage
      In an equation for ‘testBabbageEra’:
          testBabbageEra
            = eraSpecificLedgerTxBody CurrentEra undefined
                $ defaultTxBodyContent ShelleyBasedEraBabbage
    |
122 | testBabbageEra = eraSpecificLedgerTxBody CurrentEra undefined $ defaultTxBodyContent ShelleyBasedEraBabbage
    |                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

internal/Cardano/Api/Experimental/Tx.hs:122:65: error: [GHC-83865]
    • Couldn't match type ‘Cardano.Api.Eras.Core.BabbageEra’
                     with ‘Cardano.Api.Eras.Core.ConwayEra’
      Expected: TxBodyContent BuildTx (AvailableErasToSbe ConwayEra)
        Actual: TxBodyContent BuildTx Cardano.Api.Eras.Core.BabbageEra
    • In the second argument of ‘($)’, namely
        ‘defaultTxBodyContent ShelleyBasedEraBabbage’
      In the expression:
        eraSpecificLedgerTxBody CurrentEra undefined
          $ defaultTxBodyContent ShelleyBasedEraBabbage
      In an equation for ‘testBabbageEra’:
          testBabbageEra
            = eraSpecificLedgerTxBody CurrentEra undefined
                $ defaultTxBodyContent ShelleyBasedEraBabbage
    |
122 | testBabbageEra = eraSpecificLedgerTxBody CurrentEra undefined $ defaultTxBodyContent ShelleyBasedEraBabbage
```
### GADT flexibility

The flexibility of GADTs also bites us as just by looking at the code it appears the deprecated code path is reachable, but it is not:

```haskell
eraSpecificLedgerTxBody
  :: Era era
  -> Ledger.TxBody (ToConstrainedEra era)
  -> TxBodyContent BuildTx (AvailableErasToSbe era)
  -> Either UnsignedTxError (Ledger.TxBody (ToConstrainedEra era))
eraSpecificLedgerTxBody CurrentEra ledgerbody bc =
  let propProcedures = txProposalProcedures bc
      voteProcedures = txVotingProcedures bc
      treasuryDonation = txTreasuryDonation bc
      currentTresuryValue = txCurrentTreasuryValue bc
  in return $
       ledgerbody
          & L.proposalProceduresTxBodyL .~ convProposalProcedures (maybe TxProposalProceduresNone unFeatured propProcedures)
          & L.votingProceduresTxBodyL .~ convVotingProcedures (maybe TxVotingProceduresNone unFeatured voteProcedures)
          & L.treasuryDonationTxBodyL .~ maybe (L.Coin 0) unFeatured treasuryDonation
          & L.currentTreasuryValueTxBodyL .~ L.maybeToStrictMaybe (unFeatured <$> currentTresuryValue)

eraSpecificLedgerTxBody UpcomingEra ledgerbody _bc = return ledgerbody
--  sbe <- maybe (Left $ error "eraSpecificLedgerTxBody: TODO") Right $ protocolVersionToSbe CurrentEra
--
--  setUpdateProposal <- first UnsignedTxError $ convTxUpdateProposal sbe (txUpdateProposal bc)
--
--  return $ ledgerbody  & L.updateTxBodyL .~ setUpdateProposal
```

Type error when called: 

```haskell
ghci> eraSpecificLedgerTxBody UpcomingEra undefined undefined

<interactive>:9:1: error: [GHC-64725]
    • There is currently no planned upcoming era. Use CurrentEra instead.
    • In a stmt of an interactive GHCi command: print it
```

# Approach

The new api should be created adjacant to the existing one. We then slowly replace the use of the existing api in cardano-api, eventually deprecating the "old" api. 

With respect to cardano-cli, we should introduce top level `current-era` and `upcoming-era` commands. We would have to release a cli version specific for post-hardfork usage, as `current-era` and `upcoming-era` commands will have different semantics to the pre-hardfork commands. This will require a little coordination but shouldn't be too much additional overhead. 
# Decision

TBD

# Consequences

- Less boiler plate in cardano-api and in users' codebases 
- Clearer defined boundaries as to new functionality being introduced due to a new era
- Lower maintenance overhead 
- The need for an era agnostic api with respect to update proposals (these change on the Babbage/Conway boundary)

# References
- 1: [ShelleyBasedEra](https://github.com/IntersectMBO/cardano-api/blob/873397bfe0436c224c593f456a3bc237ee0af0c8/cardano-api/internal/Cardano/Api/Eon/ShelleyBasedEra.hs#L123)
- 2: [CardanoEra](https://github.com/IntersectMBO/cardano-api/blob/873397bfe0436c224c593f456a3bc237ee0af0c8/cardano-api/internal/Cardano/Api/Eras/Core.hs#L256)
