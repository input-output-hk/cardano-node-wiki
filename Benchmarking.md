This page describes the benchmarking procedure.

## Bench Deployer Access

First of all, please make sure you have access to the `bench-deployer` server by typing this command on your computer:

```
$ ssh bench
```

You will be logged in as `dev@bench-deployer`.

## `cardano-ops` Repository

Also, please make sure you have access to [cardano-ops](https://github.com/input-output-hk/cardano-ops) repository. You need it to be able to push in our branch `bench-master`.

## Working on Bench Deployer: Main Rules

Now log in to `bench-deployer` server using `ssh bench` command.

**IMPORTANT**: Please remember that all actions on `bench-deployer` server must always be performed in `screen` emulation. So first of all, run `screen -x` command.

There are important `screen`-windows we need:

1. `bench-0` - for running benchmarks on **small** clusters.
2. `bench-1` - for running benchmarks on **big** (real-world) clusters.
3. `bench-2` - for running benchmarks on **small** clusters.
4. `workbench-1` - for running **analyzing** of benchmarks results.

To move between `screen`-windows, use following key combinations: `(Ctrl+A)+P` (to the right) or `(Ctrl+A)+N` (to the left).

**IMPORTANT**: do not use `exit` command in the `screen`-window! Instead, just close your terminal window, to keep `screen`-window working.

By default, when you switch to some `screen`-window, you are in `nix-shell`. If you exited from it, it's possible to go back inside `nix-shell` using `nsh` command.

## Benchmarking Profiles on Bench Deployer

Each benchmark has a **profile**. The profile is a set of parameters that specify its particular details.

Switch to `screen`-window `bench-0` and run:

```
$ b ps
```

You'll see the list of long names, for example:

```
--( 2022-03-10T10:29:41+00:00:  ps
--( NIXOPS_DEPLOYMENT:  bench-0 (inherited)
                                                                                      smoke: 
                                                                                smoke-epoch: 
                                                                          smoke-dense-large: 
                                                                           smoke-large-1000: 
                                                                           smoke-large-5000: 
 k2-5ep-360kTx-4000kU-1000kD-64kbs-10MUTx-10BStTx-50MUBk-40BStBk-always-succeeds-spending-0: Plutus return-success
             k2-5ep-7.5kTx-4000kU-1000kD-64kbs-10MUTx-10BStTx-50MUBk-40BStBk-1i-1o-sum-3304: Plutus, 1e10-cpu
            k2-5ep-0.08kTx-4000kU-1000kD-64kbs-10MUTx-10BStTx-50MUBk-40BStBk-1i-1o-sum-3304: Plutus, 1e10-cpu smoke
             k2-5ep-7.5kTx-4000kU-1000kD-64kbs-10MUTx-10BStTx-50MUBk-40BStBk-1i-1o-sum-1144: Plutus, 1e7-mem
                k2-5ep-0.1kTx-4000kU-1000kD-64kbs-10MUTx-10BStTx-50MUBk-40BStBk-1i-1o--null: Plutus, auto-mode-smoke-test
                 k2-7ep-14kTx-4000kU-1000kD-64kbs-10MUTx-10BStTx-50MUBk-40BStBk-1i-1o--null: Plutus, baseline
                 k2-7ep-14kTx-4000kU-1000kD-73kbs-11MUTx-10BStTx-50MUBk-40BStBk-1i-1o--null: Plutus, bump 1, Dec 2 2021
                 k2-7ep-14kTx-4000kU-1000kD-73kbs-12MUTx-10BStTx-50MUBk-40BStBk-1i-1o--null: Plutus, bump 2, 2022
                                                          k2-5ep-360kTx-7000kU-1250kD-80kbs: regression, March 2022 data set sizes
                                                          k2-5ep-360kTx-4000kU-1000kD-64kbs: regression, October 2021 data set sizes
                                      k2-5ep-360kTx-4000kU-1000kD-64kbs-RTS--H4G-M6553M-c70: rtsflags: batch1, best CPU/mem
                                          k2-5ep-360kTx-4000kU-1000kD-64kbs-RTS--H4G-M6553M: rtsflags: batch1, better mem, costlier CPU
                                                 k2-5ep-360kTx-4000kU-1000kD-64kbs-RTS--A4m: rtsflags: cache fitting
                                              k2-5ep-360kTx-4000kU-1000kD-64kbs-RTS--A4m-N4: rtsflags: cache fitting + higher parallelism
                                                 k2-5ep-360kTx-4000kU-1000kD-64kbs-RTS--A1m: rtsflags: cache fitting extreme
                                              k2-5ep-360kTx-4000kU-1000kD-64kbs-RTS--A1m-N4: rtsflags: cache fitting extreme + parallelism
                                                 k2-5ep-360kTx-4000kU-1000kD-64kbs-RTS--A2m: rtsflags: cache fitting hard
                                              k2-5ep-360kTx-4000kU-1000kD-64kbs-RTS--A2m-N4: rtsflags: cache fitting hard + parallelism
                                  k2-5ep-360kTx-4000kU-1000kD-64kbs-RTS--C0-A32m-n1m-AL512M: rtsflags: suggestion from PR 3399
```

These long names are the names of available profiles. For example, `k2-5ep-360kTx-7000kU-1250kD-80kbs` is the name of profile, and `regression, March 2022 data set sizes` is its description.

Now clone `cardano-ops` repository, go to it, switch to `bench-master` branch and find `bench/profile-definitions.jq` file. This file describes existing profiles. Find `utxo_delegators_density_profiles` function with this `desc` inside:

```
{ desc: "regression, March 2022 data set sizes"
, genesis: { utxo:           7000000
           , delegators:     1250000
           , max_block_size: 80000
           }
}
```

As you can see, here we specify a profile that can be used on `bench-deployer` server.

All the numbers are described below.

## Benchmarking Profiles

The name of each profile is generated automatically and contains profile's main parameters. For example, compare the name `k2-5ep-360kTx-7000kU-1250kD-80kbs` with its description in `bench/profile-definitions.jq` file:

```
{ desc: "regression, March 2022 data set sizes"
, genesis: { utxo:           7000000
           , delegators:     1250000
           , max_block_size: 80000
           }
}
```

Here you can see the following numbers that are specific for this profile:

1. `7000kU` means that `utxo` is `7000000`,
2. `1250kD` means that `delegators` is `1250000`,
3. `80kbs` means that `max_block_size` is `80000` bytes.

Other numbers in profile's name are default ones:

1. `k2` means that there are `2` pools based on genesis,
2. `5ep` means that tx generator will work during `5` epochs,
3. `360kTx` means that tx generator will generate `360k` transactions.
