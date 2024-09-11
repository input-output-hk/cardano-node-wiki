# Status

📜 Proposed 2024-09-10

# Context

There is no canonical way to create a node configuration file. `cardano-cli` offers this possibility as part of the bigger [create-cardano](https://github.com/IntersectMBO/cardano-cli/blob/551d9b9f2f244e0d681bf03eaa6d985565ac3a5b/cardano-cli/test/cardano-cli-golden/files/golden/help/latest_genesis_create-cardano.cli#L49) command, but this is ad-hoc.

Programmatic users proceed as follows instead:

* Scripts like [mkfiles.sh](https://github.com/IntersectMBO/cardano-node/blob/51a034a51c5cefdd6ab4b9ff1e71710cf0c96643/scripts/babbage/mkfiles.sh#L85) and [jobs.nix](https://github.com/input-output-hk/cardano-parts/blob/0abf510e0ed70fda4e6ad3ae71632c20d09f135a/flakeModules/jobs.nix#L253) copy an existing template.
* [cardano-testnet](https://github.com/IntersectMBO/cardano-node/blob/51a034a51c5cefdd6ab4b9ff1e71710cf0c96643/cardano-testnet/src/Testnet/Defaults.hs#L340) uses a hardcoded Haskell value.

# Decision

## Add a new command in the CLI

Introduce a new `conway genesis check-node-config file` command in `cardano-cli`. It will have the following options:

```
[--byron-genesis FILEPATH]
[--shelley-genesis FILEPATH]
[--alonzo-genesis FILEPATH]
[--conway-genesis FILEPATH]
[--node-config FILEPATH]
```
For every `--era-genesis` file, the command will read the file specified at the given `FILEPATH`, hash its content and check that the file specified at `--node-config` contains the correct path and the correct hash for this genesis file.

## Have `create-testnet-data` create the node configuration file

1. Have `create-testnet-data` create a default node configuration file (populating it with the paths and hashes of the genesis files).
2. Add a `--node-config-template` optional option to `create-testnet-data which will be used to generate the node configuration file, instead of using a default one.

# Consequences

* This will allow users that spin testnets to generate their node configuration file using `create-testnet-data`
* As a consequence, this will avoid having to keep track of external templates
* This will allow to remove some code in [cardano-testnet](https://github.com/IntersectMBO/cardano-node/blob/51a034a51c5cefdd6ab4b9ff1e71710cf0c96643/cardano-testnet/src/Testnet/Defaults.hs#L340)
* It will make possible to generalize `cardano-testnet` so that it allows to pass custom node configuration files (this is [cardano-node/issues/3719](https://github.com/IntersectMBO/cardano-node/issues/3719), and to have a handy way to generate those files.
