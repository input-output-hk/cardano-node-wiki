This page describes the benchmarking procedure.

## Bench Deployer Access

First of all, please make sure you have access to the `bench-deployer` server by typing this command on your computer:

```
$ ssh bench
```

You will be logged in as `dev@bench-deployer`.

## `cardano-ops` Repository

Also, please make sure you have access to [cardano-ops](https://github.com/input-output-hk/cardano-ops) repository. You need it to be able to push in our branch `bench-master`.

## Benchmarking Profiles

Each benchmark has a **profile**. The profile is a set of parameters that specify its particular details.

Clone `cardano-ops` repository, go to it, switch to `bench-master` branch and find `bench/profile-definitions.jq` file. This file describes existing profiles, we'll explore it later.

## Working on Bench Deployer

Now log in to `bench-deployer` server using `ssh bench` command.

