# Status

Proposed.

# Context

The Cardano block chain can be upgraded through multiple eras throughout its existence.  CLI commands may behave differently between eras depending on what the current era is and what era the user runs the command in.

CLI commands that connect to the node will be able to find out what the current era is and the era to be used for the command can be optional.

For these cases, the question arises what the default era should be for the command.

Moreover there is the question of how and the default era should change over time, particularly in relation to a hard fork: whether the default era should change before and after the hard fork happens.

There is even scope to adopt a different strategy between query and transaction commands or even between different transaction commands.

For example we can create a transaction in 2 different ways (using the node CLI) - using `transaction build` and `transaction build-raw` commands.  The `transaction build` command is "magic" in that it does almost anything automatically, while the `transaction build-raw` is for the more experienced users and gives them options to do whatever they want.

In the context of the above commands we need to decide what the default transactions era will be in relation to the hard fork.  For the `transaction build` command it would make sense for the default to the current era - as it is a magic command that should set the transaction era as default too.

