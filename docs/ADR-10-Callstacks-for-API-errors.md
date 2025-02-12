# Status
- [ ] Adopted YYYY/MM/DD

# Recommended Reading

- [ADR 8](https://github.com/input-output-hk/cardano-node-wiki/blob/main/docs/ADR-8-Use-RIO-in-cardano%E2%80%90cli.md)

# Context

Currently, errors in `cardano-api` are instances of the class `Error`, which only requires one method:

```haskell
class Error e where
  prettyError :: e -> Doc ann
```

In `cardano-api`, each error is typically specific to a function, and the way errors are aggregated is by wrapping errors within other errors.

The issue with wrapping is that finding out the original source of an error is a lot of work.
Because it requires following the execution path manually, and tracking where each constructor is applied.

It would be more convenient to have a stack-trace, that points to the offending line and their callers.
This would allow to find the exact points in the code where the issue happened.

Because we are using `Error` almost everywhere.
It would make most sense to modify it to ensure that it provides a stack-trace.

There is a known constraint for providing stack-traces in Haskell (`HasStackTrace`).
It would be ideal to just modify the definition of `Error` to require `HasStackTrace`.
But this doesn't work, because it can only be applied to functions.

# Proposal

As a work-around, we propose adding a requirement for an error to be able to provide a value of the type `CallStack`.
This is not a guarantee, that the `CallStack` provided corresponds to the exact place where the `Error` was crafted, but we can leave that to the good will of the developer.
But it does guarantee that the developer won't forget to add a `CallStack`.

```haskell
class Error e where
  prettyError :: e -> Doc ann
  getErrorCallStack :: e -> CallStack
```

We could leave it there but, we don't want each error to manually print their own stack-trace as part of their implementation of `prettyError`, we would rather implement that functionality once and for all.
We want different errors to customize their own error message, but we expect the overall template of `error message + stack-trace` to be presented in the same consistent way.
Also, we want to be able to print the error descriptions without stack-trace if necessary.

We can achieve both things by separating the actual error from our extended `Error` type.
So we can rename the old `Error` as `ErrorContent`.
Which would leave us with a class just like this:

```haskell
class ErrorContent e where
  prettyErrorContent :: e -> Doc ann
```

And then, we could define our extended `Error` as:

```haskell
data Content = forall e. ErrorContent e => Content e

class Error e where
  getErrorContent :: e -> Content
  getErrorCallStack :: e -> CallStack
```

We need `e` in `Content` wrapper to be an existential type, to prevent the type of the `ErrorContent` from propagating to the type of `Error`, while ensuring that it is indeed an `ErrorContent`.
If we do this, we could define two generic functions for every `Error e`.

One to print the original error, without a stack-trace:

```haskell
prettyErrorWithoutStack :: Error e => e -> Doc ann
prettyErrorWithoutStack e =
  case getErrorContent e of
    Content content -> prettyErrorContent content
```

And one to print the extended error, with the stack-trace:

```haskell
prettyError :: Error e => e -> Doc ann
prettyError e =
  vsep
    [ prettyErrorWithoutStack e
    , "Call stack:\n"
    , nest 2 $ pretty $ prettyCallStack (getErrorCallStack e)
    ]
```

## Going deeper

Unfortunately, this will only get us a stack trace for the outmost wrapper error.
And the inner error, which is the most important one, will remain hidden.

One easy solution is to replace the call to `prettyErrorWithoutStack` in `prettyError` with a call to itself (`prettyError`), and ensuring that the error pretty printed by each `ErrorContent e` in its `prettyErrorContent` function recursively calls `prettyError`.

That will make sure that all `Error`s in the stack will have their chance to print their stack-trace.
And we could nest them for better readability.

But this is not ideal becuase they would be shown like this:

```
  Error A: The function failed because:
     Error B: The function failed because:
        ...
     Stack trace for Error B
  Stack trace for Error A
```

Which pushes the stack trace as far as possible form the description of the error.

While it would be much better like this:

```
  Error A: The function failed because:
  Stack trace for Error A
  Caused by:
     Error B: The function failed because:
     Stack trace for Error B
     Caused by:
        ...
```

And we can achieve that by extending our class `Error` with yet another function (`getCause`).
This is how it would look like:

```haskell
class Error e where
  getErrorContent :: e -> Content
  getErrorCallStack :: e -> CallStack
  getCause :: e -> Maybe Cause
```

Where `Cause` is another wrapper like `Content`:

```haskell
data Cause = forall c. Error c => Cause c
```

And that would allow us to define `prettyError` as follows:

```haskell
prettyError :: Error e => e -> Doc ann
prettyError e =
  vsep
    [ prettyErrorWithoutStack e
    , "Call stack:\n"
    , nest 2 $ pretty $ prettyCallStack (getErrorCallStack e)
    , case getCause e of
        Nothing -> mempty
        Just (Cause cause) ->
          vsep
            [ "Caused by:"
            , nest 2 $ "Caused by:" <+> prettyError cause
            ]
    ]
```

This allows us to keep the error structures as they are, while annotating them with their stack-traces at each point.

# Consequences

The tricky bit is updating the existing code to comply with the new instance.

The types need to be extended someway to allocate for the stack-trace, and there is no clean way around it.
It may be possible to do this in a more or less transparent way by using `Deriving` and/or `Template Haskell`.
But, other than that, it comes down to adding a space for the stack-trace in the types that implement the `Error` instance.

At least, we can have a reusable wrapper for errors (that must now implement the `ErrorContent` class).

So we can define an `ErrorWithStack` data type:

```haskell
data ErrorWithStack e =
    ErrorContent e => RootErrorWithStack e CallStack
  | forall c. (ErrorContent e, Error c) => CausedErrorWithStack e CallStack c
```

The difference between the constructors is that one has a causing error, and the other one doesn't.

We can then create convenience functions for creating them from existing `ErrorContent`s:

```haskell
mkError :: (ErrorContent e, HasCallStack) => e -> ErrorWithStack e
mkError e = RootErrorWithStack e callStack

mkErrorWithCause :: (ErrorContent e, Error c, HasCallStack) => e -> c -> ErrorWithStack e
mkErrorWithCause e = CausedErrorWithStack e callStack
```

And we can trivially derive an `Error` instance for `ErrorWithStack`:

```haskell
instance ErrorContent e => Error (ErrorWithStack e) where
  getErrorContent :: ErrorWithStack e -> Content
  getErrorContent (RootErrorWithStack e _) = Content e
  getErrorContent (CausedErrorWithStack e _ _) = Content e

  getErrorCallStack :: ErrorWithStack e -> CallStack
  getErrorCallStack (RootErrorWithStack _ cs) = cs
  getErrorCallStack (CausedErrorWithStack _ cs _) = cs

  getCause :: ErrorWithStack e -> Maybe Cause
  getCause (RootErrorWithStack _ _) = Nothing
  getCause (CausedErrorWithStack _ _ c) = Just $ Cause c
```

## Adapting existing code

Adapting existing code does require modifying the existing code quite a bit in principle, and it is not always obvious how.

For example, we can adapt the following error:

```haskell
data ProtocolParametersConversionError
  = PpceOutOfBounds !ProtocolParameterName !Rational
  | PpceVersionInvalid !ProtocolParameterVersion
  | PpceInvalidCostModel !CostModel !CostModelApplyError
  | PpceMissingParameter !ProtocolParameterName
  deriving (Eq, Show, Data)

instance Error ProtocolParametersConversionError where
  prettyError = \case
    PpceOutOfBounds name r ->
      "Value for '" <> pretty name <> "' is outside of bounds: " <> pretty (fromRational r :: Double)
    PpceVersionInvalid majorProtVer ->
      "Major protocol version is invalid: " <> pretty majorProtVer
    PpceInvalidCostModel cm err ->
      "Invalid cost model: " <> pretty @Text (display err) <> " Cost model: " <> pshow cm
    PpceMissingParameter name ->
      "Missing parameter: " <> pretty name
```

By first renaming it as `ProtocolParametersConversionErrorContent`, changing the instance to `ErrorContent` and the function implemented to `prettyErrorContent`, and by creating a type synonym for `ErrorWithStack ProtocolParametersConversionErrorContent` for convenience:

```haskell
data ProtocolParametersConversionErrorContent
  = PpceOutOfBounds !ProtocolParameterName !Rational
  | PpceVersionInvalid !ProtocolParameterVersion
  | PpceInvalidCostModel !CostModel !CostModelApplyError
  | PpceMissingParameter !ProtocolParameterName
  deriving (Eq, Show, Data)

instance ErrorContent ProtocolParametersConversionErrorContent where
  prettyErrorContent = \case
    PpceOutOfBounds name r ->
      "Value for '" <> pretty name <> "' is outside of bounds: " <> pretty (fromRational r :: Double)
    PpceVersionInvalid majorProtVer ->
      "Major protocol version is invalid: " <> pretty majorProtVer
    PpceInvalidCostModel cm err ->
      "Invalid cost model: " <> pretty @Text (display err) <> " Cost model: " <> pshow cm
    PpceMissingParameter name ->
      "Missing parameter: " <> pretty name

type ProtocolParametersConversionError = ErrorWithStack ProtocolParametersConversionErrorContent
```

We can then just resolve type errors that appear at those places where `ProtocolParametersConversionErrorContent` is created by appending `mkError` before.
For example, we get an error here:

```haskell
mkProtVer :: (Natural, Natural) -> Either ProtocolParametersConversionError Ledger.ProtVer
mkProtVer (majorProtVer, minorProtVer) =
  maybeToRight (PpceVersionInvalid majorProtVer) $
    (`Ledger.ProtVer` minorProtVer) <$> Ledger.mkVersion majorProtVer
```

And after fixing it we would get:
```haskell
mkProtVer :: (Natural, Natural) -> Either ProtocolParametersConversionError Ledger.ProtVer
mkProtVer (majorProtVer, minorProtVer) =
  maybeToRight (mkError $ PpceVersionInvalid majorProtVer) $
    (`Ledger.ProtVer` minorProtVer) <$> Ledger.mkVersion majorProtVer
```

Note that we have added `mkError` before the `PpceVersionInvalid` constructor.


## Errors within errors

In the previous example, the errors inside `ProtocolParametersConversionError` were all defined outside of the package, so we left them untouched.
But when dealing with errors within errors within inside `cardano-api`, we would remove the recursive call from the pretty-printer of the error, and we would pass the inner error to `mkTestWithCause` instead.

We can still keep the nested `ErrorContent` if we want, to be able to access it from code that may handle the error.
It won't consume any additional space, since Haskell treats it is as a reference.

We can also inspect `ErrorContent` inside `Error`s by just using the `getErrorContent` function.

# Conclusion

The proposed approach ensures every error in `cardano-api` has a stack-trace associated, and it provides a way of displaying all the layers of errors in a readable way, without losing any of the information that is currently available, and altering the existing code very minimally.

On the other hand, it does require a big refactoring, and it complicates the error types a little.


