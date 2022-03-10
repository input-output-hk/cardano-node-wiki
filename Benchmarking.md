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

## Benchmarking: 3 steps

From the highest point of view, the benchmarking task includes 3 big steps:

1. Preparing the profile for the benchmark.
2. Running the benchmark on the cluster.
3. Analyzing benchmark results.

## Benchmarking Profiles on Bench Deployer

Each benchmark has a **profile**. The profile is a set of parameters that specify its particular details.

Switch to `screen`-window `bench-0` and run the following command:

```
$ b ps
```

The command `b` is from "benchmark", it's a basic command for benchmarking.

You will see the list of long names, for example:

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

Other numbers in profile's name are default ones. For example, `5ep` means that tx generator will work during `5` epochs, and `360kTx` means that tx generator will generate `360k` transactions.

### Profile Groups

As you can see, there are 3 big groups of profiles:

1. with word `regression` - default profiles,
2. with word `rtsflags` - like default profiles, but with specific RTS flags,
3. with word `Plutus` - profiles with smart contracts.

## Benchmark Run

Suppose you want to run a benchmark using a profile from our example, `k2-5ep-360kTx-7000kU-1250kD-80kbs`. Go to `bench-deployer` server, switch to `bench-0` window and run the command:

```
$ git log
```

Make sure you are on the commit of `bench-master` branch you want to use, and the branch has a clean status.

After that, run the command:

```
$ b reinit
```

And then, you are ready to run a benchmark:

```
$ b p training origin/master k2-5ep-360kTx-7000kU-1250kD-80kbs
```

Here you can see the following arguments:

1. `p` means "profile", so we want to run a benchmark with particular profile.
2. `training` is a batch name we use to describe this benchmark.
3. `origin/master` is a commit spec for `cardano-node` repository, here we want to take the latest commit from the `master` branch. It's possible to use a commit hash as well.
4. `k2-5ep-360kTx-7000kU-1250kD-80kbs` is a profile name.

The benchmark will be launched. The following line shows the current profile:

```
--( benchmarking profile:  k2-5ep-360kTx-7000kU-1250kD-80kbs, batch training, node baa9b5e59c5d448d475f94cc88a31a5857c2bda5 (origin/master)
```

If the genesis for this profile does not exist yet, you will see this line:

```
--( generating genesis due to miss:  k2-d1-1250kD-7000kU-84cb500 @ ../geneses/k2-d1-1250kD-7000kU-84cb500
```

But if the corresponding genesis already exists, the line will differ:

```
--( genesis cache hit:  k2-d1-1250kD-7000kU-84cb500
```

Then, you will see the following lines:

```
--( deploying profile k2-5ep-360kTx-7000kU-1250kD-80kbs
--(   era:           alonzo
--(   topology:      dense
--(   node:          baa9b5e59c5d448d475f94cc88a31a5857c2bda5 / origin/master
--(   ops:           9c42c56104742c591bb5f2867efaa6b61d03dc58 / bench-master  (modified)
--(   generator:     {"add_tx_size":100,"init_cooldown":45,"inputs_per_tx":2,"outputs_per_tx":2,"tx_fee":1000000,"tps":9,"era":"alonzo","tx_count":360000}
--(   genesis:       {.......}
--(   node:          {"rts_flags_override":[]}
--( hosts to deploy:  4 total:  explorer node-0 node-1 node-2
--( that's deployable in one go -- blasting ahead
``` 

These lines describe deploying details. As you see, this small cluster has 4 hosts, including `explorer` node where tx generator is working.

### Benchmark Finish

After some time, you will see the following lines:

```
--( Mon 07 Mar 2022 07:14:44 PM UTC, termination condition satisfied, stopping cluster.
--( run directory:  /home/dev/bench-0/runs/...
...
```

It means that benchmarking is finished and the cluster is stopped. The run directory is the path with all the data relative to this run: cluster params, genesis, configs, nodes' logs, etc. 

## Benchmark Analyze: Run

After the benchmark is finished, you want to analyze its results. To do it, switch to `workbench-1` window and use the command `wb`. This is an example:

```
$ wb --cls a --filters size-full std 2022-03-07-04.05.160222.cc1e.k51-5ep-360kTx-4000kU-1000kD-64kbs
```

where:

1. `--cls` means "clear screen",
2. `a` means "analyse",
3. `--filters size-full` means "filter full blocks",
4. `2022-03-07-04.05.160222.cc1e.k51-5ep-360kTx-4000kU-1000kD-64kbs` is full id of benchmark: `TIMESTAMP.BATCH.COMMIT.PROFILENAME`.

## Benchmark Analyze: Results

After `wb` is finished, you can find the `analysis` directory in `run directory`. For example, `~/bench-1/runs/2022-03-07-04.05.160222.cc1e.k51-5ep-360kTx-4000kU-1000kD-64kbs/analysis`.

This directory contains txt files you need:

1. `logs-node-1.timeline.txt`
2. `block-propagation.txt`

These files contains [CDF](https://en.wikipedia.org/wiki/Cumulative_distribution_function) data which can be used to form the results like these ones:

```
|                       | 1.34.0-rc3 | ledger-bump |    Δ |  Δ% |
|-----------------------+------------+-------------+------+-----|
| Avg process CPU usage |         39 |          46 |    7 |  15 |
| Avg RTS GC CPU usage  |         24 |          29 |    5 |  17 |
| Avg RTS Mut CPU usage |         16 |          17 |    1 |   6 |
| Avg RSS               |       5120 |        4850 | -270 |  -6 |
| Avg RTS heap          |       5090 |        4820 | -270 |  -6 |
| Avg RTS GC live bytes |       2095 |        1820 | -275 | -15 |
|-----------------------+------------+-------------+------+-----|
#+TBLFM: $4=$3-$2::$5=round(100*$4/$3)
```

```
|                                   | 1.34.0-rc3 | ledger-bump |  Δ |  Δ% |
|-----------------------------------+------------+-------------+----+-----|
| average leadership check Δt       |         41 |          49 |  8 |  20 |
| average leadership check duration |          5 |          20 | 15 | 300 |
| average forge time                |         68 |         113 | 45 |  66 |
| average adoption time             |         65 |          58 | -7 | -11 |
| average time to announce          |          1 |           1 |  0 |   0 |
| average time to start sending     |         27 |          36 |  9 |  33 |
|-----------------------------------+------------+-------------+----+-----|
|                                   |        207 |         277 | 70 |  34 |
#+TBLFM: @>$2=vsum(@I..@II)::@>$3=vsum(@I..@II)::$4=$3-$2::$5=round(100*$4/$2)
```

```
|                               | 1.34.0-rc3 | ledger-bump |   Δ | Δ% |
|-------------------------------+------------+-------------+-----+----|
| average first time to notice  |        496 |         618 | 122 | 25 |
|-------------------------------+------------+-------------+-----+----|
| average time to request       |         12 |          18 |   6 | 50 |
| average time to fetch         |        370 |         358 | -12 | -3 |
| average adoption time         |         63 |          67 |   4 |  6 |
| average time to announce      |          1 |           1 |   0 |  0 |
| average time to start sending |        137 |         131 |  -6 | -4 |
|-------------------------------+------------+-------------+-----+----|
|                               |        583 |         575 |  -8 | -1 |
#+TBLFM: @>$2=vsum(@II..@III)::@>$3=vsum(@II..@III)::$4=$3-$2::$5=round(100*$4/$2)
```

```
 %tile   0.50  0.80  0.90  0.92  0.94  0.96  0.98  1.00
1.34.0-rc3---------------------------------------------
  0.5    0.8   1.1   1.16  1.17  1.18  1.19  1.21  1.25
  0.9    1.43  1.92  2.14  2.18  2.25  2.3   2.38  2.65
  1.0   36.16 40.89 44.57 50.96 50.96 51.22 57.57 70.26
  avg   0.965 1.380 1.491 1.529 1.574 1.618 1.698 1.965
1.34.0-rc3---------------------------------------------
  0.5    0.71  1.08  1.13  1.14  1.15  1.16  1.17  1.2
  0.9    1.3   1.69  1.91  1.98  2.04  2.12  2.2   2.51
  1.0   43.56 57.03 62.97 63.36 66.21 66.63 69.57 79.51
  avg   0.935 1.356 1.455 1.484 1.511 1.547 1.587 1.770
lehins/ledger-bump-------------------------------------
  0.5    0.83  1.13  1.18  1.19  1.19  1.2   1.22  1.26
  0.9    1.46  1.87  2.11  2.15  2.23  2.34  2.44  2.75
  1.0   55.75 71.94 83.88 84.51 84.89 87.69 92.82 99.08
  avg   1.051 1.489 1.645 1.681 1.720 1.766 1.835 2.055
```