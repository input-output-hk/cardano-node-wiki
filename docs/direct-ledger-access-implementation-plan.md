# Direct Ledger Access for cardano-rpc: Implementation Plan

Related ADR: [ADR-019](./ADR-019-direct-ledger-access-for-cardano-rpc.md)

---

## Overview

Changes span two submodules: **cardano-api** (contains cardano-rpc) and **cardano-node**.
Do cardano-rpc side first (steps 1-8), then cardano-node side (steps 9-12).

---

## Current state analysis

### Key code paths being replaced

All query/submit methods currently follow the same N2C pattern:

```haskell
-- 1. Grab the LocalNodeConnectInfo from the reader env
nodeConnInfo <- grab

-- 2. Determine the current era via N2C
AnyCardanoEra era <- liftIO . throwExceptT $ determineEra nodeConnInfo

-- 3. Execute a local state query expression over N2C
(result, ...) <- liftIO . (throwEither =<<) $
  executeLocalStateQueryExpr nodeConnInfo VolatileTip $ do
    result <- throwEither =<< throwEither =<< queryXxx sbe ...
    chainPoint <- throwEither =<< queryChainPoint
    blockNo <- throwEither =<< queryChainBlockNo
    pure (result, chainPoint, blockNo)
```

This pattern appears in:
- `Query.hs`: `readParamsMethod`, `readUtxosMethod`, `searchUtxosMethod`
- `Submit.hs`: `submitTxMethod` (era detection + `submitTxToNodeLocal`)
- `Node.hs`: `getProtocolParamsJsonMethod` (era detection + query)

### Current environment wiring

```
RpcEnv (Env.hs)
  ├── config :: RpcConfig
  ├── tracer :: Tracer m TraceRpc
  └── rpcLocalNodeConnectInfo :: LocalNodeConnectInfo  ← REPLACE with IORef

Monad.hs:
  instance Has LocalNodeConnectInfo RpcEnv              ← REPLACE
  type MonadRpc: Has LocalNodeConnectInfo e             ← REPLACE

Server.hs:
  runRpcServer :: Tracer IO TraceRpc -> (RpcConfig, NetworkMagic) -> IO ()
    calls mkLocalNodeConnectInfo to build the env       ← REPLACE
```

### Current tracing (Tracing.hs + Rpc.hs)

```
TraceRpcSubmit
  ├── TraceRpcSubmitN2cConnectionError SomeException    ← REMOVE
  ├── TraceRpcSubmitTxDecodingError DecoderError        (keep)
  ├── TraceRpcSubmitTxValidationError ...               (keep)
  └── TraceRpcSubmitSpan TraceSpanEvent                 (keep)
```

The N2C error is wrapped in `Submit.hs` via `tryAny` + `first TraceRpcSubmitN2cConnectionError`.

---

## Step 1 -- New file: `LedgerAccess.hs` in cardano-rpc

**Create:** `cardano-api/cardano-rpc/src/Cardano/Rpc/Server/Internal/LedgerAccess.hs`

```haskell
{-# LANGUAGE GADTs #-}
{-# LANGUAGE RankNTypes #-}

module Cardano.Rpc.Server.Internal.LedgerAccess
  ( LedgerAccess (..)
  , withLedgerAccess
  ) where

import Cardano.Api
import Cardano.Api.Consensus (TxValidationErrorInCardanoMode)
import Cardano.Api.Network.IPC (SubmitResult)
import Cardano.Ledger.Core qualified as Ledger
import Data.IORef (IORef, readIORef)
import Network.GRPC.Spec (GrpcError (GrpcUnavailable), GrpcException (..))

data LedgerAccess = LedgerAccess
  { laDetermineEra  :: IO AnyCardanoEra
  , laQueryChainTip :: IO (ChainPoint, WithOrigin BlockNo)
  , laQueryPParams  :: forall era. ShelleyBasedEra era -> IO (Ledger.PParams (ShelleyLedgerEra era))
  , laQueryUtxo     :: forall era. ShelleyBasedEra era -> QueryUTxOFilter -> IO (UTxO era)
  , laSubmitTx      :: TxInMode -> IO (SubmitResult TxValidationErrorInCardanoMode)
  }

-- | Read the LedgerAccess IORef; throw gRPC UNAVAILABLE if Nothing
-- (kernel not yet initialized).
withLedgerAccess :: IORef (Maybe LedgerAccess) -> (LedgerAccess -> IO a) -> IO a
withLedgerAccess ref f = do
  mla <- readIORef ref
  case mla of
    Nothing -> throwIO GrpcException
      { grpcError         = GrpcUnavailable
      , grpcErrorMessage  = Just "Node kernel not yet initialized"
      , grpcErrorMetadata = []
      }
    Just la -> f la
```

**Notes:**
- `SubmitResult` comes from `Cardano.Api.Network.IPC` (re-exported from ouroboros-network)
- `TxValidationErrorInCardanoMode` comes from `Cardano.Api.Consensus`
- `Ledger.PParams` comes from `cardano-ledger-core` (already a cabal dep)
- Higher-rank `forall era` fields need `RankNTypes`

---

## Step 2 -- Update `Env.hs`: replace `LocalNodeConnectInfo` with `IORef`

**Modify:** `cardano-api/cardano-rpc/src/Cardano/Rpc/Server/Internal/Env.hs`

Current state:
```haskell
module Cardano.Rpc.Server.Internal.Env
  ( RpcEnv (..)
  , mkLocalNodeConnectInfo
  ) where

import Cardano.Api
import Cardano.Rpc.Server.Config
import Cardano.Rpc.Server.Internal.Tracing
import Control.Tracer (Tracer)

data RpcEnv = RpcEnv
  { config :: !RpcConfig
  , tracer :: forall m. MonadIO m => Tracer m TraceRpc
  , rpcLocalNodeConnectInfo :: !LocalNodeConnectInfo
  }

mkLocalNodeConnectInfo :: SocketPath -> NetworkMagic -> LocalNodeConnectInfo
mkLocalNodeConnectInfo nodeSocketPath networkMagic = ...
```

Changes:
- Remove `mkLocalNodeConnectInfo` from exports and definition
- Replace `rpcLocalNodeConnectInfo :: !LocalNodeConnectInfo` with `rpcLedgerAccess :: !(IORef (Maybe LedgerAccess))`
- Replace `import Cardano.Api` with specific imports: `Data.IORef (IORef)`
- Add `import Cardano.Rpc.Server.Internal.LedgerAccess (LedgerAccess)`

New state:
```haskell
module Cardano.Rpc.Server.Internal.Env
  ( RpcEnv (..)
  ) where

import Cardano.Rpc.Server.Config
import Cardano.Rpc.Server.Internal.LedgerAccess (LedgerAccess)
import Cardano.Rpc.Server.Internal.Tracing

import Control.Tracer (Tracer)
import Data.IORef (IORef)
import RIO (MonadIO)

data RpcEnv = RpcEnv
  { config :: !RpcConfig
  , tracer :: forall m. MonadIO m => Tracer m TraceRpc
  , rpcLedgerAccess :: !(IORef (Maybe LedgerAccess))
  }
```

---

## Step 3 -- Update `Monad.hs`: replace `Has` instance + `MonadRpc`

**Modify:** `cardano-api/cardano-rpc/src/Cardano/Rpc/Server/Internal/Monad.hs`

Current state (relevant parts):
```haskell
import Cardano.Api

instance Has LocalNodeConnectInfo RpcEnv where
  obtain RpcEnv{rpcLocalNodeConnectInfo} = rpcLocalNodeConnectInfo

type MonadRpc e m =
  ( Has (Tracer m TraceRpc) e
  , Has LocalNodeConnectInfo e
  , ...
  )
```

Changes:
- Replace `instance Has LocalNodeConnectInfo RpcEnv` with `instance Has (IORef (Maybe LedgerAccess)) RpcEnv`
  obtaining via `rpcLedgerAccess`
- Replace `Has LocalNodeConnectInfo e` in `MonadRpc` with `Has (IORef (Maybe LedgerAccess)) e`
- Replace `import Cardano.Api` with `import Cardano.Api.Era (Inject (..))` (needed by `putTrace`) and
  `import Cardano.Rpc.Server.Internal.LedgerAccess (LedgerAccess)`
- Add `import Data.IORef (IORef)`

New state for the changed parts:
```haskell
import Cardano.Api.Era (Inject (..))
import Cardano.Rpc.Server.Internal.LedgerAccess (LedgerAccess)
import Data.IORef (IORef)

instance Has (IORef (Maybe LedgerAccess)) RpcEnv where
  obtain RpcEnv{rpcLedgerAccess} = rpcLedgerAccess

type MonadRpc e m =
  ( Has (Tracer m TraceRpc) e
  , Has (IORef (Maybe LedgerAccess)) e
  , HasCallStack
  , MonadReader e m
  , MonadUnliftIO m
  )
```

**Verify:** `Inject` is used by `putTrace` (line 59: `inject t'`). Currently comes from
`Cardano.Api` re-export. After removing `Cardano.Api`, import from `Cardano.Api.Era`.

---

## Step 4 -- Update `Tracing.hs`: replace N2C trace constructors

**Modify:** `cardano-api/cardano-rpc/src/Cardano/Rpc/Server/Internal/Tracing.hs`

In `TraceRpcSubmit`:
- **Remove:** `TraceRpcSubmitN2cConnectionError SomeException`
- **Add:** `TraceRpcLedgerAccessUnavailable` (no payload -- startup-window case)
- **Add:** `TraceRpcForkerError String` (forker acquisition failure)

Current:
```haskell
data TraceRpcSubmit
  = TraceRpcSubmitN2cConnectionError SomeException
  | TraceRpcSubmitTxDecodingError DecoderError
  | TraceRpcSubmitTxValidationError TxValidationErrorInCardanoMode
  | TraceRpcSubmitSpan TraceSpanEvent
  deriving Show
```

New:
```haskell
data TraceRpcSubmit
  = TraceRpcLedgerAccessUnavailable
  | TraceRpcForkerError String
  | TraceRpcSubmitTxDecodingError DecoderError
  | TraceRpcSubmitTxValidationError TxValidationErrorInCardanoMode
  | TraceRpcSubmitSpan TraceSpanEvent
  deriving Show
```

Update `Pretty TraceRpcSubmit`:
```haskell
instance Pretty TraceRpcSubmit where
  pretty = \case
    TraceRpcSubmitSpan (SpanBegin _) -> "Started submit method"
    TraceRpcSubmitSpan (SpanEnd _) -> "Finished submit method"
    TraceRpcLedgerAccessUnavailable -> "Ledger access unavailable (node kernel not yet initialized)"
    TraceRpcForkerError e -> "Forker error: " <> pretty e
    TraceRpcSubmitTxDecodingError e -> "Failed to decode transaction: " <> pshow e
    TraceRpcSubmitTxValidationError e -> "Failed to submit transaction: " <> pshow e
```

**Note:** `Control.Exception` import for `SomeException` is still needed because `TraceRpc`
uses it in `TraceRpcError` and `TraceRpcFatalError` constructors defined in this same module.

---

## Step 5 -- Update `Server.hs`: new signature + re-export `LedgerAccess`

**Modify:** `cardano-api/cardano-rpc/src/Cardano/Rpc/Server.hs`

Changes:
- Add `LedgerAccess (..)` to exports
- Change signature:
  ```haskell
  -- OLD:
  runRpcServer :: Tracer IO TraceRpc -> (RpcConfig, NetworkMagic) -> IO ()
  -- NEW:
  runRpcServer :: Tracer IO TraceRpc -> RpcConfig -> IORef (Maybe LedgerAccess) -> IO ()
  ```
- Update body: remove `nodeSocketPath` destructuring, remove `mkLocalNodeConnectInfo` call
- Construct `RpcEnv` with `rpcLedgerAccess = ledgerAccessRef` instead of `rpcLocalNodeConnectInfo`
- Replace `import Cardano.Api` with `import Data.IORef (IORef)`
- Add `import Cardano.Rpc.Server.Internal.LedgerAccess`

Current body:
```haskell
runRpcServer tracer (rpcConfig, networkMagic) = handleFatalExceptions $ do
  let RpcConfig
        { isEnabled = Identity isEnabled
        , rpcSocketPath = Identity (File rpcSocketPathFp)
        , nodeSocketPath = Identity nodeSocketPath
        } = rpcConfig
      ...
      rpcEnv = RpcEnv
        { config = rpcConfig
        , tracer = natTracer liftIO tracer
        , rpcLocalNodeConnectInfo = mkLocalNodeConnectInfo nodeSocketPath networkMagic
        }
```

New body:
```haskell
runRpcServer tracer rpcConfig ledgerAccessRef = handleFatalExceptions $ do
  let RpcConfig
        { isEnabled = Identity isEnabled
        , rpcSocketPath = Identity (File rpcSocketPathFp)
        } = rpcConfig
      ...
      rpcEnv = RpcEnv
        { config = rpcConfig
        , tracer = natTracer liftIO tracer
        , rpcLedgerAccess = ledgerAccessRef
        }
```

---

## Step 6 -- Rewrite query/submit/node methods to use `LedgerAccess`

### `Query.hs`

All 3 methods (`readParamsMethod`, `readUtxosMethod`, `searchUtxosMethod`) follow the same
transformation:

```haskell
-- OLD pattern (each method):
nodeConnInfo <- grab
AnyCardanoEra era <- liftIO . throwExceptT $ determineEra nodeConnInfo
...
(result, chainPoint, blockNo) <- liftIO . (throwEither =<<) $
  executeLocalStateQueryExpr nodeConnInfo target $ do ...

-- NEW pattern:
laRef <- grab @(IORef (Maybe LedgerAccess))
(result, chainPoint, blockNo, eon) <- liftIO $ withLedgerAccess laRef $ \la -> do
  AnyCardanoEra era <- laDetermineEra la
  eon <- forEraInEon @Era era (error "Minimum Conway era required") pure
  let sbe = convert eon
  result <- laQueryPParams la sbe  -- or laQueryUtxo la sbe filter
  (chainPoint, blockNo) <- laQueryChainTip la
  pure (result, chainPoint, blockNo, eon)
```

The rest of each method (protobuf conversion, pagination, filtering) stays unchanged.

Import changes:
- **Add:** `Cardano.Rpc.Server.Internal.LedgerAccess`
- **Remove unused:** `throwExceptT` (from Error module), `executeLocalStateQueryExpr`,
  `queryProtocolParameters`, `queryChainPoint`, `queryChainBlockNo`, `queryUtxo`,
  `VolatileTip` -- these were all from `Cardano.Api`

#### `readParamsMethod` -- detailed

Current (lines 42-61):
```haskell
readParamsMethod _req = do
  nodeConnInfo <- grab
  AnyCardanoEra era <- liftIO . throwExceptT $ determineEra nodeConnInfo
  eon <- forEraInEon @Era era (error "Minimum Conway era required") pure
  let sbe = convert eon
  let target = VolatileTip
  (pparams, chainPoint, blockNo) <- liftIO . (throwEither =<<) $
    executeLocalStateQueryExpr nodeConnInfo target $ do
      pparams <- throwEither =<< throwEither =<< queryProtocolParameters sbe
      chainPoint <- throwEither =<< queryChainPoint
      blockNo <- throwEither =<< queryChainBlockNo
      pure (pparams, chainPoint, blockNo)
  pure $ ...
```

New:
```haskell
readParamsMethod _req = do
  laRef <- grab @(IORef (Maybe LedgerAccess))
  (pparams, chainPoint, blockNo, eon) <- liftIO $ withLedgerAccess laRef $ \la -> do
    AnyCardanoEra era <- laDetermineEra la
    eon <- forEraInEon @Era era (error "Minimum Conway era required") pure
    let sbe = convert eon
    pparams <- laQueryPParams la sbe
    (chainPoint, blockNo) <- laQueryChainTip la
    pure (pparams, chainPoint, blockNo, eon)
  pure $ ...  -- same protobuf conversion using eon, pparams, chainPoint, blockNo
```

#### `readUtxosMethod` -- detailed

Current (lines 73-82):
```haskell
  nodeConnInfo <- grab
  AnyCardanoEra era <- liftIO . throwExceptT $ determineEra nodeConnInfo
  eon <- forEraInEon @Era era (error "Minimum Conway era required") pure
  let target = VolatileTip
  (utxo, chainPoint, blockNo) <- liftIO . (throwEither =<<) $
    executeLocalStateQueryExpr nodeConnInfo target $ do
      utxo <- throwEither =<< throwEither =<< queryUtxo (convert eon) utxoFilter
      chainPoint <- throwEither =<< queryChainPoint
      blockNo <- throwEither =<< queryChainBlockNo
      pure (utxo, chainPoint, blockNo)
```

New:
```haskell
  laRef <- grab @(IORef (Maybe LedgerAccess))
  (utxo, chainPoint, blockNo, eon) <- liftIO $ withLedgerAccess laRef $ \la -> do
    AnyCardanoEra era <- laDetermineEra la
    eon <- forEraInEon @Era era (error "Minimum Conway era required") pure
    let sbe = convert eon
    utxo <- laQueryUtxo la sbe utxoFilter
    (chainPoint, blockNo) <- laQueryChainTip la
    pure (utxo, chainPoint, blockNo, eon)
```

#### `searchUtxosMethod` -- detailed

Current (lines 108-117):
```haskell
  nodeConnInfo <- grab
  AnyCardanoEra era <- liftIO . throwExceptT $ determineEra nodeConnInfo
  eon <- forEraInEon @Era era (error "Minimum Conway era required") pure
  let target = VolatileTip
  (utxo, chainPoint, blockNo) <- liftIO . (throwEither =<<) $
    executeLocalStateQueryExpr nodeConnInfo target $ do
      utxo <- throwEither =<< throwEither =<< queryUtxo (convert eon) utxoFilter
      chainPoint <- throwEither =<< queryChainPoint
      blockNo <- throwEither =<< queryChainBlockNo
      pure (utxo, chainPoint, blockNo)
```

New:
```haskell
  laRef <- grab @(IORef (Maybe LedgerAccess))
  (utxo, chainPoint, blockNo, eon) <- liftIO $ withLedgerAccess laRef $ \la -> do
    AnyCardanoEra era <- laDetermineEra la
    eon <- forEraInEon @Era era (error "Minimum Conway era required") pure
    let sbe = convert eon
    utxo <- laQueryUtxo la sbe utxoFilter
    (chainPoint, blockNo) <- laQueryChainTip la
    pure (utxo, chainPoint, blockNo, eon)
```

### `Submit.hs`

Current `submitTxMethod` (lines 36-66):
```haskell
submitTxMethod req = do
  nodeConnInfo <- grab
  AnyCardanoEra era <- liftIO . throwExceptT $ determineEra nodeConnInfo
  eon <- forEraInEon era (error "Minimum Shelley era required") pure
  tx <- putTraceThrowEither
    . first TraceRpcSubmitTxDecodingError
    . deserialiseTx eon
    $ req ^. U5c.tx . U5c.raw
  txId' <- submitTx eon tx
  pure $ def & U5c.ref .~ serialiseToRawBytes txId'
 where
  ...
  submitTx sbe tx = do
    nodeConnInfo <- grab
    putTraceThrowEither . join . first TraceRpcSubmitN2cConnectionError
      =<< tryAny
        ( submitTxToNodeLocal nodeConnInfo (TxInMode sbe tx) >>= \case
            Net.Tx.SubmitFail reason -> pure . Left $ TraceRpcSubmitTxValidationError reason
            Net.Tx.SubmitSuccess -> pure . Right $ getTxId $ getTxBody tx
        )
```

New:
```haskell
submitTxMethod req = do
  laRef <- grab @(IORef (Maybe LedgerAccess))
  AnyCardanoEra era <- liftIO $ withLedgerAccess laRef laDetermineEra
  eon <- forEraInEon era (error "Minimum Shelley era required") pure
  tx <- putTraceThrowEither
    . first TraceRpcSubmitTxDecodingError
    . deserialiseTx eon
    $ req ^. U5c.tx . U5c.raw
  txId' <- submitTx laRef eon tx
  pure $ def & U5c.ref .~ serialiseToRawBytes txId'
 where
  ...
  submitTx laRef sbe tx = do
    result <- liftIO $ withLedgerAccess laRef $ \la ->
      laSubmitTx la (TxInMode sbe tx)
    case result of
      Net.Tx.SubmitFail reason ->
        putTraceThrowEither . Left $ TraceRpcSubmitTxValidationError reason
      Net.Tx.SubmitSuccess ->
        pure $ getTxId $ getTxBody tx
```

- Remove `tryAny` + `first TraceRpcSubmitN2cConnectionError` wrapping (N2C errors no longer possible)
- Keep `Net.Tx.SubmitFail`/`Net.Tx.SubmitSuccess` patterns (still needed)

### `Node.hs`

Current `getEraMethod` (line 35):
```haskell
getEraMethod _ = pure . Proto $ defMessage & Rpc.era .~ Rpc.Conway
```

This is currently hardcoded. Replace with dynamic era detection:
```haskell
getEraMethod _ = do
  laRef <- grab @(IORef (Maybe LedgerAccess))
  AnyCardanoEra era <- liftIO $ withLedgerAccess laRef laDetermineEra
  let rpcEra = case era of
        ConwayEra -> Rpc.Conway
        _         -> Rpc.Conway  -- fallback, extend when new eras are added
  pure . Proto $ defMessage & Rpc.era .~ rpcEra
```

Current `getProtocolParamsJsonMethod` (lines 38-54):
```haskell
getProtocolParamsJsonMethod _ = do
  nodeConnInfo <- grab
  AnyCardanoEra era <- liftIO . throwExceptT $ determineEra nodeConnInfo
  eon <- forEraInEon @Era era (...) pure
  let sbe = convert eon
  let target = VolatileTip
  pparams <- liftIO . (throwEither =<<) $
    executeLocalStateQueryExpr nodeConnInfo target $
      throwEither =<< throwEither =<< queryProtocolParameters sbe
  ...
```

New:
```haskell
getProtocolParamsJsonMethod _ = do
  laRef <- grab @(IORef (Maybe LedgerAccess))
  (pparams, eon) <- liftIO $ withLedgerAccess laRef $ \la -> do
    AnyCardanoEra era <- laDetermineEra la
    eon <- forEraInEon @Era era (error "...") pure
    let sbe = convert eon
    pparams <- laQueryPParams la sbe
    pure (pparams, eon)
  ...
```

Import changes:
- **Remove:** `Cardano.Rpc.Server.Internal.Error` (no more `throwExceptT`, `throwEither`)
- **Add:** `Cardano.Rpc.Server.Internal.LedgerAccess`

---

## Step 7 -- Config.hs: no changes

`nodeSocketPath` stays in `RpcConfig` because:
1. It's needed to derive the default `rpcSocketPath` in `makeRpcConfig`
2. `nodeSocketPathToRpcSocketPath` is imported by:
   - `cardano-node/cardano-testnet/src/Testnet/Types.hs` (line 155: `nodeRpcSocketPath`)
   - `cardano-node/cardano-node/src/Cardano/Node/Configuration/POM.hs`

The field just isn't used at runtime for N2C connections anymore.

---

## Step 8 -- Update `cardano-rpc.cabal`

**Modify:** `cardano-api/cardano-rpc/cardano-rpc.cabal`

Add `Cardano.Rpc.Server.Internal.LedgerAccess` to `exposed-modules` (after `Error`, line 58):

```
    Cardano.Rpc.Server.Internal.Error
    Cardano.Rpc.Server.Internal.LedgerAccess    -- NEW
    Cardano.Rpc.Server.Internal.Monad
```

No new build-depends needed: `cardano-ledger-core`, `grpc-spec` already listed.

---

## Step 9 -- Create `mkLedgerAccess` in cardano-node

**Create:** `cardano-node/cardano-node/src/Cardano/Node/Rpc/LedgerAccess.hs`

```haskell
module Cardano.Node.Rpc.LedgerAccess (mkLedgerAccess) where

mkLedgerAccess
  :: NodeKernel IO RemoteAddress LocalConnectionId (CardanoBlock StandardCrypto)
  -> LedgerAccess
```

### Implementation strategy

All fields share a helper that acquires a read-only forker, runs `answerQuery`, closes forker:

```haskell
runQuery :: QueryInMode result -> IO result
runQuery qim = withRegistry $ \reg -> do
  let chainDB = getChainDB nk
      cfg     = ExtLedgerCfg (getTopLevelConfig nk)
  forker <- getReadOnlyForkerAtPoint chainDB reg VolatileTip >>= \case
    Left err -> throwIO (userError $ "Forker error: " <> show err)
    Right f  -> pure f
  let Some cQuery = toConsensusQuery qim
  result <- answerQuery cfg forker cQuery
  roforkerClose forker
  pure $ fromConsensusQueryResult qim cQuery result
```

### Per-field implementation

**`laDetermineEra`:** `runQuery QueryCurrentEra` then wrap in `AnyCardanoEra`.

**`laQueryChainTip`:** `runQuery QueryChainPoint` and `runQuery QueryChainBlockNo`.
Or combine into one forker acquisition with two `answerQuery` calls for efficiency.

**`laQueryPParams`:**
```haskell
\sbe -> runQuery (QueryInEra (QueryInShelleyBasedEra sbe QueryProtocolParameters))
```
Unwrap the `Either EraMismatch` result (throw on mismatch).

**`laQueryUtxo`:**
```haskell
\sbe filter -> runQuery (QueryInEra (QueryInShelleyBasedEra sbe (QueryUTxO filter)))
```
Same era-mismatch handling.

**`laSubmitTx`:**
```haskell
\txInMode -> do
  let genTx = toConsensusGenTx txInMode
  result <- addLocalTxs (getMempool nk) (Solo genTx)
  case Solo.getSolo result of
    MempoolTxAdded _        -> pure SubmitSuccess
    MempoolTxRejected _ err -> pure (SubmitFail (fromConsensusApplyTxErr err))
```

### Key imports needed

```haskell
import Cardano.Api (...)
import Cardano.Api.Query.Internal.Type.QueryInMode (toConsensusQuery, fromConsensusQueryResult)
import Cardano.Api.Consensus.Internal.InMode (toConsensusGenTx, fromConsensusApplyTxErr)
import Cardano.Rpc.Server.Internal.LedgerAccess (LedgerAccess (..))
import Ouroboros.Consensus.Ledger.Query (answerQuery)
import Ouroboros.Consensus.Storage.ChainDB qualified as ChainDB
import Ouroboros.Consensus.Mempool.API (addLocalTxs, MempoolAddTxResult (..))
import Ouroboros.Consensus.Node (NodeKernel (..))
import Ouroboros.Consensus.Ledger.Extended (ExtLedgerCfg (..))
import Ouroboros.Consensus.Block (Target (..))
import Ouroboros.Consensus.Util.ResourceRegistry (withRegistry)
```

---

## Step 10 -- Wire up in `Run.hs`

**Modify:** `cardano-node/cardano-node/src/Cardano/Node/Run.hs` (~line 558)

Current:
```haskell
    withAsync (runRpcServer (rpcTracer tracers) (ncRpcConfig nc, networkMagic))  $ \_ ->
        Node.run
          nodeArgs {
              rnNodeKernelHook = \registry nodeKernel -> do
                -- reinstall `SIGHUP` handler
                installSigHUPHandler ...
                rnNodeKernelHook nodeArgs registry nodeKernel
          }
```

New:
```haskell
    ledgerAccessRef <- newIORef Nothing

    withAsync (runRpcServer (rpcTracer tracers) (ncRpcConfig nc) ledgerAccessRef) $ \_ ->
        Node.run
          nodeArgs {
              rnNodeKernelHook = \registry nodeKernel -> do
                -- populate the LedgerAccess so the RPC server can serve requests
                writeIORef ledgerAccessRef (Just (mkLedgerAccess nodeKernel))
                -- reinstall `SIGHUP` handler
                installSigHUPHandler ...
                rnNodeKernelHook nodeArgs registry nodeKernel
          }
```

Import additions:
- `import Cardano.Node.Rpc.LedgerAccess (mkLedgerAccess)`
- `import Data.IORef (newIORef, writeIORef)`

---

## Step 11 -- Update cardano-node tracing (`Rpc.hs`)

**Modify:** `cardano-node/cardano-node/src/Cardano/Node/Tracing/Tracers/Rpc.hs`

### `LogFormatting TraceRpc` (`forMachine`)

Replace:
```haskell
TraceRpcSubmitN2cConnectionError _ -> []
```

With:
```haskell
TraceRpcLedgerAccessUnavailable -> []
TraceRpcForkerError _ -> []
```

Add missing `SearchUtxos` span handling:
```haskell
TraceRpcQuerySearchUtxosSpan s ->
  [ "queryName" .= String "SearchUtxos"
  , spanToObject s
  ]
```

### `asMetrics`

Add:
```haskell
TraceRpcQuery (TraceRpcQuerySearchUtxosSpan (SpanBegin _)) ->
  [CounterM "rpc.request.QueryService.SearchUtxos" Nothing]
```

### `MetaTrace TraceRpc`

**`namespaceFor`:** Replace:
```haskell
TraceRpcSubmitN2cConnectionError _ -> ["N2cConnectionError"]
```
With:
```haskell
TraceRpcLedgerAccessUnavailable -> ["LedgerAccessUnavailable"]
TraceRpcForkerError _ -> ["ForkerError"]
```

Add:
```haskell
TraceRpcQuerySearchUtxosSpan _ -> ["SearchUtxos", "Span"]
```

**`severityFor`:** Replace:
```haskell
["SubmitService", "N2cConnectionError"] -> Just Warning
```
With:
```haskell
["SubmitService", "LedgerAccessUnavailable"] -> Just Warning
["SubmitService", "ForkerError"] -> Just Warning
```

Add:
```haskell
["QueryService", "SearchUtxos", "Span"] -> Just Debug
```

**`documentFor`:** Replace the N2cConnectionError entry with:
```haskell
["SubmitService", "LedgerAccessUnavailable"] ->
  Just "Ledger access not yet available. Node kernel is still initializing."
["SubmitService", "ForkerError"] ->
  Just "Error acquiring a forker from the ChainDB."
```

Add:
```haskell
["QueryService", "SearchUtxos", "Span"] -> Just "Span for the SearchUtxos UTXORPC method."
```

**`metricsDocFor`:** Add:
```haskell
["QueryService", "SearchUtxos", "Span"] ->
  [("rpc.request.QueryService.SearchUtxos", "Span for the SearchUtxos UTXORPC method.")]
```

**`allNamespaces`:** Replace `["SubmitService", "N2cConnectionError"]` with:
```haskell
, ["SubmitService", "LedgerAccessUnavailable"]
, ["SubmitService", "ForkerError"]
```

Add:
```haskell
, ["QueryService", "SearchUtxos", "Span"]
```

---

## Step 12 -- Update cardano-node cabal

**Modify:** `cardano-node/cardano-node/cardano-node.cabal`

Add `Cardano.Node.Rpc.LedgerAccess` to exposed-modules (after `Cardano.Node.Queries`):

```
                        Cardano.Node.Queries
                        Cardano.Node.Rpc.LedgerAccess    -- NEW
                        Cardano.Node.Run
```

Verify `resource-registry` is already a dependency (it is, line 202).

---

## Step 13 -- Integration tests (minimal changes expected)

Tests use the gRPC client API which is unchanged. The only thing that could need updating:

- **`Testnet/Types.hs`**: `nodeRpcSocketPath` uses `nodeSocketPathToRpcSocketPath` which we kept -- no change.
- **RPC test files**: Use `Rpc.withConnection` to connect to the gRPC socket -- no change.
- The testnet starts a real cardano-node, so `Run.hs` wiring picks up automatically.

No test file changes expected.

---

## File summary

### New files (2)

| File | Purpose |
|------|---------|
| `cardano-api/cardano-rpc/src/Cardano/Rpc/Server/Internal/LedgerAccess.hs` | `LedgerAccess` record + `withLedgerAccess` |
| `cardano-node/cardano-node/src/Cardano/Node/Rpc/LedgerAccess.hs` | `mkLedgerAccess` from `NodeKernel` |

### Modified files (9)

| File | Change |
|------|--------|
| `cardano-api/cardano-rpc/src/Cardano/Rpc/Server.hs` | New signature, re-export `LedgerAccess`, IORef wiring |
| `cardano-api/cardano-rpc/src/Cardano/Rpc/Server/Internal/Env.hs` | `RpcEnv` holds `IORef`, drop `LocalNodeConnectInfo` |
| `cardano-api/cardano-rpc/src/Cardano/Rpc/Server/Internal/Monad.hs` | `Has` instance + `MonadRpc` update |
| `cardano-api/cardano-rpc/src/Cardano/Rpc/Server/Internal/Tracing.hs` | Replace N2C trace with ledger-access traces |
| `cardano-api/cardano-rpc/src/Cardano/Rpc/Server/Internal/UtxoRpc/Query.hs` | Rewrite 3 methods |
| `cardano-api/cardano-rpc/src/Cardano/Rpc/Server/Internal/UtxoRpc/Submit.hs` | Rewrite submit |
| `cardano-api/cardano-rpc/src/Cardano/Rpc/Server/Internal/Node.hs` | Rewrite era + pparams methods |
| `cardano-api/cardano-rpc/cardano-rpc.cabal` | Add `LedgerAccess` module |
| `cardano-node/cardano-node/src/Cardano/Node/Run.hs` | IORef + writeIORef in kernel hook |
| `cardano-node/cardano-node/src/Cardano/Node/Tracing/Tracers/Rpc.hs` | Update trace instances |
| `cardano-node/cardano-node/cardano-node.cabal` | Add new module |

### Unchanged

| File | Why |
|------|-----|
| `cardano-rpc/src/Cardano/Rpc/Server/Config.hs` | `nodeSocketPath` still needed for socket path derivation |
| `cardano-rpc/src/Cardano/Rpc/Server/Internal/UtxoRpc/Type.hs` | Conversion code untouched |
| `cardano-rpc/src/Cardano/Rpc/Server/Internal/UtxoRpc/Predicate.hs` | Filtering untouched |
| Testnet test files | gRPC client API unchanged |

---

## Verification

1. `nix build .#cardano-rpc` in cardano-api submodule
2. `nix build .#cardano-rpc:test:cardano-rpc-test` -- unit tests
3. `nix build .#cardano-node` in cardano-node submodule
4. Integration tests via testnet (if available)

---

## Risks and mitigations

| Risk | Mitigation |
|------|------------|
| `answerQuery` API differs from expected signature | Verify exact types in ouroboros-consensus 0.30 before implementing step 9 |
| Forker lifecycle -- leaking forkers on exceptions | Use `bracket` pattern: `bracket (acquire) roforkerClose (\f -> answerQuery ...)` |
| `QueryCurrentEra` not available via `toConsensusQuery` | Verify; may need to use `Ouroboros.Consensus.HardFork.Combinator.Ledger.Query` directly |
| `addLocalTxs` API changed in recent consensus | Check `Ouroboros.Consensus.Mempool.API` exports in the pinned version |
| Concurrent reads during kernel initialization race | `IORef` write in `rnNodeKernelHook` happens before any RPC can succeed; `withLedgerAccess` checks atomically |
