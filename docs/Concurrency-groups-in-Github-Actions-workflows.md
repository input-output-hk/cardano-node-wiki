
Github merge-queues provide a way to merge PRs in a way that guarantees the `main`
branch of the project continues to build in CI.

Github concurrency provides a way to cancel jobs that are redundant because a new
commit has been pushed to a PR rendering the old job obsolete.

Unfortunately those two features do not easily work together.  Github merge-queues
by-design queue up multiple builds on the same branch.  When used with Github
concurrency, this results in new jobs in the merge-queue cancelling older jobs, and
if old jobs are cancelled, that causes the merge-queue to fail completely, which
will cancel the new jobs as well.

To avoid this scenario, concurrency must be carefully configured to avoid interfering
with the operation of merge-queues.

Below is an example of a concurrent configuration that does this:

```yaml
    concurrency:
      group: >
        a+${{ github.event_name }}
        b+${{ github.workflow_ref }}
        c+${{ github.job }}
        d+${{ matrix.ghc }}
        e+${{ matrix.cabal }}
        f+${{ matrix.os }}
        g+${{ (startsWith(github.ref, 'refs/heads/gh-readonly-queue/') && github.run_id) || github.event.pull_request.number || github.ref }}
      cancel-in-progress: true
```

The `>` symbol is `yaml` operator that means the text in the nested block are used to construct
a string with no newlines.

The `group` field defines a concurrency key.  This tells Github Actions that to cancel any older
jobs that share the same key.  This must be defined carefully.  If it is too general, jobs will
be cancelled that shouldn't and if it is to specific then jobs that should be cancelled won't be.

Each line in the `group` block defines a component of the concurrency key.  Each component begins
with `<letter>+` as a useful convention.  This is if we print this key without it (for example
for debugging purposes) the way the `>` `yaml` operator combines multiple lines into a single
line makes it difficult to determine which substring of the printed key corresponds to which
component in the `yaml` file.  With the prefix, we can identify the substring with the key
component in the workflow by the `<letter>+` prefix.

The components as identified by their prefix are included because:

* `a`: Jobs triggered due to different Github events should not cancel each other
* `b`: Jobs triggered due to different Github workflow refs (for example branches) should not cancel each other
* `c`: Jobs with different job names (for example linting vs building) should not cancel each other
* `d`: Jobs that build with different `ghc` versions should not cancel each other
* `e`: Jobs that build with different `cabal` version should not cancel each other
* `f`: Jobs that build on different `os`es should not cancel each other
* `g`: Jobs on the merge queue should not cancel each other

The `cancel-in-progress` signals to Github Actions that in the event of two jobs with the
same concurrency key running concurrently, the older one should be cancelled.
