Hello all, this page will be used to keep a static status of the Byron Re-Write node. 

This will support the capabilities page https://github.com/input-output-hk/cardano-node/wiki/Cardano-Haskell-Node-Capabilities

The work below is that specifically required to make the node _**public test and main net ready**_



📅 _As of 12/07/2019_


***

****Node Implementation**** 🛠 Tasks - remaining development team effort

**Consensus**
* Extend the mempool:
   * Track transaction sizes
   * Expire older transactions
   * In addition to transactions, support delegation and update payloads
* Chain Sync Client optimisations:
    * Track block sizes
    * Pipe lining
    * Disconnect from nodes with invalid blocks

***

****Monitoring & Benchmarking**** 🔎📝  Tasks remaining

* Transaction generator to run system benchmarks
* Benchmarking and monitoring of cardano-node ;”live-view”  
* Capture mempool metrics and chain metrics  
* Support teams for implementing logging, benchmarking, monitoring - wallet BE, consensus, networking, ledger, devops

***
****Test Infrastructure**** 🏗  Tasks remaining

* Valid Chain Generators
* Invalid Chain Generators

***

**Testing** ⚡️ 🐛   Major test coverage remaining  

**Consensus**
* Protocol testing: do we reach consensus in the following cases (some are overlapping)
  * A node joins the network at a later point in time
  * A node disconnects and rejoins the network
  * There is a (temporary) network partition
  * There are delays in the network
  * Different network topologies
  * A node encounters corruption and restarts with a truncated chain
  * There is an explicit adversary
  * A node produces a few (or many) invalid blocks
  * A block doesn't match its header
  * There are multiple slot leaders or none at all
* Test what happens when there is a shortage of file handles. Additionally, we should make sure we don't leak file handles and what happens when a handle fails to close.


