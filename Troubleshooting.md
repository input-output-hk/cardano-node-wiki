## Delegated some stake is not earning rewards

There are a number of reasons you may not be earning rewards.  Check for the following things to ensure the testnet is set up properly:

* `d` parameter above zero: Check protocol parameters
* Pool makes blocks: There's a forge adopted metric on prometheus exporter built-in to node. One could also check logs.
* Pool meets pledge: This can be checked via `ledgerState.nesES.esLState._delegationState._pstate._pParams.<POOL_ID>._poolPledege` in the ledger dump. To get the owners' stake (to see if they meet the pledge, get the key hashes from `ledgerState.nesES.esLState._delegationState._pstate._pParams.<POOL_ID>._poolOwners`. Then maybe use the CLI to get the stake associated with each owner, and sum them.
* Rho above zero: check protocol parameters
* Reserves above zero (total supply > sum of all utxo).  Check for `ledgerState.nesES.esAccountState._reserves` in the ledger state.
