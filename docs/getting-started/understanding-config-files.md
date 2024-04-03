# Understanding configuration files and how to use them

## The topology.json file

This file tells your node which nodes in the network it should talk to. A minimal version of this file looks as follows:

```json
{
  "Producers": [
    {
      "addr": "x.x.x.x",
      "port": 3001,
      "valency": 1
    }
  ]
}
```
* This means that your node will contact the node at IP `x.x.x.x` on `port 3001`.

* `valency` defines the number of active (hot) outgoing connections to different resolved IP addresses that your node should maintain when a DNS address is specified. The valency setting has no impact when setting an IP address, except when using `0` to disable the connection.

Your __block-producing__ node must __ONLY__ talk to your __relay nodes__, and the relay nodes should talk to other relay nodes in the network. See this [topology](https://explorer.cardano.org/relays/topology.json) to find out the IP addresses and ports of peers. The `topology.json` found at this link is updated once a week.

## The P2P topology.json file

The P2P topology file specifies how to obtain the _root peers_.

* The term _local roots_ refers to a group of peer nodes with which a node will aim to maintain a specific number of active, or 'hot' connections.
  These hot connections are those that play an active role in the consensus algorithm.
  Conversely, 'warm' connections refer to those not yet actively participating in the consensus algorithm.
  Local roots should comprise local relays or a local block-producing node, and any other peers that the node needs to maintain a connection with.
  These connections are typically kept private.


* _public roots_: publicly known nodes (e.g. IOG relays, or ledger nodes).
  They are either read from the configuration file directly or from the chain.
  The configured ones will be used to pass a recent snapshot of peers needed before the node caches up with the recent enough chain to construct root peers by itself.

The node does not guarantee a connection with every public root, unlike local ones.
However, by being present in the set, it gets an opportunity to establish an outbound connection with that peer.

A minimal version of this file looks like this:

```json
{
  "localRoots": [
      { "accessPoints": [
            {
              "address": "x.x.x.x",
              "port": 3001
            }
          ],
        "advertise": false,
        "valency": 1,
        "warmValency": 2,
        "trustable": true
      }
  ],
  "publicRoots": [
    { "accessPoints": [
        {
          "address": "y.y.y.y",
          "port": 3002
        }
        ],
      "advertise": false
    }
  ],
  "useLedgerAfterSlot": 0,
  "bootstrapPeers": [
    {
      "address": "z.z.z.z",
      "port": 3003
    }
  ]
}
```

* The main difference between `LocalRoots` and `PublicRoots` is the ability to specify various groups with varying valencies in the former.
  This can be valuable for informing your node about different targets within a group to achieve.
  `LocalRoots` is designed for peers that the node should always keep as hot or warm, such as its own block producer.
  On the other hand, `PublicRoots` serves as a source of fallback peers, to be used if we are before the configured `useLedgerAfterSlot` slot.


* This means that your node will initiate contact with the node at IP `x.x.x.x` on `port 3001` and resolve the DNS domain `y.y.y.y` (provided it exists).
  It will then make efforts to establish a connection with at least one of the resolved IPs.

* `hotValency` tells the node the number of connections it should attempt to select from the specified group.
  When a DNS address is provided, valency determines the count of resolved IP addresses for which the node should maintain an active (hot) connection.
  Note: one can also use the deprecated now `valency` field for `hotValency`.


- `warmValency` is an optional field, similar to `hotValency`, that informs the node about the number of peers it should maintain as warm.
  This field is optional and defaults to the value set in the `valency` or `hotValency` field.
  If a value is specified for `warmValency`, it should be greater than or equal to the one defined in `hotValency`; otherwise, or `hotValency` will be adjusted to match this value.
  We recommend users set the `warmValency` value to `hotValency + 1` to ensure at least one backup peer is available to be promoted to a hot connection in case of unexpected events.
  Check [this issue](https://github.com/intersectmbo/ouroboros-network/issues/4565) for more context on this `WarmValency` option.


* Local root groups shall be non-overlapping.

* The advertise parameter instructs a node about the acceptability of sharing its address through *peer sharing* (which we'll explain in more detail in a subsequent section).
  In essence, if a node has activated peer sharing, it can receive requests from other nodes seeking peers.
  However, it will only disclose those peers for which it has both local and remote permissions.

  Local permission corresponds to the value of the advertise parameter.
  On the other hand, 'remote permission' is tied to the `PeerSharing` value associated with the remote address, which is ascertained after the initial handshake between nodes.


* Local roots should not be greater than the `TargetNumberOfKnownPeers`.
  If they are, they will get clamped to the limit.

- For `trustablePeers` and `bootstrapPeers` read the next section.

Your __block-producing__ node must __ONLY__ talk to your __relay nodes__, and the relay node should talk to other relay nodes in the network.

You have the option to notify the node of any changes to the topology configuration file by sending a SIGHUP signal to the `cardano-node` process.
This can be done, for example, with the command `pkill -HUP cardano-node`.
Upon receiving the signal, the `cardano-node` will re-read the configuration file and restart all DNS resolutions.

Please be aware that this procedure is specific to the topology configuration file, not the node configuration file.
Additionally, the SIGHUP signal will prompt the system to re-read the block forging credentials file paths and attempt to fetch them to initiate block forging.
If this process fails, block forging will be disabled.
To re-enable block forging, ensure that the necessary files are present.

You can disable ledger peers by setting the `useLedgerAfterSlot` to a negative value.

### Genesis lite a.k.a Bootstrap Peers

Bootstrap Peers is a pre-Genesis feature that means to provide a way for a node to connect to a trustable set of peers when the its chain falls behind.

Bootstrap peers can be disabled by setting `bootstrapPeers: null`.
They are enabled by providing a list of addresses.
By default bootstrap peers are disabled.

Trustable peers are composed by the bootstrap peers (see `bootstrapPeers` option in the example topology file above) and the trustable local root peers (see `trustable` option in the example topology file above).
By default local root peers are not trustable.

In order for the node to be able to start and make progress, when bootstrap peers are enabled, the user _must_ provide a trustable source of peers via topology file.
This means that the node will only start if either the bootstrap peers list is non-empty or there's a local root group that is trustable.
Failing to configure the node with trustable peer sources will make it so that the node crashes with an exception.
*_Please note_* that if the only source of trustable peers is a DNS name, the node might not be able to make progress once in fallback state, if DNS is not providing any addresses.

With bootstrap peers enabled the node will trace:

- `TraceLedgerStateJudgmentChanged {TooOld,YoungEnoug}`: If it has changed to any of these states.

  - `TooOld` state means that the information the node is getting from its peers is outdated and behind at least 20 min.
    This means there's something wrong and that the node should only connect to trusted peers (trusted peers are bootstrap peers and trustable local root peers) in order to sync.

  - `YoungEnough` state means that the node is caught up and that it can now connect to non-trusted peers.

- `TraceOnlyBootstrap`: Once the node transitions to `TooOld` the node will disconnect from all non-trusted peers and reconnect to only trusted ones in order to sync from trusted sources only.
  This tracing message means that the node has successfully purged all non-trusted connections and is only going to connect to trusted peers.

## The genesis.json file

The genesis file is generated with the `cardano-cli` by reading a `genesis.spec.json` file, which is out of scope for this document.
But it is important because it is used to set:

* `genDelegs`, a mapping from genesis keys to genesis delegates
* `initialFunds`, a mapping from the initial addresses to the initial values at those addresses
* `maxLovelaceSupply`, the total amount of lovelaces in the blockchain
* `systemStart`, the time of slot zero.

The `genesis.json` file looks like the one below:
```json
{
  "activeSlotsCoeff": 0.05,
  "protocolParams": {
    "protocolVersion": {
      "minor": 0,
      "major": 2
    },
    "decentralisationParam": 1,
    "eMax": 18,
    "extraEntropy": {
      "tag": "NeutralNonce"
    },
    "maxTxSize": 16384,
    "maxBlockBodySize": 65536,
    "maxBlockHeaderSize": 1100,
    "minFeeA": 44,
    "minFeeB": 155381,
    "minUTxOValue": 1000000,
    "poolDeposit": 500000000,
    "minPoolCost": 340000000,
    "keyDeposit": 2000000,
    "nOpt": 150,
    "rho": 0.003,
    "tau": 0.20,
    "a0": 0.3
  },
  "genDelegs": {
    "ad5463153dc3d24b9ff133e46136028bdc1edbb897f5a7cf1b37950c": {
      "delegate": "d9e5c76ad5ee778960804094a389f0b546b5c2b140a62f8ec43ea54d",
      "vrf": "64fa87e8b29a5b7bfbd6795677e3e878c505bc4a3649485d366b50abadec92d7"
    },
    "b9547b8a57656539a8d9bc42c008e38d9c8bd9c8adbb1e73ad529497": {
      "delegate": "855d6fc1e54274e331e34478eeac8d060b0b90c1f9e8a2b01167c048",
      "vrf": "66d5167a1f426bd1adcc8bbf4b88c280d38c148d135cb41e3f5a39f948ad7fcc"
    },
    "60baee25cbc90047e83fd01e1e57dc0b06d3d0cb150d0ab40bbfead1": {
      "delegate": "7f72a1826ae3b279782ab2bc582d0d2958de65bd86b2c4f82d8ba956",
      "vrf": "c0546d9aa5740afd569d3c2d9c412595cd60822bb6d9a4e8ce6c43d12bd0f674"
    },
    "f7b341c14cd58fca4195a9b278cce1ef402dc0e06deb77e543cd1757": {
      "delegate": "69ae12f9e45c0c9122356c8e624b1fbbed6c22a2e3b4358cf0cb5011",
      "vrf": "6394a632af51a32768a6f12dac3485d9c0712d0b54e3f389f355385762a478f2"
    },
    "162f94554ac8c225383a2248c245659eda870eaa82d0ef25fc7dcd82": {
      "delegate": "4485708022839a7b9b8b639a939c85ec0ed6999b5b6dc651b03c43f6",
      "vrf": "aba81e764b71006c515986bf7b37a72fbb5554f78e6775f08e384dbd572a4b32"
    },
    "2075a095b3c844a29c24317a94a643ab8e22d54a3a3a72a420260af6": {
      "delegate": "6535db26347283990a252313a7903a45e3526ec25ddba381c071b25b",
      "vrf": "fcaca997b8105bd860876348fc2c6e68b13607f9bbd23515cd2193b555d267af"
    },
    "268cfc0b89e910ead22e0ade91493d8212f53f3e2164b2e4bef0819b": {
      "delegate": "1d4f2e1fda43070d71bb22a5522f86943c7c18aeb4fa47a362c27e23",
      "vrf": "63ef48bc5355f3e7973100c371d6a095251c80ceb40559f4750aa7014a6fb6db"
    }
  },
  "updateQuorum": 5,
  "networkId": "Mainnet",
  "initialFunds": {},
  "maxLovelaceSupply": 45000000000000000,
  "networkMagic": 764824073,
  "epochLength": 432000,
  "systemStart": "2017-09-23T21:44:51Z",
  "slotsPerKESPeriod": 129600,
  "slotLength": 1,
  "maxKESEvolutions": 62,
  "securityParam": 2160
}
```
Here is a brief description of each parameter. You can learn more in the [spec](https://github.com/intersectmbo/cardano-ledger/tree/master/eras/shelley/impl).


| PARAMETER | MEANING |
|----------| --------- |
| activeSlotsCoeff | The proportion of slots in which blocks should be issued |
| poolDeposit | The amount of a pool registration deposit |
| protocolVersion| Accepted protocol versions |
| decentralisationParam | Percentage of blocks produced by federated nodes |
| maxTxSize | Maximum transaction size |
| minPoolCost | Stake pools cannot register/re-register their stake cost below this value |
| minFeeA | The linear factor for the minimum fee calculation |
| maxBlockBodySize | Maximum block body size |
| minFeeB | The constant factor for the minimum fee calculation |
| eMax | Epoch bound on pool retirement |
| extraEntropy | Well, extra entropy =) |
| maxBlockHeaderSize | Maximum block header size |
| keyDeposit | The amount of a key registration deposit |
| nOpt | Desired number of pools |
| rho | Monetary expansion |
| tau | Treasury expansion |
| a0 | Pool's pledge influence |
| networkMagic | Used to identify the testnets |
| systemStart | Time of slot 0 |
| genDelegs | Mapping from genesis keys to genesis delegate |
| updateQuorum | Determines the quorum needed for votes on the protocol parameter updates |
| initialFunds | Mapping address to values |
| maxLovelaceSupply | The total number of lovelace in the system, used in the reward calculation |
| networkMagic | To identify the testnet |
| epochLength | Number of slots in an epoch |
| staking | Initial delegation |
| slotsPerKESPeriod | Number of slots in the KES period |
| slotLength | in seconds |
| maxKESEvolutions | The maximum number of times a KES key can be evolved before a pool operator must create a new operational certificate |
| securityParam | Security parameter k |


## The config.json file

The default `config.json` file that we downloaded is shown below.

This file has __4__ sections that allow you to have full control of what your node does and how the information is presented.

__NOTE: Due to how the config.json file is generated, fields on the real file are shown in a different (less coherent) order. Here we present them in a more structured way__

### Basic node configuration

The first section relates the basic node configuration parameters. Make sure you have `TPraos`as the protocol, the correct path to the `mainnet-shelley-genesis.json` file, and `RequiresMagic`for its use on testnet:

	  "Protocol": "TPraos",
	  "GenesisFile": "mainnet-shelley-genesis.json",
	  "RequiresNetworkMagic": "RequiresMagic",

### Update parameters

This protocol version number is used by block-producing nodes as part of the system for agreeing on and synchronizing protocol updates. You just need to be aware of the latest versions supported by the network. You don't need to change anything here:

	  "LastKnownBlockVersion-Alt": 0,
	  "LastKnownBlockVersion-Major": 2,
	  "LastKnownBlockVersion-Minor": 0,


### Tracing

`Tracers` tell your node what information you are interested in when logging. Like switches that you can turn ON or OFF according to the type and quantity of information that you are interested in. This provides fairly coarse grained control, but it is relatively efficient at filtering out unwanted trace output:

`TurnOnLogging`: enables or disables logging overall.

`TurnOnLogMetrics`: enables the collection of various OS metrics such as memory and CPU use. These metrics can be directed to the logs or monitoring backends.

`setupBackends`, `defaultBackends`, `hasEKG`and `hasPrometheus`: the system supports a number of backends for logging and monitoring. These settings list the backends available to use in the configuration. The logging backend is called `Katip`. Also, enable the EKG backend if you want to use the EKG or Prometheus monitoring interfaces.

`setupScribes` and `defaultScribes`: you must set up outputs (called scribes) for the Katip logging backend. The available types of scribes are:

* FileSK: for files
* StdoutSK/StderrSK: for stdout/stderr
* JournalSK: for systemd's journal system
* DevNullSK
* The scribe output format can be ScText or ScJson.

`rotation`: the default file rotation settings for Katip scribes, unless overridden in the setupScribes above for specific scribes.

```json
"TurnOnLogging": true,
"TurnOnLogMetrics": true,
"TracingVerbosity": "NormalVerbosity",
"minSeverity": "Debug",
"TraceBlockFetchClient": false,
"TraceBlockFetchDecisions": false,
"TraceBlockFetchProtocol": false,
"TraceBlockFetchProtocolSerialised": false,
"TraceBlockFetchServer": false,
"TraceBlockchainTime": false,
"TraceChainDb": true,
"TraceChainSyncBlockServer": false,
"TraceChainSyncClient": false,
"TraceChainSyncHeaderServer": false,
"TraceChainSyncProtocol": false,
"TraceDNSResolver": true,
"TraceDNSSubscription": true,
"TraceErrorPolicy": true,
"TraceForge": true,
"TraceHandshake": false,
"TraceIpSubscription": true,
"TraceLocalChainSyncProtocol": false,
"TraceLocalErrorPolicy": true,
"TraceLocalHandshake": false,
"TraceLocalTxSubmissionProtocol": false,
"TraceLocalTxSubmissionServer": false,
"TraceMempool": true,
"TraceMux": false,
"TraceTxInbound": false,
"TraceTxOutbound": false,
"TraceTxSubmissionProtocol": false,
"setupBackends": [
  "KatipBK"
],
"defaultBackends": [
  "KatipBK"
],
"hasEKG": 12788,
"hasPrometheus": [
  "127.0.0.1",
  12798
],
"setupScribes": [
  {
    "scFormat": "ScText",
    "scKind": "StdoutSK",
    "scName": "stdout",
    "scRotation": null
  }
],
"defaultScribes": [
  [
    "StdoutSK",
    "stdout"
  ]
],
"rotation": {
  "rpKeepFilesNum": 10,
  "rpLogLimitBytes": 5000000,
  "rpMaxAgeHours": 24
  },
```

### Fine grained logging control

It is also possible to have more fine grained control over filtering of trace output, and to match and route trace output to particular backends. This is less efficient than the coarse trace filters above but provides much more precise control. `options`:

`mapBackends`: this routes metrics matching specific names to particular backends. This overrides the defaultBackends listed above. Note that it is an **override** and not an extension so anything matched here will not go to the default backend, only to the explicitly listed backends.

`mapSubtrace`: this section is more expressive, work on its documentation is ongoing.

```json
	  "options": {
	    "mapBackends": {
	      "cardano.node.metrics": [
	        "EKGViewBK"
	      ]
	    },
	    "mapSubtrace": {
	      "cardano.node.metrics": {
	        "subtrace": "Neutral"
	      }
	    }
	  }
	}
```

### Peer-to-peer (P2P) parameters and tracers

To run a node in P2P mode, configure the `EnableP2P` setting to `true`, which is the default value since `cardano-node-8.10`, in the configuration file. Additionally, ensure you specify the topology in the new format as described above.

There are a few new tracers and configuration options that you can set (listed below by
component):

#### Outbound governor

The outbound governor is responsible for satisfying targets of root peers, known (_cold_,
_warm_ and _hot_), established (_warm_ and _hot_) and active peers (synonym for _hot_ peers)
and local root peers.  The primary way to configure them is by setting the following
options:

* `TargetNumberOfRootPeers` (_default value: `100`_) - a minimal number of root peers
  (unlike other targets this one is one-sided, eg, a node might have more root peers
* `TargetNumberOfKnownPeers` (_default value: `100`_) - a target of known peers (must be
  larger or equal to `TargetNumberOfRootPeers`)
* `TargetNumberOfEstablishedPeers` (_default value: `50`_) - a target of all established
  peers (including local roots, ledger peers)
* `TargetNumberOfActivePeers` (_default value: `20`_) - a target for _hot_ peers, which
  engage in the consensus protocol.

Let's take note of two additional targets. In the topology file, you have the option to include local root peers. This comprises a list of peer groups, with each group having its own valency. The outbound governor ensures a connection with every local root peer and ensures that at least the specified number of them (the valency) are actively connected (hot). Consequently, the values for `TargetNumberOfKnownPeers`, `TargetNumberOfEstablishedPeers`, and `TargetNumberOfActivePeers` should be set sufficiently high to accommodate these local root peers.

The following traces can be enabled:

* `TracePeerSelection` (_by default on_) - tracks the selection of upstream peers done by the
  _outbound-governor_.  **Warm peers** are ones with which we have an open connection but
  don't engage in consensus protocol, **hot peers** are peers which engage in consensus
  protocol (via `chain-sync`, `block-fetch`, and `tx-submission` mini-protocols), **cold
  peers** are ones that we know about but the node doesn't have an established
  connection with.  Note that the notions of _hot_, _warm_ and _cold_ are only related to usage
  of initiator sides of mini-protocols in a connection (which can be either inbound or
  outbound).
* `TracePeerSelectionCounters` (_by default on_) - traces how many cold/warm/hot/local
   root peers the node has, it's also available via EKG.
* `TracePeerStateActions` (_by default on_) - includes traces from a component, which
  executes peer promotion/demotions between cold/warm and hot states.
* `TracePublicRootPeers` (_by default off_) - traces information about root/ledger peers
  (eg, IP addresses or DNS names of ledger peers, DNS resolution)
* `DebugPeerSelectionInitiator` and `DebugPeerSelectionInitiatorResponder` (_by default
  off_) - debug tracers that log the information about the current state of the _outbound
  governor_.

At this point [Haddock
documentation](https://ouroboros-network.cardano.intersectmbo.org/ouroboros-network/Ouroboros-Network-PeerSelection-Governor.html)
of the outbound governor is available.

#### Peer Sharing

Peer sharing is a novel feature that provides an additional method for the outbound
governor to reach its targets for known peers. With peer sharing, the node can request
peer information from other nodes with which it has an established connection.

**NOTE** _Peer sharing_ is a relatively new feature which is turned off by default.
It is important to use big ledger peers (on by default) when enabling peer sharing.

The main method for configuring peer sharing involves setting the following option:

- `PeerSharing` (a boolean value, default `False`)
    * `False`: peer sharing is disabled, which means the node won't request peer
      information from any other node, and will not respond to such requests from others
      (the mini-protocol won't even start);
    * `True`: peer sharing is enabled, the node may issue as well as respond
      to requests for peers.

When `advertise: true` is set in the topology file for a given local root peers
group, the node can disseminate knowledge of such peers with other participants
of the network through the peer-sharing mechanism. The node only sends IP
addresses and port numbers through peer sharing, no additional information is
shared, in particular, the node doesn't send DNS names.

Let's consider an example configuration:

* a block-producing node (BP) has the `PeerSharing: False` set in its
  configuration file;
* a relay node should set `advertise: false` for the BP in its local peer
  section of its topology file.

**IMPORTANT** When the handshake between the BP and the relay occurs, the relay will see that
the BP doesn't want to participate in peer sharing. As a result, neither side
will engage in peer sharing on this connection.  If the BP had `PeerSharing:
True` it could ask its relays for additional peers to connect to and up talking
directly to untrusted peers

**IMPORTANT** Please note that if the relay set `advertise: true` for the BP in its
topology file, it could leak the knowledge about the BP's IP address to the
outside word.

For two nodes to share peers between themselves, both of them
have to run with `PeerSharing: True`, if either of them sets it to `False`
sharing peers will not be possible in any direction.

**IMPORTANT** Note that if BP connects to some other relay which doesn't have
a BP entry in its local root peers, the BP's IP address might be
leaked by it to the outside word, since that relay will not know that BP's IP
address should not be advertised.  It is __recommended__ to setup a firewall
which blocks incoming traffic to `cardano-node`'s port apart from SPO's own
relays (if one is using P2P mode on BPs and relays, one can actually block all
incoming traffic, the relays will reuse the connection opened by the BP).

#### Inbound governor

The inbound governor is maintaining the responder side of all mini-protocols. Unlike the
outbound governor, it is a purely responsive component that reacts to the actions of the remote
peer (its outbound governor).

* `TraceInboundGovernor` (_by default on_) - traces information about inbound connection,
  eg, we track if the remote side is using  our node as _warm_ or _hot peer_, traces when
  we restart a responder.
* `TraceInboundGovernorCounters` (_by default on_) - traces the number of peers which use the
  node as `cold`, `warm` or `hot` (which we call `remote cold`, `remote warm` or `remote
  hot`). Note that we only know if a peer is in the remote cold state if we connect to
  that peer and it's not using the connection. This information is also available via
  EKG.
* `TraceInboundGovernorTransitions` (_by default on_) - a debug tracer which traces
  transitions between remote cold, remote warm, and remote hot states.

The inbound governor is documented in [The Shelley networking
protocol](https://ouroboros-network.cardano.intersectmbo.org/pdfs/network-spec) (section 4.5).

#### Connection manager

The connection manager tracks the state of all TCP connections, and enforces various timeouts, for example,
when the connection is not used by either of the sides. The following traces are
available:

* `TraceConnectionManager` (_by default on_) - traces information about new inbound or
  outbound connections, connection errors.
* `TraceConnectionManagerCounters` (_by default on_) - traces the number of inbound,
  outbound, duplex (connections which negotiated P2P mode and can use a connection in full
  duplex mode), full duplex (connections that run mini-protocols in both directions, eg,
  at least _warm_ and _remote warm_ at the same time), unidirectional connections
  (connections with non-P2P nodes, or P2P nodes, which configured themselves as initiator
  only nodes).
* `TraceConnectionManagerTransitions` (_by default on_) - low level traces which trace
  connection state changes in the connection manager state machine.

The connection manager is documented in [The Shelley networking
protocol](https://ouroboros-network.cardano.intersectmbo.org/pdfs/network-spec) (section 4).

#### Ledger peers

Ledger peers are the relays registered on the chain. Currently, we use a square of the stake
distribution to randomly pick new ledger peers. You can enable `TraceLedgerPeers` (_by
default off_) to log actions taken by this component.

#### Server

The accept loop. You can enable `TraceServer` to log its actions or errors it encounters
(_by default it is off_, however we suggest to turn it on) .

**Please note that this version contains no breaking changes**
