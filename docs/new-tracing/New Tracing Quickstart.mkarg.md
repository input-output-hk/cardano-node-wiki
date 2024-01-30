[//]: # (This document represents an entry point for those who want to get a basic grasp of new tracing system.)
[//]: # (Its focus is on how to get the new system up and running for production, as well as provide paths to more in-depth documentation resources.)

# New Tracing Quickstart

* What is it?

* What does it deliver?

* Architecture (service; polling; socket-only)

* choice legacy

##Table of contents
- [New Tracing Quickstart](#new-tracing-quickstart)
   * [Difference to the legacy system](#difference-to-the-legacy-system)
- [TBD: split this up](#tbd-split-this-up)
- [Running `cardano-tracer`](#running-cardano-tracer)
- [Setting up `cardano-tracer`](#setting-up-cardano-tracer)
   * [Running on different machines](#running-on-different-machines)
- [Configuring tracers in `cardano-node`](#configuring-tracers-in-cardano-node)
   * [TraceOptions](#traceoptions)
      + [Severity](#severity)
      + [Backends:](#backends)
      + [Detail](#detail)
      + [MaxFrequency](#maxfrequency)
   * [Other configuration entries](#other-configuration-entries)
   * [Example configuration](#example-configuration)
- [Migration to new tracing](#migration-to-new-tracing)
      + [Using the legacy system](#using-the-legacy-system)
      + [Mapping of legacy (Prometheus) names](#mapping-of-legacy-prometheus-names)
      + [For developers](#for-developers)
      + [Support](#support)
- [Glossary](#glossary)
- [Further reading](#further-reading)
      + [Documentation of trace messages and further documentation](#documentation-of-trace-messages-and-further-documentation)


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

# Setting up `cardano-tracer`

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

## Running on different machines

> TBD: ssh tunnel for sockets
> TBD: SSH setup!!!
>
> Initiator lingo

# Configuring tracers in `cardano-node`

The conventional method for configuring tracers in cardano-node involves utilizing the node configuration file. The node configuration file, specified by the --config flag, refers to a .yaml (or .json) file responsible for configuring logging and other crucial settings for the node.

To enable or disable new tracing during the transition period, the following line in the configuration file toggles the TraceDispatcher, which is the name of the new library:

```yaml
UseTraceDispatcher: True
```

## TraceOptions

The primary configuration for new tracing is accomplished through the TraceOptions entry in the config file. These options are organized by namespace, with tracers structured hierarchically within a namespace. The trace dispatcher mandates that all messages have a unique name within this namespace.

The empty namespace ("") configures all tracers, and more specific namespace settings override more general ones. This allows the configuration of groups of messages down to individual messages.

The configurable values are:

### Severity

Filter messages based on severity, where messages with a severity less than the configured cutoff are excluded. Severity levels follow the enumeration outlined in section 6.2.1 of RFC 5424. Severity levels range from Debug (least severe) to Emergency (most severe).

  * `Emergency`: system is unusable
  * `Alert`: action must be taken immediatel
  * `Critical`: critical conditions
  * `Error`: error conditions
  * `Warning`: warning conditions
  * `Notice`: normal but significant condition
  * `Info`: informational messages
  * `Debug`: debug-level messages

Additionally, a `Silent` filter level unconditionally filters out all messages.

So e.g. if you want to see all messages of type `BlockFetch.Client.StartedFetchBatch` you can set the filter level of this messages to `Debug`, which means that it is shown independent of its severity. As this message has severity `Info`, it will as well be displayed for severity `Ìnfo`. If you don't want to see this messages, set the severity filter to Silent (or any value greater then `Info`).

```yaml
TraceOptions:
  "":
    severity: Notice

  BlockFetch.Client.StartedFetchBatch:
    severity: Debug
```

Tip: You can look up the severity, privacy and the default detail level of all cardano messages in the following document: https://github.com/input-output-hk/cardano-node-wiki/blob/main/docs/new-tracing/tracers_doc_generated.md


### Backends:
Configure the backends to which messages are sent. Available backends currently are:

* Forwarder:
Send these messages to cardano-tracer for logging it in a central place. This will not happen for private messages, which will not be forwarded.

* Stdout
Write these messages to your local stdout. It takes the additional parameters: `MachineFormat`, `HumanFormatColoured` and `HumanFormatUncoloured`, which specifies the format in which it writes to Stdout.

* EKGBackend
This tracer submits metrics to a local EKG store, which then further forwards the messages. It is necessary to specify this backend to get metrics delivered, e.g. to Prometheus.

So a TraceOption entry will start like this:

```yaml
TraceOptions:
  "":
    backends:
      - Stdout MachineFormat
      - EKGBackend
      - Forwarder
```

### Detail

Some messages support the representation in different detail levels. This applies only to the forMachine representation. Options are `DMinimal`, `DNormal`, `DDetailed`, and `DMaximum`. If no configuration value is given, the default output will use the `DNormal` representation.

```yaml
TraceOptions:
  "":
    detail: DNormal

  Net.Mux.Local.HandshakeClientError:
    detail: DDetailed
```

### MaxFrequency

Frequency filtering is used to limit the frequency of individual trace messages or group of trace messages. It offers a probabilistic suppression of messages when their average frequency surpasses a specified threshold parameter.

The frequency limiter, in addition to controlling message frequency, emits a suppression summary message under specific conditions:

- When message suppression commences.
- Every 10 seconds during active limiting, providing the count of suppressed messages.
- When message suppression concludes, indicating the total number of suppressed messages.

It is important to note that frequency filtering is designed to be applied selectively to a subset of traces, specifically those identified as potentially noisy. The configuration of frequency limits can thus be tailored to this subset of traces.

```yaml
TraceOptions:
  ChainDB.AddBlockEvent.AddedBlockToQueue:
    maxFrequency: 2.0
    # Limit the AddedBlockToQueue events to a maximum of two per second.
```

## Other configuration entries

With the following option you can set the frequency in which _peer messages_ are send.

```yaml
TraceOptionPeerFrequency: 2000
```
The unit are milliseconds. So the above configuration will trace peer messages every 2 seconds. If you leave out this entry, two seconds is as well the default.

With the following option you can set the frequency in which resource messages are send. Resource messages reports ghc statistics and OS information f

```yaml
TraceOptionResourceFrequency: 4000,
```
The unit are milliseconds. So the above configuration will trace resource messages every 4 seconds. If you leave out this entry, 5 seconds are the default.

With the following option you can specify an optional node name, which is used only for the NodeInfo message. If not given, or in  all other cases the hostName of the origin machine is used. The hostName is a standard part of traceMessages. TODO: Does this make any sense at all? Either use hostName always, or use this value always?

```yaml
TraceOptionNodeName: "heavenAndHell",
```

The TraceOptionForwarder options control parameters for the forwarding of trace messges to the cardano-tracer process from the node side. It should be generally fine to keep the default options here.

```yaml
TraceOptionForwarder:
  connQueueSize = 2000
  disconnQueueSize = 200000
  verbosity = Minimum
```

Cardano-tracer queries periodically trace messages from cardano-node. For this reason we have a queue in
cardaon-node, which stores messages. When the queue overfolws, we loose trace messages. If at start up, the connection to cardano tracer, yet has not been established, the disconnQueueSize is used as the size of the queue. After establishing the connection the smaller connQueueSize is used. The verbosity can be used to debug message forwarding to cardano-tracer via the trace-forward library.

## Example configuration

Finally an example configuration, which includes different aspects and which we show in YAML and JSON:

```yaml
UseTraceDispatcher: True
TraceOptions:
  "":
    severity: Notice
    detail: DNormal
    backends:
      - Stdout MachineFormat
      - EKGBackend
      - Forwarder
  ChainDB:
    severity: Info
    detail: DDetailed
  ChainDB.AddBlockEvent.AddedBlockToQueue:
    maxFrequency: 2.0
TraceOptionPeerFrequency: 5000

```

```json
{
  "UseTraceDispatcher": true,
  "TraceOptions": {
    "": {
      "severity": "Notice",
      "detail": "DNormal",
      "backends": [
        "Stdout MachineFormat",
        "EKGBackend",
        "Forwarder"
      ]
    },
    "ChainDB": {
      "severity": "Info",
      "detail": "DDetailed"
    },
    "ChainDB.AddBlockEvent.AddedBlockToQueue": {
      "maxFrequency": 2.0
    }
  },
  "TraceOptionPeerFrequency": 5000
}
```

When `TraceOptions` is empty, or other entries are missing in the configuration file, default entries are taken from
[Cardano.Node.Tracing.DefaultTraceConfig](https://github.com/intersectmbo/cardano-node/blob/master/cardano-node/src/Cardano/Node/Tracing/DefaultTraceConfig.hs) module.

  > Q: Is Silence a regular severity level for which traces can never be emitted?   (only from config viewpoint)

It is a filter level. So no message can have a severity of Silence

  > Q: Can I nested overriding? How is the severity of a specific trace evaluated? -- most specific info is taken

True

  > Q: Is there a related concept of detail in legacy tracing? Yes - different

  The same in old tracing, but with a different name



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
