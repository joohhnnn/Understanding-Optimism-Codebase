# Understanding Optimism Codebase 

This document provides a comprehensive explanation of the Optimism codebase, aiming to help newcomers to Optimism quickly get started and truly understand how the code flow in the codebase works.


## Project Directory

### Working on：
- [**sequencer**](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/sequencer): sequencer part
### TODO：
- [docs](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/docs): A collection of documents including audits and post-mortems
- [op-bindings](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/op-bindings): Go bindings for Bedrock smart contracts.
- [op-batcher](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/op-batcher): L2-Batch Submitter, submits bundles of batches to L1
- [op-bootnode](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/op-bootnode): Standalone op-node discovery bootnode
- [op-chain-ops](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/op-chain-ops): State surgery utilities
- [op-challenger](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/op-challenger): Dispute game challenge agent
- [op-e2e](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/op-e2e): End-to-End testing of all bedrock components in Go
- [op-exporter](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/op-exporter): Prometheus exporter client
- [op-heartbeat](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/op-heartbeat): Heartbeat monitor service
- [op-node](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/op-node): rollup consensus-layer client
- [op-preimage](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/op-preimage): Go bindings for Preimage Oracle
- [op-program](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/op-program): Fault proof program
- [op-proposer](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/op-proposer): L2-Output Submitter, submits proposals to L1
- [op-service](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/op-service): Common codebase utilities
- [op-signer](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/op-signer): Client signer
- [op-wheel](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/op-wheel): Database utilities
- [ops-bedrock](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/ops-bedrock): Bedrock devnet work
- [packages](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/packages)
  - [chain-mon](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/packages/chain-mon): Chain monitoring services
  - [common-ts](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/packages/common-ts): Common tools for building apps in TypeScript
  - [contracts-ts](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/packages/contracts-ts): ABI and Address constants
  - [contracts-bedrock](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/packages/contracts-bedrock): Bedrock smart contracts
  - [core-utils](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/packages/core-utils): Low-level utilities that make building Optimism easier
  - [sdk](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/packages/sdk): provides a set of tools for interacting with Optimism
- [proxyd](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/proxyd): Configurable RPC request router and proxy
- [specs](https://github.com/joohhnnn/Understanding-Optimism-Codebase-CN/tree/main/specs): Specs of the rollup starting at the Bedrock upgrade