# Status

📜 Proposed 2023-09-04

# Context

The Cardano block chain can be hard forked through multiple eras throughout its existence.  CLI commands may behave differently between eras depending on what the current era is and what era the user runs the command in.

The era to be used for a CLI command can be optional in which case the command may connect to the node to find out what the current era is and choose an era appropriately.

For these cases, the question arises what the default era should be for the CLI command.

Moreover there is the question of how the default era should change over time, particularly in relation to a hard fork: whether the default era should change in the release before or after the hard fork happens.

There is even scope to adopt a different strategy between query and transaction commands or even between different transaction commands.

For example we can create a transaction in two different ways using the CLI - using `transaction build` and `transaction build-raw` commands.  The `transaction build` command is "magical" in that it tries to be user friendly by inferring transaction parameters wherever it can so the user doesn't have to, while the `transaction build-raw` is for more experienced users because it provides no inference and leaves it up to the user to supply all the necessary transaction parameters.

Would it make sense for "magical" commands like the `transaction build` command to try to be more helpful and use the current era as the default?  That would _seem_ to be in the spirit of the command.

Historically we changed the default transactions era in the first release after the hard fork in order to give the tool/dapp developers the possibility to update the default transaction era in their code after the hard fork - otherwise their tools would stop working after the hard fork.

Care needs to be taken to avoid making a CLI release that does not work by default by defaulting to using an era that is not yet active. So that's why we have historically only updated the default era in the release after the hard fork, not the release before.

This way, third party tools that use the CLI would not stop working after the hard fork, they would continue to create and submit transactions in the older format, for the older era. Historically we always made that backwards compatibility work. It just meant that CLI users would not get the new features for the new era without explicitly using the right era flag - until the following release changed the default.

Moreover, it is often the case that CLI commands in the release that crossed the hard fork may not yet be updated to work in the new era.  If we adopted policy of using the current era for our CLI commands, even "magical" commands that try to infer as much as possible, we risk possibility of the command breaking after the hard fork.

This was the context from the adoption of [[ADR-1 Default eras for CLI commands]] until now.

Going foward we have additional concerns to consider:

Some commands require access to the protocol parameters in the era the command is operating in and the contents of the protocol parameters could have meaningful bearing on how any transactions derived from those commands will be interpreted on-chain.

It is important therefore that the protocol parameters that is used contain the sorts of value that reflect the current state of the ledger.

Unfortunately, the upcoming Conway era introduces many new protocol parameters and obsoletes many others, meaning that there is no way to construct sensible values for these protocol parameters that would guarantee that transactions derived from them operate in a predictable way on chain.

This would mean that submitting transactions built for an older era inherently carries with the risk that the transaction behaves other than intended.

This is especially true for transactions built in an older era with an incomplete, made up or out-of-date values.

This is also true for transactions upgraded from an older era, although to a lesser extent.

Constructing a command in the currently era is the for any command that requires protocol-parametes is the only way to guarantee a transaction works as intended.

Another thing that has changed since [[ADR-1 Default eras for CLI commands]] was adopted is the intruction of an era-based command structure.

The era-based command structure looks like this: `cardano-cli selected-era command sub-command ...`.  For example `cardano-cli babbage transaction build ...`.

We will for a time continue to support commands of the structure `cardano-cli command sub-command --selected-era` up until the Babbage era, but for Conway onwards, the era-based command structure must be used.

The era-based command structure means that users when running a command must make some kind of decision upfront about what era they wish to run their command in.

Such a structure also affords the possibility of having era selectors that follow an upgrade policy.  For example the `latest` era may select latest era that the CLI supports as a hard-coded value, defaulting to `babbage` whilst `conway` support is still in development.

This ADR concerns what era selectors we will have and how they will behave.

The decision should as much as possible take into account the following considerations:

* Some users may care that CLI behaviour not suddenly change after a hard fork in a way that can break third party tools and user expectations.
* Some users may care that CLI behaviour be such that transactions always behave as expected on-chain at the cost of breaking at a hard fork.
* Historical precedent so as to avoid surprising SPOs and in a new release.
* Simplicity and consistency.
* Behaviour that just works where it makes sense.

# Decision

There will be no "default" era.  Instead, two era selectors will be included with the CLI:

* `local` - This era is the latest hard-coded era supported by `cardano-cli` at the time `cardano-cli` was released.  This will not change in the event of a
  hard-fork.  This command is offline and can run in the absence of a node socket.  For commands that require protocol parameters, this must be provided by
  the user.  Any commands that are inherently online (eg `transaction submit`) will still use the same era as commands that are not inherently online.
  In the event that the era selects an era different behind that of the network, `cardano-cli` will attempt to automically upgrade transaction to the network
  era unless explicitly instructed not to.  `cardano-cli` maybe be instructed to fail on an era-mismatch with the network era or use some other
  strategy if the need arises to have further strategies.

* `network` - This era is the era of the network the run of the command is intended for.  In order to acquire the era, `cardano-cli` must connect to the node
  via a socket.  This command is online and cannot run in the absence of a node socket.  For commands that require protocol parameters, they will default to
  the one supplied by the node.  If the version of `cardano-cli` used does not support the network's current era, the command will fail.  Because the
  network era can change at a hard fork, this means that a call to a command that worked before the hard fork may fail after a hard fork.  This could happen
  for example if the invocation used a command line option that was valid before the hardfork but is no longer valid after the hard fork.

* `latest` - As an era selector will be removed because it is ambiguous as to what it means.

This obsoletes [[ADR-1 Default eras for CLI commands]].

# Consequences

* The possibility of a hard fork means tradeoffs are necessary with selecting eras.
  There is continues to be a risk that users get surprises after a hard fork.  Providing `local` and `network` selectors will just mean that users will get
  choose what kind of surprises they may get.
* Third party tools will need to consider how to code around an impending hard fork boundary by being aware of what changes are coming and code defensively,
  choosing the appropriate era selector with the behaviour they need to implement their features in a hard-fork tolerant manner.
* Users may or may not necessarily be using the current era by default depending on their choices.
* No default era imposes on the user the burden of either choosing an era or an era selector.
* The decision to have a default era is postponed so we're less likely to make the wrong choice.
