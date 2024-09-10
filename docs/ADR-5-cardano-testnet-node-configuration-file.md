# Status

📜 Proposed 2024-09-10

# Context

There is no canonical way to create a node configuration file. `cardano-cli` offers this possibility as part of the bigger [create-cardano](https://github.com/IntersectMBO/cardano-cli/blob/551d9b9f2f244e0d681bf03eaa6d985565ac3a5b/cardano-cli/test/cardano-cli-golden/files/golden/help/latest_genesis_create-cardano.cli#L49) command, but this is ad-hoc.

Programmatic users proceed as follows instead:

* Scripts like [mkfiles.sh](https://github.com/IntersectMBO/cardano-node/blob/51a034a51c5cefdd6ab4b9ff1e71710cf0c96643/scripts/babbage/mkfiles.sh#L85) and [jobs.nix](https://github.com/input-output-hk/cardano-parts/blob/0abf510e0ed70fda4e6ad3ae71632c20d09f135a/flakeModules/jobs.nix#L253) copy an existing template.
* [cardano-testnet](https://github.com/IntersectMBO/cardano-node/blob/51a034a51c5cefdd6ab4b9ff1e71710cf0c96643/cardano-testnet/src/Testnet/Defaults.hs#L340) uses a hardcoded Haskell value.

# Decision

Introduce a new `conway genesis create-node-config file` command in `cardano-cli`. It will have the following options:

```
[--byron-genesis FILEPATH]
[--shelley-genesis FILEPATH]
[--alonzo-genesis FILEPATH]
[--conway-genesis FILEPATH]
[--node-config-template FILEPATH]
--out-file FILEPATH
```

If specified, the `--byron-genesis`, `--shelley-genesis`, etc. files will be read and their content be hashed, and
the output node configuration file will have the corresponding fields set. The `--node-config-template` allows to pass an existing file, which is useful if you want to augment it with the hashes. Finally the `--out-file` option is a mandatory option to specify where to write the generated node configuration file. If `--node-config-template` is omitted, the generated file will use defaults values.

An open question is whether to add flags to tune the configuration's file content on an individual field basis. It could be nice to show what possible tuning the node's configuration file allows.

# Consequences

* This will allow users that spin testnets to generate their node configuration file using `cardano-cli`
* As a consequence, this will avoid having to keep track of external templates
* This will allow to remove some code in [cardano-testnet](https://github.com/IntersectMBO/cardano-node/blob/51a034a51c5cefdd6ab4b9ff1e71710cf0c96643/cardano-testnet/src/Testnet/Defaults.hs#L340)
* It will make possible to generalize `cardano-testnet` so that it allows to pass custom node configuration files (this is [cardano-node/issues/3719](https://github.com/IntersectMBO/cardano-node/issues/3719), and to have a handy way to generate those files.
