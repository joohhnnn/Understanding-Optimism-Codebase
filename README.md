# Understanding Optimism Codebase

This document provides a comprehensive explanation of the Optimism codebase, aiming to help newcomers to Optimism quickly get started and truly understand how the code flow in the codebase works.

## Project Directory

### Finished：

- [**00-how-sequencer-generates-L2-blocks**](https://github.com/joohhnnn/Understanding-Optimism-Codebase/blob/main/sequencer/00-how-sequencer-generates-L2-blocks.md)
- [**01-how-block-sync**](https://github.com/joohhnnn/Understanding-Optimism-Codebase/blob/main/sequencer/01-how-block-sync.md)
- [**02-how-optimism-use-libp2p**](https://github.com/joohhnnn/Understanding-Optimism-Codebase/blob/main/sequencer/02-how-optimism-use-libp2p.md)
- [**03-how-op-batcher-works**](https://github.com/joohhnnn/Understanding-Optimism-Codebase/blob/main/sequencer/03-how-batcher-works.md)
- [**04-how-derivation-works**](https://github.com/joohhnnn/Understanding-Optimism-Codebase/blob/main/sequencer/04-how-derivation-works.md)
- [**05-how-op-proposer-works**](https://github.com/joohhnnn/Understanding-Optimism-Codebase/blob/main/sequencer/05-how-proposer-works.md)
- [**06-upgrade-of-opstack-in-EIP-4844**](https://github.com/joohhnnn/Understanding-Optimism-Codebase/blob/main/sequencer/06-upgrade-of-opstack-in-EIP-4844.md)
- [**07-what-is-fault-proof**](https://github.com/joohhnnn/The-book-of-optimism-fault-proof/blob/main/01-what-is-fault-proof.md)
- [**08-fault-dispute-game**](https://github.com/joohhnnn/The-book-of-optimism-fault-proof/blob/main/02-fault-dispute-game.md)
- [**09-cannon**](https://github.com/joohhnnn/The-book-of-optimism-fault-proof/blob/main/03-cannon.md)
- [**10-op-program**](https://github.com/joohhnnn/The-book-of-optimism-fault-proof/blob/main/04-op-program.md)
- [**11-op-challenger**](https://github.com/joohhnnn/The-book-of-optimism-fault-proof/blob/main/05-op-challenger.md)
  
---

### [The-book-of-optimism-fault-proof](https://github.com/joohhnnn/The-book-of-optimism-fault-proof)

---

### TODO：

- [op-e2e](https://github.com/joohhnnn/Understanding-Optimism-Codebase/tree/main/op-e2e): End-to-End testing of all bedrock components in Go
- [op-exporter](https://github.com/joohhnnn/Understanding-Optimism-Codebase/tree/main/op-exporter): Prometheus exporter client
- [op-heartbeat](https://github.com/joohhnnn/Understanding-Optimism-Codebase/tree/main/op-heartbeat): Heartbeat monitor service
- [op-service](https://github.com/joohhnnn/Understanding-Optimism-Codebase/tree/main/op-service): Common codebase utilities
- [op-wheel](https://github.com/joohhnnn/Understanding-Optimism-Codebase/tree/main/op-wheel): Database utilities

## Brief Introduction：

- [**00-how-sequencer-generates-L2-blocks**](https://github.com/joohhnnn/Understanding-Optimism-Codebase/blob/main/sequencer/00-how-sequencer-generates-L2-blocks.md)
  - The sequencer plays a critical role in Layer 2 (L2) solutions, handling transaction aggregation, L1 data derivation, L2 block generation, and proposing the L2 state root in L1. This document explains the sequencer's process for generating L2 blocks, focusing on the creation of a block payload, event loop structure, code flow for block generation, and error handling.
  
- [**01-how-block-sync**](https://github.com/joohhnnn/Understanding-Optimism-Codebase/blob/main/sequencer/01-how-block-sync.md)
  - This document discusses the block propagation process in Optimism, detailing the three types of L2 blocks (Unsafe, Safe, and Finalized) and different syncing methods, including Op-node P2P gossip sync, reverse block header sync, execution layer sync, and RPC-based sync. It includes code explanations related to enabling these sync mechanisms.
  
- [**02-how-optimism-use-libp2p**](https://github.com/joohhnnn/Understanding-Optimism-Codebase/blob/main/sequencer/02-how-optimism-use-libp2p.md)
  - The document explains how Optimism leverages the libp2p network for block propagation and reverse sync. The focus is on the networking aspects and how blocks are broadcasted using libp2p's native streaming and request-response model.
  
- [**03-how-op-batcher-works**](https://github.com/joohhnnn/Understanding-Optimism-Codebase/blob/main/sequencer/03-how-batcher-works.md)
  - The batcher is responsible for transmitting Layer 2 data to Layer 1 in an optimized manner. It collects multiple L2 blocks and combines them into channels for efficient data compression. These are then sent to Layer 1 in multiple frames to reduce costs and increase efficiency. The document includes code for how the batcher processes blocks and sends transactions to Layer 1.
  
- [**04-how-derivation-works**](https://github.com/joohhnnn/Understanding-Optimism-Codebase/blob/main/sequencer/04-how-derivation-works.md)
  - This file explains the derivation process in Optimism. It covers how L1 transactions are processed to derive L2 blocks, including syncing data from L1, block derivation mechanisms, and how Optimism maintains the synchronization between L1 and L2.
  
- [**05-how-op-proposer-works**](https://github.com/joohhnnn/Understanding-Optimism-Codebase/blob/main/sequencer/05-how-proposer-works.md)
  - The proposer plays a role in proposing new L2 blocks to the rollup. The document likely details how the proposer functions within the broader rollup protocol.
  
- [**06-upgrade-of-opstack-in-EIP-4844**](https://github.com/joohhnnn/Understanding-Optimism-Codebase/blob/main/sequencer/06-upgrade-of-opstack-in-EIP-4844.md)
  - This document provides an overview of the upgrade in Optimism's OPStack related to Ethereum's EIP-4844. EIP-4844 introduces "blobs" that significantly reduce data availability costs for L2. The document explains how OPStack adopts these blobs for transmitting L2 data to L1, replacing calldata with a more efficient blob format.

- [**07-what-is-fault-proof**](https://github.com/joohhnnn/The-book-of-optimism-fault-proof/blob/main/01-what-is-fault-proof.md)
  - This document covers the concept of fault proofs, a mechanism used in Optimism to detect incorrect state transitions or invalid rollup assertions.

- [**08-fault-dispute-game**](https://github.com/joohhnnn/The-book-of-optimism-fault-proof/blob/main/02-fault-dispute-game.md)
  - This explains the dispute resolution process in Optimism, known as the fault dispute game. It's a challenge-response game designed to resolve discrepancies between the state of the rollup and its verifiers.

- [**09-cannon**](https://github.com/joohhnnn/The-book-of-optimism-fault-proof/blob/main/03-cannon.md)
  - This file covers "Cannon," the tool used to execute the fault-proof system. Cannon facilitates state verification to ensure integrity in the rollup’s execution.

- [**10-op-program**](https://github.com/joohhnnn/The-book-of-optimism-fault-proof/blob/main/04-op-program.md)
  - This document introduces the OP Program, which describes how Optimism's fault proof mechanism integrates into the overall architecture of the protocol.

- [**11-op-challenger**](https://github.com/joohhnnn/The-book-of-optimism-fault-proof/blob/main/05-op-challenger.md)
  - This explains the OP Challenger, the entity responsible for initiating challenges in the fault-proof system.

## Contact Information

If you have any questions or need assistance in developing public products, please don't hesitate to contact me via email at [joohhnnn8@gmail.com](mailto:joohhnnn8@gmail.com). Should I have available time, I'd be more than happy to assist.

