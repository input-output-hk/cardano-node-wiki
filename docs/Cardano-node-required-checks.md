## Checks for the `master` branch

These are the agreed-upon required checks for the [`cardano-node` repository](https://github.com/input-output-hk/cardano-node).

Legend:
* ✅ Required check.
  Merges disallowed unless this job is successful.
* 👉 Pre-requisite required check.
  A required check depends on this job.  A job with a required check could not pass without this passing.
* ⛔ No required check and not a pre-requisite.

* ⛔ Check Markdown links / markdown-link-check (push)
* ⛔ Check HLint / build (push)
* ⛔ Check Stylish Haskell / build (push)
* ✅ Check cabal files / check-cabal-files (push)
  Ensures cabal files are well-formed
* ⛔ Check git dependencies / build (push)
* ⛔ Check mainnet configuration / build (push)
* ⛔ Check nix configuration / build (push)
* 👉 Haskell Linux CI / linux_ci (8.10.7, 3.10.1.0, ubuntu-latest)
  CI build and test with `cabal`.
* 👉 Haskell Windows & Mac CI / build (9.2.7, 3.10.1.0, macos-latest)
  CI build and test with `cabal`.
* 👉 Haskell Linux CI / linux_ci (9.2.7, 3.10.1.0, ubuntu-latest)
  CI build and test with `cabal`.
* 👉 Haskell Windows & Mac CI / build (9.2.7, 3.10.1.0, windows-latest)
  CI build and test with `cabal`.
* ✅ Haskell Linux CI / build-complete (push)
  Depends on:
  * Haskell Linux CI / linux_ci (8.10.7, 3.10.1.0, ubuntu-latest)
  * Haskell Linux CI / linux_ci (9.2.7, 3.10.1.0, ubuntu-latest)
* ✅ Haskell Windows & Mac CI / build-complete (push)
  Depends on:
  * Haskell Windows & Mac CI / build (9.2.7, 3.10.1.0, macos-latest)
  * Haskell Windows & Mac CI / build (9.2.7, 3.10.1.0, windows-latest)
* ⛔ Haskell Linux CI / release (push)
* ⛔ Haskell Windows & Mac CI / release (push)
* ⛔ ci/pr/nonrequired
* ⛔ ci/pr/system-tests
* ⛔ buildkite/cardano-node
* ⛔ buildkite/cardano-node/check-cabal-project
* ⛔ buildkite/cardano-node/docker-image
* ⛔ buildkite/cardano-node/pipeline
* ⛔ ci/cardano-deployment
* ⛔ ci/eval
* ✅ ci/hydra-build:required
* ⛔ ci/pr/required
