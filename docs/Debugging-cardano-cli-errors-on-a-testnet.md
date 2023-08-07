# Debugging cardano-cli errors on a testnet
This document is intended for Cardano node and `cardano-cli` (CLI) users, as well as Cardano developers that wish to investigate CLI problems.

We recommend making sure the `cardano-node` tests pass before start debugging a problem, and also take a look at the [Consensus sanity checklist](https://input-output-hk.github.io/ouroboros-consensus/docs/for-developers/SanityChecks).

Once the node tests sucessfully pass and you've read through the Consensus checklist, we need to collect node logs to speed up the proccess of finding the cause of a problem. We describe below the steps to run a node that connects to a given testnet and get access to a `cardano-cli` that can be used to communicate with the node. These instructions work on a Unix environment with `nix` installed.

Clone [cardano-world](https://github.com/input-output-hk/cardano-world) and checkout the testnet branch (eg `sanchonet`).

Once inside the `cardano-world` directory we need to enter a `nix` shell which has `cardano-node` in its path:

```sh
nix shell github:input-output-hk/cardano-node\?ref=8.1.1\#cardano-node
```

where `8.1.1` should be replace by the version that is intended to be tested. Once inside the `nix` shell we can check that the `cardano-node` executable is in our `PATH`:

```sh
which cardano-node
```

```text
/nix/store/scmkvbvx1355nwmclmrfd2snwvsgsn30-cardano-node-exe-cardano-node-8.1.1/bin/cardano-node
```

The configuration files for the node can be found inside the `docs/environment/testnet_name`. You can copy these files to a new directory to edit them and have the node use the configuration inside said directory. For instance, after we copy the configuration files to a director named `node-custom-config`, the node can be readily run:

```sh
 cardano-node run \
  --config node-custom-config/config.json \
  --database-path my-node-db \
  --socket-path node.socket \
  --topology custom-node-config/topology.json | tee -a node.log
```

In particular, make sure the enable the relevant tracers in the node configuration file. In the example above this file is `node-custom-config/config.json`.

Once the node is running, we need a CLI to communicate with it. To enter a shell that has `cardano-cli` in its `PATH` you can run:

```sh
nix develop .#x86_64-linux.automation.devshells.dev
which cardano-cli
```

```text
/nix/store/cbr6kgfsq7pqjwdirvm75lwzaibxvyfm-devshell-dir/bin/cardano-cli
```


## Debugging options

It might be a good idea to adjust the `mapSeverity` attribute for the `LocalTxSubmissionProtocol` to `Debug` in the node configuration file.

```json
...
    "mapSeverity": {
      "cardano.node.LocalTxSubmissionProtocol": "Debug"
    }
...

```
