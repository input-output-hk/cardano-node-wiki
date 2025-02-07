# Status

📜 Proposed

# Context

# Proposed Design

## Top level exports

Each top level export should export distinct symbols.
This may depend on the case, for example: `Cardano.Api.Byron` symbols which are still in use in current eras, can be reexported through `Cardano.Api`.

* `Cardano.Api` - everything domain related for the currently used and upcoming eras
* `Cardano.Api.E` - everything for the obsoleted `E` era, for example `Cardano.Api.Byron`.
  1. Any symbol from era `E`, not in use in current era, should be exported through `Cardano.Api.E`.
  2. Any symbol from era `E` and still in use in current era, should be exported through `Cardano.Api` and additionally through `Cardano.Api.E`.
    If that's not possible to move it to a module under `Cardano.Api` module, the symbol should be copied to a module under `Cardano.Api`.
* `Cardano.Api.U` utility module, related to `U` feature set like, like `Cardano.Api.Pretty`, `Cardano.Api.Monad.Error`.
  Those should **not** be reexported from `Cardano.Api`, because they would pollute symbol namespace, and are not required for using `cardano-api` core features but can be useful for the advanced users.
  1. Reexports from upstream libraries like consensus, network or ledger, also fall into this category e.g. `Cardano.Api.Ledger`.

  Modules which are exposing general purpose classes and functions should go also into a separate module, and they not need to be reexported through `Cardano.Api`, for example:
  * `Cardano.Api.Tx.UTxO` - a wrapper type, with functions common to maps which would clash with other map data structures.
    A good example from Hackage is [`Data.Aeson.KeyMap`](https://hackage.haskell.org/package/aeson-2.2.3.0/docs/Data-Aeson-KeyMap.html).


## Deeper level exports

Any other modules not meant to be a top level export, should be placed inside `Cardano.Api.Internal` and be reexported through a suitable top level module.
In general `Cardano.Api.Internal.*` modules shouldn't be exposed in cabal package.
This however depends on the specific use case.
`Internal` in the module name should indicate that the module isn't really meant to be used directly, and users can expect more frequent breakage when relying on it.

* `Cardano.Api.Internal.X` should contain everything related to the domain `X`.
  For example `Cardano.Api.Address`.
  A good example from Hackage how to nest `Internal` modules is [`bytestring`](https://github.com/haskell/text/blob/master/src/Data/Text/Internal/Builder/RealFloat/Functions.hs) package.

# Decision


# Consequences

