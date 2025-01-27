# Status
- [ ] Adopted YYYY/MM/DD

# Context
- In `cardano-cli` We have the (anti-)pattern `ExceptT someError IO someResult` 292 times in our codebase.
![image](https://github.com/user-attachments/assets/f405dfca-7d75-404c-b291-3d5140fc8093)

- The vast majority of these errors are not used to recover, they are propagated and reported to the user.
```
main :: IO ()
main = toplevelExceptionHandler $ do
  Crypto.cryptoInit

  envCli <- getEnvCli

  GHC.mkTextEncoding "UTF-8" >>= GHC.setLocaleEncoding
#ifdef UNIX
  _ <- setFileCreationMask (otherModes `unionFileModes` groupModes)
#endif
  co <- Opt.customExecParser pref (opts envCli)

  orDie (docToText . renderClientCommandError) $ runClientCommand co
...
...
...

runClientCommand :: ClientCommand -> ExceptT ClientCommandErrors IO ()
runClientCommand = \case
  AnyEraCommand cmds ->
    firstExceptT (CmdError (renderAnyEraCommand cmds)) $ runAnyEraCommand cmds
  AddressCommand cmds ->
    firstExceptT AddressCmdError $ runAddressCmds cmds
  NodeCommands cmds ->
    runNodeCmds cmds
      & firstExceptT NodeCmdError
  ByronCommand cmds ->
    firstExceptT ByronClientError $ runByronClientCommand cmds
  CompatibleCommands cmd ->
    firstExceptT (BackwardCompatibleError (renderAnyCompatibleCommand cmd)) $
      runAnyCompatibleCommand cmd
  HashCmds cmds ->
    firstExceptT HashCmdError $ runHashCmds cmds
  KeyCommands cmds ->
    firstExceptT KeyCmdError $ runKeyCmds cmds
  LegacyCmds cmds ->
    firstExceptT (CmdError (renderLegacyCommand cmds)) $ runLegacyCmds cmds
  QueryCommands cmds ->
    firstExceptT QueryCmdError $ runQueryCmds cmds
  CliPingCommand cmds ->
    firstExceptT PingClientError $ runPingCmd cmds
  CliDebugCmds cmds ->
    firstExceptT DebugCmdError $ runDebugCmds cmds
  Help pprefs allParserInfo ->
    runHelp pprefs allParserInfo
  DisplayVersion ->
    runDisplayVersion
```
- As a result we have a lot of errors wrapped in errors which makes the code unwieldly and difficult to compose with other code blocks. See image below of incidences of poor composability where we use `firstExceptT` (and sometimes `first`) to wrap errors in other errors.
![image](https://github.com/user-attachments/assets/6a50e8f2-9140-4b7e-afa5-ff778d696742)
 
- The ExceptT IO is a known anti-pattern for these reasons and others as per:
  - https://github.com/haskell-effectful/effectful/blob/master/transformers.md
  - https://tech.fpcomplete.com/blog/2016/11/exceptions-best-practices-haskell/


# Decision
- Error handling: All errors will be converted to exceptions that will be caught by a single exception handler at the top level.
- Top level monad: [RIO](https://hackage.haskell.org/package/rio-0.1.22.0/docs/RIO.html#t:RIO). See the [ReaderT design pattern](https://tech.fpcomplete.com/blog/2017/06/readert-design-pattern/).

# Consequences
- This should dramatically improve our code's composability and remove many unnecessary error types. 
- Readability concerning what errors can be thrown will be negatively impacted. However `ExceptT` already lies about what exceptions can be thrown because it is not limited to the error type stated in `ExceptT`'s type signature. In other words `IO` can implicitly throw other `Exception`s. 
- Initially this will be adopted under the "compatible" group of commands so `cardano-cli` will have a design split temporarily. Once we are happy with the result we will propagate to the rest of `cardano-cli`