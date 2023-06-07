## What repositories should and shouldn't use `git submodule`?

The official way to integrate multiple components in Cardano software is to
maintain the code in `git` as separate repositories and publish releases to
[Cardano Haskell Packages (CHaP)](https://github.com/input-output-hk/cardano-haskell-packages).

All published components must operator in this manner.

In other to streamline development within teams however, teams may opt-in to
use submodules to integrate some subset of components that they deal with
frequently.

Some ground rules are in place however to ensure that all components remain
CHaP compatible.

* Any repository that uses `git submodule` must:
  * Not contain any component code directly.  Component code must only be
    referenced via submodules.  In this sense, repositories that use
    `git submodule` are not projects of themselves.  They are "meta-projects"
    that are merely collections of other projects.
  * Be owned by a particular team.  There is no shared ownership of such
    repositories.
  * Not be used to make released artifacts directly or indirectly.  This is to
    reduce accidental inclusion of anything `git submodule` related into the
    release project.  `git submodule` are not a first class method to
    build software within IOG.
  * Not have a name that makes it sound like official software or tie itself
    to one.  For example `node-and-cli-team` is not an acceptable name because
    it suggests it is related to `cardano-node` or `cardano-cli`.  This is to
    prevent users from accidently believing the meta-project is in anyway
    official and ask for support.

## Projects using `git submodule`

Presently the following meta-projects use `git submodule`:

* [fusion-flamingo](https://github.com/input-output-hk/fusion-flamingo)

## Cloning a repository that uses `git submodule`

Clone the meta-project like this to also initialise the submodules within the
meta-project:

```bash
$ git clone --recurse-submodules git@github.com:input-output-hk/<project-name>.git
```

If you accidentally cloned the meta-project using regular clone:

```bash
$ git clone git@github.com:input-output-hk/<project-name>.git
```

you can initialise the submodules afterwards:

```bash
<project-name> $ git submodule update --init --recursive
```

## Branches and `git submodule`

`git submodule` does not have support for tracking branches.  The feature
deals directly with commits and when changes for a meta-project are pushed
the changes will refer to specific commits for each submodule.

You may nevertheless use tracking branches by switching to them in each
submodule manually.

One problem you may experience however is that whenever you switch branches
or commits at the meta-project level, the submodules will revert to reference
specific commits rather than tracking branches.

All is not lost however.  Even though `git submodule` doesn't support tracking
branches directly, they may be configured to have a branch:

```bash
$ git submodule set-branch --branch <branch-name> <submodule-name>
```

This will configure the specified submodule to use the specified branch.

This branch configuration is fairly crude however.  When you switch git references
at the meta-project level, you will continue have your submodule switch to
reference commits instead of branches.

A script is available in [cardano-dev](https://github.com/input-output-hk/cardano-dev)
to deal with this.  Whilst it cannot keep your submodules pointing at the relevant
branch it can restore your submodules to their configured branches with a simple
invocation.

Here is an end-to-end example:

```bash
$ git clone git@github.com:input-output-hk/fusion-flamingo.git fusion-flamingo-2
$ cd fusion-flamingo-2
$ git submodule update --init --recursive
$ git checkout newhoggy/test-submodules
$ cat .gitmodules
[submodule "cardano-api"]
	path = cardano-api
	url = git@github.com:input-output-hk/cardano-api.git
	branch = newhoggy/use-tag-script-from-cardano-dev-instead
[submodule "cardano-cli"]
	path = cardano-cli
	url = git@github.com:input-output-hk/cardano-cli.git
	branch = main
```

But the submodules are on specific commits:

```bash
$ (cd cardano-api; git status)
HEAD detached at 6f3ce029e
nothing to commit, working tree clean
$ (cd cardano-cli; git status)
HEAD detached at 8399c472f
nothing to commit, working tree clean
```

Running the script puts the submodules on the branches instead:

```bash
$ ../cardano-dev/scripts/switch-submodule-branch.sh
Switching submodule cardano-api to use branch newhoggy/use-tag-script-from-cardano-dev-instead
Switching submodule cardano-cli to use branch main
$ (cd cardano-api; git status)
On branch newhoggy/use-tag-script-from-cardano-dev-instead
Your branch is up to date with 'origin/newhoggy/use-tag-script-from-cardano-dev-instead'.

nothing to commit, working tree clean
$ (cd cardano-cli; git status)
On branch main
Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean
```
