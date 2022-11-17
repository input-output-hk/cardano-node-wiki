# Context

In a project, it is desirable to have a standard module structure so that a developer can easily navigate the code base and know when they write new code they know where to put the code so that other people can find it.

At IOG, we do not yet have an organisation-level standard for organising modules.  Short of having an organisation-level standard, it is worthwhile for `cardano-node` code to formally define the module structure for its code.

Different people have different ideas for how modules should be structured.

For example doing a search for modules that export generators in `cardano-node`, we have the following:

* `cardano-api`
  * `library gen`
    * `Gen.Cardano.Api`
    * `Gen.Cardano.Api.Metadata`
    * `Gen.Hedgehog.Roundtrip.Bech32`
* `cardano-node`
  * `test-suite cardano-node-test`
    * `Test.Cardano.Node.Gen`

For other IOG projects, we have the following:

* `cardano-crypto-wrapper`
  * `test-suite test`
    * `Test.Cardano.Crypto.Gen`
* `cardano-crypto-wrapper`
  * `test-suite test`
    * `Test.Cardano.Crypto.Gen`
* `cardano-crypto-test`
  * `library`
    * `Test.Cardano.Crypto.Gen`
* `byron-spec-ledger`
  * `library`
    * `Hedgehog.Gen.Double`
    * `Byron.Spec.Ledger.Core.Generators`
* `cardano-ledger-byron`
  * `test-suite cardano-ledger-byron-test`
    * `Test.Cardano.Chain.Block.Gen`
* `cardano-ledger-byron-test`
  * `library`
    * `Test.Cardano.Chain.Block.Gen`
* `cardano-ledger-shelley-test`
  * `library`
    * `Test.Cardano.Ledger.Shelley.Generator.Block`
    * `Test.Cardano.Ledger.Shelley.Serialisation.Generators`
  * `benchmark mainbench`
    * `Cardano.Ledger.Shelley.Bench.Gen`
* `cardano-prelude-test`
  * `library`
    * `Test.Cardano.Prelude.Gen`
* `ouroboros-consensus-byron-test`
  * `library`
    * `Test.Consensus.Byron.Generators`
* `plutus-chain-index-core`
  * `test-suite plutus-chain-index-test`
    * `Generators`
* `plutus-example`
  * `test-suite plutus-example-test`
    * `Test.PlutusExample.Gen`
* `plutus-core`
  * `library plutus-core-testlib`
    * `PlutusCore.Generators`
    * `PlutusCore.Generators.AST`
  * `benchmark cost-model-budgeting-bench`
    * `Generators`
* `small-steps-test`
  * library`
    * `Control.State.Transition.Generator`
    * `Control.State.Transition.Trace.Generator.QuickCheck`


I think standardisation is worthwhile, but I think we ought to have. a process to establish that standard rather than just create one more standard among many.

I had quick look at what we currently do with regards to generators and it is highly varied.

For example we have Control.State.Transition.Generator, Generators, PlutusIR.Generators.AST, Test.Cardano.Prelude.Gen, Hedgehog.Gen.Double, etc.

At the time when deciding where how to structure modules, I chose the following structure Gen.Cardano.*and the decision was deliberate, although it wasn’t advertised.

The Gen.Cardano.* I wanted to convey the following information:

Everything that has the structure Cardano.* with Cardano as the top-level is production code, so by inference anything where Cardano is not top-level is not production code.


These modules aren’t just for generators, but also they in a package library component that make available for import into multiple test suites.  I wanted to make it different from modules export generators but aren’t available for sharing.  This is to prevent people from getting confused when writing generators and/or tests.  For example.  I’m in a module that defines generators.  I want to import that other module that containers generators so I can use them to build my own generator, but compiler says it can’t fine the module, and only after some head scratching to I realise the modules are in different packages.
The modules are not part