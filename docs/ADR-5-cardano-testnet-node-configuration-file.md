# Status

📜 Proposed 2024-09-10

# Context

There is no way to check a node configuration file for basic sanity issues. This is a problem for people running testnets, because they will only discover _after having started their testnet_ that something is wrong (because the node will refuse to start). And starting a testnet is a costly operation, meaning the failures will only happen after tenth of seconds; causing the feedback loop to be slow.

# Decision

## Add a new command in the CLI

Introduce a new `debug check-node-configuration FILEPATH` command in `cardano-cli`. It will have the following options:

```
--node-configuration-file FILEPATH
```

Option `-node-configuration-file` specifies the path of the file to check. This command checks that:

1. The configuration file can be loaded (according to [this encoding](https://github.com/IntersectMBO/cardano-api/blob/4dde2e65c496f989f079354f407e7617563f4bc7/cardano-api/internal/Cardano/Api/LedgerState.hs#L1063)).
2. Paths of genesis files are specified.
3. Hashes of genesis files are specified and are correct.

## Alternatives considered

### Have a `--fix` flag

We considered having a `--fix` flag that would make `check-node-configuration`:

1. Fill the hashes of genesis files if missing
2. Fix the hashes of genesis files if present but wrong

However, in the end, we favored having a pure command (i.e. a command that doesn't modify things) so we ruled `--fix` out.

### Have `create-testnet-data` create the node configuration file

We considered having `create-testnet-data` create a default node configuration file, and populating it with the paths and hashes of genesis files. This could have been nice to users, but it is complicated to implement, because most of the types for the node configuration file live in `cardano-node` (see [here](https://github.com/IntersectMBO/cardano-node/blob/ef5f0a9ed52d969b3753c96955add25b9e08f02d/cardano-node/src/Cardano/Node/Configuration/POM.hs#L87)). What we have in `cardano-api` (see [here](https://github.com/IntersectMBO/cardano-api/blob/4dde2e65c496f989f079354f407e7617563f4bc7/cardano-api/internal/Cardano/Api/LedgerState.hs#L1048)) for the node configuration file is not general enough to create a full-fledged node configuration file.

# Consequences

This will allow users that spin testnets to check their node configuration file using `check-node-configuration` and catch errors earlier (before starting their testnet).
