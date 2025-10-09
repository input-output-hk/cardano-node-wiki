
# ADR-016: Backwards and Forwards Compatibility

**Status:** Proposed

**Date:** 2025-10-07

## Context

We are working on [a new API](./ADR-004-Support-only-for-mainnet-and-upcoming-eras.md) for `cardano-api` that is centred around the `Era era` type. We would like to maintain backwards compatibility with the "old" api under two conditions:

1) Existing exposed functionality that uses the "old" api which may be in use.
2) Any functionality needed to enable QA's ability to test hardforking from Byron to the current era.

## Proposed Solution

The solution proposed below is specific to certificates but can be expanded to any era depdendent type.

Our "old" Certificate definition is as follows:

```haskell
data Certificate era where
  -- Pre-Conway
  ShelleyRelatedCertificate
    :: Typeable era
    => ShelleyToBabbageEra era
    -> Ledger.ShelleyTxCert (ShelleyLedgerEra era)
    -> Certificate era
  -- Conway onwards
  ConwayCertificate
    :: Typeable era
    => ConwayEraOnwards era
    -> Ledger.ConwayTxCert (ShelleyLedgerEra era)
    -> Certificate era
``` 

`Cardano.Api.Certificate` exposes several helper functions that use this "old" type. E.g `makeStakeAddressDelegationCertificate`, `makeStakeAddressRegistrationCertificate` and `makeStakeAddressUnregistrationCertificate` to name a few. All of these "old" functions will be deprecated in favour of a new API that allows us to expose functionality from any era required without adding significant boilerplate as seen in the above definition of `data Certificate era`. We achieve this by the following:


Replace era-variant GADTs (like Certificate era) with a uniform wrapper (Certificate :: L.TxCert era -> Certificate era) and using open type families to resolve era-specific types (e.g., Delegatee era). Era constraints (IsEra, IsShelleyBasedEra) gate access to functionality.

I will make this clearer with an example below.

**NB**: We will only implement functionality constrained by `IsShelleyBasedEra era` and `IsEra era` that either is currently exposed by the "old" api or specifically required by QA. If we do not need backwards compatibility then we will use`Era era` or `IsEra era` without implementing an additional type synonym family.


Let's use `makeStakeAddressDelegationCertificate` as an example.

```haskell
data StakeDelegationRequirements era where
  StakeDelegationRequirementsConwayOnwards
    :: ConwayEraOnwards era
    -> StakeCredential
    -> Ledger.Delegatee
    -> StakeDelegationRequirements era
  StakeDelegationRequirementsPreConway
    :: ShelleyToBabbageEra era
    -> StakeCredential
    -> PoolId
    -> StakeDelegationRequirements era

makeStakeAddressDelegationCertificate :: StakeDelegationRequirements era -> Certificate era
makeStakeAddressDelegationCertificate = \case
  StakeDelegationRequirementsConwayOnwards cOnwards scred delegatee ->
    conwayEraOnwardsConstraints cOnwards $
      ConwayCertificate cOnwards $
        Ledger.mkDelegTxCert (toShelleyStakeCredential scred) delegatee
  StakeDelegationRequirementsPreConway atMostBabbage scred pid ->
    shelleyToBabbageEraConstraints atMostBabbage $
      ShelleyRelatedCertificate atMostBabbage $
        Ledger.mkDelegStakeTxCert (toShelleyStakeCredential scred) (unStakePoolKeyHash pid)

-- The `StakeDelegationRequirements era` GADT will necessitate more data constructors if there are any changes to the requirements of stake delegation in future eras. 
-- This is already seen in the transition from Babbage (can only delegate to stake pools) -> Conway (can delegate to stake pools and dreps). 
-- This obviously does not scale in the face of possible incoming changes and requires the propagation of an additional data constructor.

-- I propose the following:

-- Already exists in Cardano.Api.Experimental.Tx.Internal.Certificate
data Certificate era where
  Certificate :: L.EraTxCert era => L.TxCert era -> Certificate era

-- Type family definition capturing the change of possible delegatees. We can now delegate to stake pools or dreps or delegate
-- to dreps and stake pools simultaneously. This is captured in the `Ledger.Delegatee` type. 
type family Delegatee era where
  Delegatee ConwayEra = Ledger.Delegatee
  Delegatee BabbageEra = Api.Hash Api.StakePoolKey
  Delegatee AlonzoEra = Api.Hash Api.StakePoolKey 
  ... 
  ...
  Delegate ShelleyEra = Api.Hash Api.StakePoolKey

makeStakeAddressDelegationCertificate
  :: forall era
   . IsEra era
  => StakeCredential
  -> Delegatee era
  -> Certificate (LedgerEra era)
makeStakeAddressDelegationCertificate sCred delegatee =
  case useEra @era of
    e@ConwayEra ->
      obtainCommonConstraints e $
        Certificate $
          Ledger.mkDelegTxCert (Api.toShelleyStakeCredential sCred) delegatee


--- BELOW IS IN A SEPARATE "COMPATIBLE" MODULE DESIGNED FOR BACKWARDS COMPATIBILITY.

makeStakeAddressDelegationCertificate
  :: forall era
   . IsShelleyBasedEra era
  => StakeCredential
  -> Delegatee era
  -> Certificate (LedgerEra era)
makeStakeAddressDelegationCertificate sCred delegatee =
  case shelleyBasedEra @era of
    ...

-- All functions using the old Certificate era GADT will be marked {-# DEPRECATED #-}

``` 

We remove entirely the need to define a type like `StakeDelegationRequirements era`. The type synonym family is now responsible for resolving the era dependent type.

## Decision

The ADR gets adopted in `cardano-api`.

## Consequences

- An additional type synonym family per era dependent type.
- Removal of all the "old" api's data definitions that capture the notion of backwards compatibility.
- Removal of all helper functions that use these data definitions described in the aforementioned point.
- Implementation of new equivalent helper functions that utilize a type synonym family for the era dependent type.


