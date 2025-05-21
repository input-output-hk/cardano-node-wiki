# Status

- [ ] Proposed 2024-05-21

# Context

## The Problem
In `cardano-api` we have multiple functions performing conversions between one value of the type to the other, for example:

```haskell
toShelleyScriptHash :: ScriptHash -> ScriptHash
fromShelleyScriptHash :: ScriptHash -> ScriptHash
```

There are multiple naming conventions for the conversion functions which makes them hard to locate.
Some conversion functions with lengthy names, are not very convenient to use.

## Solution Proposal

### Type classes

For total functions, which are simply converting a value from one type to another, we can use type classes [`Inject`](https://cardano-ledger.cardano.intersectmbo.org/cardano-ledger-core/Cardano-Ledger-BaseTypes.html#t:Inject) & [`Convert`](https://cardano-api.cardano.intersectmbo.org/cardano-api/Cardano-Api-Internal-Eras.html#t:Convert):
```haskell
class Inject t s where
  inject :: t -> s

class Convert (f :: a -> Type) (g :: a -> Type) where
  convert :: forall a. f a -> g a
```

The use of those conversion functions is limited to **internal use only**.
The library should still export conversion functions with explicit type names for better readability.

### Explicit conversion functions

For explicit conversion functions, the following naming convention should follow:

```haskell
fooToBar :: Foo -> Bar
```

#### Qualified imports
If the module exporting conversion functions is meant to be imported qualified, and provides functions for operating on a single data type, a shorter name with `from` prefix is allowed:

```haskell
module Data.Foo where

fromBar :: Bar -> Foo
```

where the usage would look like:
```haskell
import Data.Foo qualified as Foo

Foo.fromBar bar
```

# Decision

TBD

# Consequences

## Advantages
- An uniform API for total conversions
- A list of `Inject` instances lists all available conversions for the type
- Less maintenance burden with regards to the naming conventions of the conversion functions

## Disadvantages
- It may be a bit surprising how to discover available conversions, because one would have to browse the type's `Inject` instances to find the conversion functions they are looking for - instead of looking for exported functions.
