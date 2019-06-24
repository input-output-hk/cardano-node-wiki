❗️ The node is made available for early integration, QA & DevOps activities. Further security, performance and general testing is required across all components ahead of a test and mainet launch ❗️ 

# Independent Node Cluster Capabilities 📦 

## Iteration #1 Capabilities - Made available mid June 2019 ✔️ 
* Node that communicate using TCP across different machines
* In memory storage for chain  
* Establish independent cluster using PBFT
    * Some limitations due to lack of connection management integration 
* Full network stack (exclusions listed further down)
    * Node to node:
        * Chain sync
        * Block fetch
        * Block fetch logic
    * Node to client:
        * Chain sync
        * Transaction submission to mempool
* Create blocks and distribute using PBFT Consensus algorithm 
* Validate blocks and transactions with blocks using final Byron compatible Ledger rules 
* Submit transactions local to block producing nodes
* Mock transactions that are elaborated into real transactions 
* Initial node shell integration 
* Consensus as a feature 
* Configuration 
* Logging and Monitoring 
    * Command line arguments parser 
    * Prometheus interface (EKG / monitoring metrics )
    * Mempool Tx level logging (transactions in Mempool)
    * Tx submission logging


⚠️ **Iteration #1 Limitations** ⚠️  

There are a few limitations to keep in mind due to the fact that certain things have not yet been integrated.

**Storage** The real storage system has not yet been integrated. The node now is using the mock storage implementation. The consequence of this is that the chain storage is in-memory and not on disk. So this limits the size of the chain that can be tested, and the chain is not persistent on restart. So this precludes syncing with mainnet. The full storage subsystem uses the same API as the mock one so the integration is expected to be straightforward, and it is expected to be integrated for the next demo.

**Connection management** The network connection management subsystem has not yet been integrated. The node is currently using very simplistic static connection logic: it will establish the requested connections on startup but not re-establish connections on failure. The consequence is that any peer disconnecting causes an exception that will shut down the node. The full connection management subsystem is currently in testing and it is expected to be integrated for the next demo.

**Mainnet validation** We have previously demonstrated mainnet validation for the ledger rules, but have not yet demonstrated that for the PBFT consensus rules. While the PBFT rules are integrated, there are some remaining parts to complete to be able to validate mainnet. In particular the support for EBBs (epoch boundary blocks). Therefore it is not currently possible with the node at this stage to sync with mainnet. We anticipate that this will be completed in time for the next demo.

**Local clients** The node to client protocol has been integrated but the chain sync part does not yet have a example client. This will be needed for wallet integration. This is a relatively small addition and so we expect to have a chain sync client template available for the next demo.

**Transaction relay** The node to node transaction submission protocol has not yet been integrated. This means that while it is possible to submit transactions locally to a block producing node, it is not yet possible to submit them to one node and have them forwarded to other nodes. The code for this component is mostly complete, including tests, so we anticipate that this will be integrated in time for the next demo.

## Iteration #2 Capabilities - Due early July 2019 🔨 

**Consensus**
* Chain state storage implementation & integration
* Mainet synchronization completion (e.g. EBBs)

**Network**
* Completion of peer discovery & connection management 
* Node to node protocol complete
* Node to client protocol complete
* Old <> New Proxy alignment with node protocols complete

**Logging**
* Tracepoint updates (ongoing activity during QA & DevOps testing)

## Iteration #2 Capabilities - Due mid July 2019 🔨 

**Network**
* Old <> New Proxy Bi-directional complete (new to old)

