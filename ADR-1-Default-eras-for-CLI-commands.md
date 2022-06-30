# Status

Proposed.

# Context

The Cardano block chain can be upgraded through multiple eras throughout its existence.  CLI commands may behave differently between eras depending on what the current era is and what era the user runs the command in.

CLI commands that connect to the node will be able to find out what the current era is and the era to be used for the command can be optional.

For these cases, the question arises what the default era should be for the command.

Moreover there is the question of how and the default era should change over time, particularly in relation to a hard fork: whether the default era should change before and after the hard fork happens.

There is even scope to adopt a different strategy between query and transaction commands or even between different transaction commands.

For example we can create a transaction in 2 different ways (using the node CLI) - using `transaction build` and `transaction build-raw` commands.  The `transaction build` command is "magical" in that it does almost anything automatically, while the `transaction build-raw` is for the more experienced users and gives them options to do whatever they want.

We need to decide what the default era will be in relation to the hard fork.  For the `transaction build` command it would make sense for the default to the current era - as it is a magic command that should set the transaction era as default too.

Historically we changed the default transactions era in the first release after the hard fork in order to give the tool/dapp developers the possibility to update the default transaction era in their code after the hard fork - otherwise their tools would stop working after the hard fork.

We don't want to make a CLI release that does not work by default by defaulting to using an era that is not yet active. So that's why we do it the release after the Hard Fork, not the release before.

Third party tools that use the CLI would not stop working after the HF, they would continue to create and submit transactions in the older format, for the older era. Historically we always made that backwards compatibility work. It just meant that CLI users would not get the new features for the new era without explicitly using the right era flag - until the following release changed the default.

Moreover, it is often the case that CLI commands in the release that crossed the hard fork may not yet be updated to work in the new era.  If we adopted policy of using the current era for our CLI commands, even "magical" commands that try to infer as much as possible, by using the current era we open ourselves to the possibility of the command breaking after the hard fork.

The decision should as much as possible take into account the following considerations:

* CLI behaviour should not suddenly change after a hard fork in a way that can break third party tools.
* Historical precedent so as to avoid surprising SPOs and in a new release.
* Simplicity, ease of use and tools that just work

# Decision

A given release should use the current era up until the latest era supported by the Mainnet blockchain as at the point of release.

For example, given the hard fork from Alonzo to Babbage:

* All era-sensitive CLI commands should have a default era.

* All era-sensitive CLI commands in release immediately before the hard fork should use the current era up to the Alonzo era only and only use the Babbage era if it were explicitly specified.

* All era-sensitive CLI commands in release immediately after the hard fork should use the current era up until Babbage era.

# Consequences

CLI commands that for a given release that worked prior to a hard fork would continue to work after a hard fork.

Users will not be surprised by changes in behaviour and third party tools will continue to work after the hard fork.

Users will a reasonably recent era by default, if not always the current era.