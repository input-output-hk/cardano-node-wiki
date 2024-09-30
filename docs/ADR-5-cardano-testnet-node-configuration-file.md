# Status

📜 Proposed 2024-09-10

# Context

There is no canonical way to create a node configuration file. `cardano-cli` offers this possibility as part of the bigger [create-cardano](https://github.com/IntersectMBO/cardano-cli/blob/551d9b9f2f244e0d681bf03eaa6d985565ac3a5b/cardano-cli/test/cardano-cli-golden/files/golden/help/latest_genesis_create-cardano.cli#L49) command, but this is ad-hoc.

Programmatic users proceed as follows instead:

* Scripts like [mkfiles.sh](https://github.com/IntersectMBO/cardano-node/blob/51a034a51c5cefdd6ab4b9ff1e71710cf0c96643/scripts/babbage/mkfiles.sh#L85) and [jobs.nix](https://github.com/input-output-hk/cardano-parts/blob/0abf510e0ed70fda4e6ad3ae71632c20d09f135a/flakeModules/jobs.nix#L253) copy an existing template.
* [cardano-testnet](https://github.com/IntersectMBO/cardano-node/blob/51a034a51c5cefdd6ab4b9ff1e71710cf0c96643/cardano-testnet/src/Testnet/Defaults.hs#L340) uses a hardcoded Haskell value.

# Decision

## Add a new command in the CLI

Introduce a new `debug check-node-configuration FILEPATH` command in `cardano-cli`. It will have the following options:

```
--node-configuration-file FILEPATH
[--fix-configuration-file]
```

Option `-node-configuration-file` specifies the path of the file to check. The optional flag `--fix-configuration-file` lets `cardano-cli` fix the wrong genesis hashes (or fill them in) in the file specified by `--node-configuration-file`. By default (i.e. when `--fix-configuration-file` is not specified), `check-node-configuration` only checks that the genesis hashes are correct: it doesn't correct them.

## Have `create-testnet-data` create the node configuration file

Have `create-testnet-data` create a default node configuration file (populating it with the paths and hashes of the genesis files).

# Consequences

* This will allow users that spin testnets to generate their node configuration file using `create-testnet-data`
* As a consequence, this will avoid having to keep track of external templates
* This will allow to remove some code in [cardano-testnet](https://github.com/IntersectMBO/cardano-node/blob/51a034a51c5cefdd6ab4b9ff1e71710cf0c96643/cardano-testnet/src/Testnet/Defaults.hs#L340)
* It will make possible to generalize `cardano-testnet` so that it allows to pass custom node configuration files (this is [cardano-node/issues/3719](https://github.com/IntersectMBO/cardano-node/issues/3719), and to have a handy way to generate those files.
