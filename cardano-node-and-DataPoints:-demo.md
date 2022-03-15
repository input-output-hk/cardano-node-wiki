This document provides a step-by-step example of how to work with `demo-acceptor`.

## What is it?

As you know, `trace-forward` library provides a new mini-protocol for working with `DataPoint`s. And `demo-acceptor` is a small application allowing to ask for arbitrary `DataPoint`. This application is mostly for demonstration/testing if the node provides a particular `DataPoint` or not. If so - its value will be printed out to stdout once per second.

## `cardano-node`: configuration

Since `DataPoint`s are provided by the new tracing infrastructure, please make sure it's enabled in the node's configuration. Open the configuration file and set `UseTraceDispatcher` to `true`. Also, make sure that `TraceOptionForwarder` section contains valid values, for example:

```
"TraceOptionForwarder": {
    "address": {"filePath": "/tmp/forwarder.sock"},
    "mode" : "Initiator",
    "queueSize" : 1000,
    "verbosity" : "Maximum"
}
```

## `demo-acceptor`: building

Since `demo-acceptor` is a part of `cardano-tracer` project, run this command in order to build it:

```
$ cabal build cardano-tracer
```

Also, you can build it directly:

```
$ cabal build exe:demo-acceptor
```

As a result, `demo-acceptor` will be built.

