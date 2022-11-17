I think standardisation is worthwhile, but I think we ought to have. a process to establish that standard rather than just create one more standard among many.

I had quick look at what we currently do with regards to generators and it is highly varied.

For example we have Control.State.Transition.Generator, Generators, PlutusIR.Generators.AST, Test.Cardano.Prelude.Gen, Hedgehog.Gen.Double, etc.

At the time when deciding where how to structure modules, I chose the following structure Gen.Cardano.*and the decision was deliberate, although it wasn’t advertised.

The Gen.Cardano.* I wanted to convey the following information:

Everything that has the structure Cardano.* with Cardano as the top-level is production code, so by inference anything where Cardano is not top-level is not production code.


These modules aren’t just for generators, but also they in a package library component that make available for import into multiple test suites.  I wanted to make it different from modules export generators but aren’t available for sharing.  This is to prevent people from getting confused when writing generators and/or tests.  For example.  I’m in a module that defines generators.  I want to import that other module that containers generators so I can use them to build my own generator, but compiler says it can’t fine the module, and only after some head scratching to I realise the modules are in different packages.
The modules are not part