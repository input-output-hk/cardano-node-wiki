[_Demo run internally 12th July 2019_]

# Background
This demo is the second demo of a cluster of independent Cardano nodes in the Shelley era, first release - Byron Re-write.

The previous demo showed the full integration of all the major components. This demo focuses on the consensus chain state functionality. Whereas the previous demo used the mock implementation of this functionality, this time we are using the full implementation. The full implementation uses the on-disk storage components (immutable, volatile and ledger), and provides efficient, concurrent, implementations of the chain state API.

There are still some features of the node missing that will be included over the coming weeks. For a comprehensive overview of the current capabilities and limitations, see the [this page](https://github.com/input-output-hk/cardano-node/wiki/Cardano-Haskell-Node-Capabilities).

# What are we looking at?
- TODO: fill in

# What does this show?
This shows that we are able to use the proper implementation of the Ouroboros chain validation and chain selection, with on-disk block storage. This is designed to be efficient and resistant to asymetric resource consumption and other denial of service attacks. It also demonstrates the integration of all the on-disk storage components, which are designed to be robust to hard shutdowns and filesystem corruption (silent or otherwise).

This builds on the previuous demo so it also shows that multiple nodes can run independently and communicate with each other using the full network stack. It shows transaction submission with a real mempool implementation, using the real PBFT ledger.

The addition of the on-disk storage means we can now set up a long-running staging cluster, since we are no longer using the mock in-memory storage (which was not intended to scale).

# Next steps
The next steps across the various parts of the node include:
- Completing EBB support in the ledger/consensus integration, and testing validating against mainnet epochs
- Updating the Byron proxy to use the full node's chain storage subsystem, and to work transparently with nodes by using exactly the same protocol bundle.
- Completing integration and testing of node-to-node transaction relay in the consensus layer
- Starting integration with the Wallet Backend
- Improving test coverage and infrastructure across the whole system

# Running the demo yourself

TODO: fill this in after recording the demo session.