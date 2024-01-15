[//]: # (This document represents an entry point for those who want to get a basic grasp of new tracing system.)
[//]: # (Its focus is on how to get the new system up and running for production, as well as provide paths to more in-depth documentation resources.)

# New Tracing Quickstart

* What is it?  

* What does it deliver?  

* Architecture (service; polling; socket-only)

* choice legacy

## Table of contents

- [New Tracing Quickstart](#new-tracing-quickstart)
  - [Table of contents](#table-of-contents)
  - [Difference to the legacy system](#difference-to-the-legacy-system)
- [TBD: split this up](#tbd-split-this-up)
- [Running `cardano-tracer`](#running-cardano-tracer)
  - [Setting up `cardano-tracer`](#setting-up-cardano-tracer)
    - [Running on different machines](#running-on-different-machines)
    - [Configuring tracers in `cardano-node`](#configuring-tracers-in-cardano-node)
- [Migration to new tracing](#migration-to-new-tracing)
    - [Using the legacy system](#using-the-legacy-system)
    - [Mapping of legacy (Prometheus) names](#mapping-of-legacy-prometheus-names)
    - [For developers](#for-developers)
    - [Support](#support)
- [Glossary](#glossary)
- [Further reading](#further-reading)
    - [Documentation of trace messages and further documentation](#documentation-of-trace-messages-and-further-documentation)

## Difference to the legacy system




# TBD: split this up

The current tracing system has two ways to identify the message, a hierarchical name we
call it's `Namespace` and the `Kind field` in machine representation. We base our implementation
on the namespace, and require a one-to-one correspondence between namespaces and messages (bijective mapping).
As we have two mechanisms for the same purpose for historic reasons, we will soon
__deprecate the Kind field__, and it will disappear in the near future. So we strongly
advice to use namespaces for any analysis tools of traces!

For new tracing, much of the tracing functionality has been put in a seperate program called
cardano-tracer. You need to start this before you start cardano-node. However it is still
possible to log to stdout without cardano-tracer.

# Running `cardano-tracer`

`cardano-tracer` is part of the new tracing infrastructure. It is a separate service that accepts different messages from the node and handles them. So it is assumed that if you want to use the new tracing infrastructure - you will use `cardano-tracer`. Please read its [documentation](https://github.com/input-output-hk/cardano-node/blob/master/cardano-tracer/docs/cardano-tracer.md) for more details.

First of all, add `--tracer-socket-path-connect /tmp/forwarder.sock` option to the node's command line options, asking it to connect to `cardano-tracer`.
> TBD: does that exist in node config?

## Setting up `cardano-tracer`

1. Build and run `cardano-tracer` using `cabal`:
~~~shell
$ cabal build cardano-tracer && cabal install cardano-tracer --installdir=<PATH_TO_DIR> --overwrite-policy=always
$ cd <PATH_TO_DIR>
$ ./cardano-tracer --config <PATH_TO_CONFIG>
~~~

2. Build and run `cardano-tracer` using `nix`:  
  (A `nix` build assumes the identical `nix` setup as described for `cardano-node`; for details please see [here](https://github.com/input-output-hk/cardano-node/blob/master/doc/getting-started/building-the-node-using-nix.md))
~~~shell
$ nix build .#cardano-tracer -o cardano-tracer-build
$ ./cardano-tracer-build/bin/cardano-tracer --config <PATH_TO_CONFIG>
~~~

   

`<PATH_TO_CONFIG>` is a path to `cardano-tracer`'s configuration file. Configuration can be provided both in JSON and YAML format. This is a minimal example:

~~~yaml
networkMagic: 764824073
network:
  tag: AcceptAt
  contents: "/tmp/forwarder.sock"
logging:
  - logRoot: "/tmp/cardano-tracer-logs"
    logMode: FileMode
    logFormat: ForMachine
~~~

This example describes the simplest case, when the node and `cardano-tracer` on the same machine.

> TBD. explain fields

After you run the node, it will establish the connection with `cardano-tracer` and will start to forward messages to it.
> TBD assume c-t is up and running
As a result, you will find log files, in JSON format, in `/tmp/cardano-tracer-logs` directory.

### Running on different machines

> TBD: ssh tunnel for sockets 
> TBD: SSH setup!!!
>
> Initiator lingo

### Configuring tracers in `cardano-node`



In Cardano a default configuration is given in the module [Cardano.Node.Tracing.DefaultTraceConfig](https://github.com/input-output-hk/cardano-node/blob/master/cardano-node/src/Cardano/Node/Tracing/DefaultTraceConfig.hs). In the config file all entries of the default configuration can be overridden. To remove a frequency limiter, define a limiter with maxFrequency 0.0.

> TBD.: include default config as explicit appendix in JSON / YAML
> root level ""

1. Specify a filter for the severity of the messages you want to see, e.g.:

  ~~~yaml
  # Show messages of Severity Notice or higher as default
  "":
      severity: Notice

    # But show ChainDB messages starting from Info
  ChainDB:
      severity: Info
  ~~~


  So the namespaces are used for configuration values, which works
  down to individual messages, and the more specialized value overwrites the more general.

  If you don't want to see any messages from tracers the new severity `Silence`
  exists, which suppresses all messages.
  
  > Q: Is Silence a regular severity level for which traces can never be emitted?   (only from config viewpoint) 
  > Q: Can I nested overriding? How is the severity of a specific trace evaluated? -- most specific info is taken


2. Specify in which detail level, the messages get shown
> TBD rendering

  ~~~yaml
  "":
      # Keep this
      severity: Notice
      # All messages are shown with normal detail level
      detail: DNormal
  ~~~

  Other options would be DMinimal, DDetailed and DMaximum. This has only an effect on messages which support the representation in different ways.

  > Q: Is there a related concept of detail in legacy tracing? Yes - different

3. Specify limiters for the frequency of messages

    Eliding tracers are not supported in new-tracing, instead you can limit the
    frequency in which messages get shown.

    ~~~yaml
    ChainDB.AddBlockEvent.AddedBlockToQueue:
        # Only show a maximum of 2 of these messages per second
        maxFrequency: 2.0
    ~~~

    The activity of limiters will be written in the traces as well.

4. Specify the backends the messages are routed to.

  ~~~yaml
  "":
      # Keep this
      severity: Notice
      # And this
      detail: DNormal
      # And specify a list of backends to use
      backends:
        - Stdout MachineFormat
        - EKGBackend
        - Forwarder
  ~~~

  These are all the backends currently supported. With Stdout you have the
  options MachineFormat or HumanFormatColoured/HumanFormatUncoloured.
  If messages don't support representation in HumanFormat* they are shown in MachineFormat anyway.

  Forwarder means that messages are send to cardano-tracer

Configuration can be written in JSON and YAML, we have shown the examples in YAML.


# Migration to new tracing

From now on, the legacy tracing system, which is based on `iohk-monitoring-framework` (TODO: link)
is no longer enabled by default. Instead, the new tracing system based on `trace-dispatcher` and `cardano-tracer`
will be activated.

However, the legacy tracing system will remain fully functional and usable within `cardano-node`. Up until
its eventual removal, the functionality of the new system will see minor constraints. In particular the
hot reloading of tracer configuration for a running node won't be available due to technical incompatibilites.

> Q: when is now? node release 8.1 !
> Q: deprecation plan for legacy tracing ? (at least 3 mon)

### Using the legacy system

To continue using the legacy system, the value `UseTraceDispatcher` has to be set to `false` in the node configuration.

Additionally, make sure the configuration does still contain the other values necessary for running the old tracing system.

### Mapping of legacy (Prometheus) names

> TBD IN C-T CONFIG

### For developers

For developing tracers in the transition time we suggest:

1. Don't use strictness annotations for trace types. Trace messages are either
discarded immediately (which happens frequently) or instantly converted to another format
but never stored. So strictness annotations make the code less efficient without any benefit.

2. If you develop new tracers we suggest that you develop the new tracers format first,
and then map to old tracers, as the new tracers will survive. You will find plenty of
examples in cardano-node under Cardano.Node.Tracing.Tracers.

3. Contact the Performance & Tracing team or the **#perf-tracing** channel for any questions and feedback.

4. workbench to try out

5. AcceptAt -> reverse connection

### Support

In this transition time new-tracing can be tested and improved. Since we have several hundred trace messages
it is expected that you will find regressions and bugs in the port, please help to find them.


# Glossary

> TBD: tracer nomenclature here;  
> This is the terminology we want to be used in all sorts of communication, e.g. issue tracking, support questions...
> cardano node forum

# Further reading

> TBD: provice .md links in repo

### Documentation of trace messages and further documentation

This is a document which is regenerated periodically and documents all trace-messages,  metrics and data-points in cardano-node. It as well displays the handling of these messages with the current default configuration:

[Cardano Trace Documentation](https://github.com/input-output-hk/cardano-node/blob/master/doc/new-tracing/tracers_doc_generated.md)

This document describes the underlying library trace-dispatcher:

[trace-dispatcher: efficient, simple and flexible program tracing](https://github.com/input-output-hk/cardano-node/blob/master/trace-dispatcher/doc/trace-dispatcher.md)

This document describes a seperate service for logging and monitoring Cardano nodes:

[Cardano Tracer](https://github.com/input-output-hk/cardano-node/blob/master/cardano-tracer/docs/cardano-tracer.md)

This document describes RTView, which is a new real-time monitoring tool:

[RTView](https://github.com/input-output-hk/cardano-node/blob/master/cardano-tracer/docs/cardano-rtview.md)
