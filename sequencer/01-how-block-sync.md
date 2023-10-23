# Block Propagation in Optimism

Block propagation is an important concept in the entire Optimism rollup system. In this chapter, we will unveil the process of block propagation in the whole system by introducing the principles behind various sync methods in Optimism.

## Types of Blocks

Before we go further, let's understand some basic concepts.

- **Unsafe L2 Block**:
    - This refers to the highest L2 block on the L1 chain, whose L1 origin is a *possible* extension of the canonical L1 chain (as known to the op-node). This means that although this block is linked to the L1 chain, its integrity and correctness have not yet been fully verified.

- **Safe L2 Block**:
    - This refers to the highest L2 block on the L1 chain, whose epoch sequence window is complete in the canonical L1 chain (as known to the op-node). This means that all the prerequisites of this block have been verified on the L1 chain, making it considered safe.

- **Finalized L2 Block**:
    - This refers to the L2 block known to be entirely derived from finalized L1 block data. This means that the block is not only safe but also fully confirmed according to the data of the L1 chain and will no longer be changed.

## Types of Sync

1. **Op-node p2p gossip sync**:
    - Op-node receives the latest unsafe blocks via the p2p gossip protocol, pushed by the sequencer.

2. **Op-node libp2p-based request-response reverse block header sync**:
    - Through this sync method, the op-node can fill in any gaps in unsafe blocks.

3. **Execution Layer (EL, also known as engine sync) sync**:
    - In op-node, there are two flags that allow unsafe blocks coming from gossip to trigger long-range syncs in the engine. The relevant flags are `--l2.engine-sync` and `--l2.skip-sync-start-check` (for handling very old safe blocks). Then, if EL is set up for this, it can perform any sync, such as snap-sync (requires op-geth p2p connections, etc., and needs to sync from some nodes).

4. **Op-node RPC sync**:
    - This is a simpler sync method based on trusted RPC methods when there are issues with L1.


## Op-node p2p gossip sync

This sync scenario occurs when a new L2 block is generated, specifically under the sequencer mode discussed in the previous section on how new blocks are produced.

After a new block is generated, the sequencer broadcasts to the 'new unsafe block' topic via the pub/sub (publish/subscribe) module of the libp2p-based P2P network. All nodes that have subscribed to this topic will directly or indirectly receive this broadcast message. [More details can be found here](https://github.com/joohhnnn/Understanding-Optimism-Codebase/blob/main/sequencer/02-how-optimism-use-libp2p.md#block-propagation-under-gossip).

## Op-node libp2p-based request-response reverse block header sync

This sync scenario occurs when, due to special circumstances such as a node being down and reconnecting, there may be some blocks that haven't been synced (gaps).

In this situation, fast syncing through the reverse chain method of the p2p network can be employed. This involves using libp2p's native streaming to establish connections with other p2p nodes while sending sync requests. [More details can be found here](https://github.com/joohhnnn/Understanding-Optimism-Codebase/blob/main/sequencer/02-how-optimism-use-libp2p.md#quick-sync-via-p2p-when-blocks-are-missing).

## Execution Layer (EL, also known as engine sync) sync

This sync scenario occurs when there are a large number of blocks, and a wide range of blocks needs to be synced. Syncing derived slowly from L1 could be too slow, and one may want fast syncing.

Use `--l2.engine-sync` and `--l2.skip-sync-start-check` flags to start the op-node, the payload is sent to initiate a long-range sync request.

### Code-level Explanation

First, let's look at the definitions of these two flags.

Defined and explained in `op-node/flags/flags.go`:

- **L2EngineSyncEnabled Flag (`--l2.engine-sync`)**:
    - This flag is used to enable or disable the execution engine's P2P sync functionality. When set to `true`, it allows the execution engine to sync block data with other nodes via the P2P network. Its default value is `false`, meaning that by default, this P2P sync functionality is disabled.

- **SkipSyncStartCheck Flag (`--l2.skip-sync-start-check`)**:
    - This flag is used to skip the reasonableness check for the L1 origin consistency of unsafe L2 blocks when determining the sync starting point. When set to `true`, it defers the verification of L1 origin. If you are using `--l2.engine-sync`, it is recommended to enable this flag to skip the initial consistency check. Its default value is `false`, meaning that by default, this reasonableness check is enabled.

```go
	L2EngineSyncEnabled = &cli.BoolFlag{
		Name:     "l2.engine-sync",
		Usage:    "Enables or disables execution engine P2P sync",
		EnvVars:  prefixEnvVars("L2_ENGINE_SYNC_ENABLED"),
		Required: false,
		Value:    false,
	}
	SkipSyncStartCheck = &cli.BoolFlag{
		Name: "l2.skip-sync-start-check",
		Usage: "Skip sanity check of consistency of L1 origins of the unsafe L2 blocks when determining the sync-starting point. " +
			"This defers the L1-origin verification, and is recommended to use in when utilizing l2.engine-sync",
		EnvVars:  prefixEnvVars("L2_SKIP_SYNC_START_CHECK"),
		Required: false,
		Value:    false,
	}
```

#### L2EngineSyncEnabled

The `L2EngineSyncEnabled` flag is used in `op-node` to trigger p2p sync within `op-geth` when a new unsafe payload (block) is received and sent for further validation. During the sync period, all unsafe blocks are treated as validated and proceed to the next step in the unsafe process. The internal p2p sync within `op-geth` is particularly useful for acquiring a long range of unsafe blocks. Actually, within `op-geth`, regardless of whether the `L2EngineSyncEnabled` flag is enabled or not, sync will be initiated to synchronize data when the parent block is not found.

Let's delve into the code level to take a closer look. 
Firstly, we have `op-node/rollup/derive/engine_queue.go`.

`EngineSync` is the concrete representation of the `L2EngineSyncEnabled` flag. Here, it is nested within two check functions.

```go
   // checkNewPayloadStatus checks returned status of engine_newPayloadV1 request for next unsafe payload.
   // It returns true if the status is acceptable.
   func (eq *EngineQueue) checkNewPayloadStatus(status eth.ExecutePayloadStatus) bool {
      if eq.syncCfg.EngineSync {
         // Allow SYNCING and ACCEPTED if engine P2P sync is enabled
         return status == eth.ExecutionValid || status == eth.ExecutionSyncing || status == eth.ExecutionAccepted
      }
      return status == eth.ExecutionValid
   }

   // checkForkchoiceUpdatedStatus checks returned status of engine_forkchoiceUpdatedV1 request for next unsafe payload.
   // It returns true if the status is acceptable.
   func (eq *EngineQueue) checkForkchoiceUpdatedStatus(status eth.ExecutePayloadStatus) bool {
      if eq.syncCfg.EngineSync {
         // Allow SYNCING if engine P2P sync is enabled
         return status == eth.ExecutionValid || status == eth.ExecutionSyncing
      }
      return status == eth.ExecutionValid
   }
```

Let's Shift Our Focus to `eth/catalyst/api.go` in op-geth
When the parent block is missing, it triggers sync and returns a SYNCING Status.

```go
   func (api *ConsensusAPI) newPayload(params engine.ExecutableData) (engine.PayloadStatusV1, error) {
      …
      // If the parent is missing, we - in theory - could trigger a sync, but that
      // would also entail a reorg. That is problematic if multiple sibling blocks
      // are being fed to us, and even more so, if some semi-distant uncle shortens
      // our live chain. As such, payload execution will not permit reorgs and thus
      // will not trigger a sync cycle. That is fine though, if we get a fork choice
      // update after legit payload executions.
      parent := api.eth.BlockChain().GetBlock(block.ParentHash(), block.NumberU64()-1)
      if parent == nil {
         return api.delayPayloadImport(block)
      }
      …
   }
```

```go
   func (api *ConsensusAPI) delayPayloadImport(block *types.Block) (engine.PayloadStatusV1, error) {
      …
      if err := api.eth.Downloader().BeaconExtend(api.eth.SyncMode(), block.Header()); err == nil {
         log.Debug("Payload accepted for sync extension", "number", block.NumberU64(), "hash", block.Hash())
         return engine.PayloadStatusV1{Status: engine.SYNCING}, nil
      }
      …
   }
```

#### SkipSyncStartCheck
The `SkipSyncStartCheck` flag mainly serves to optimize performance and reduce unnecessary checks in the sync mode selection. Once a qualifying L2 block is confirmed, the code skips further sanity checks to accelerate syncing or subsequent operations. This is an optimization technique, especially useful for operating quickly in situations with high certainty.

In the `op-node/rollup/sync/start.go` directory:

The `FindL2Heads` function backtracks from a given "start" point (i.e., a previous unsafe L2 block) to find these three types of blocks. During the backtracking, the function checks whether each L2 block's L1 origin matches the known L1 canonical chain, among other conditions and checks. This allows the function to more quickly determine the "safe" head of L2, potentially speeding up the entire sync process.

```go
   func FindL2Heads(ctx context.Context, cfg *rollup.Config, l1 L1Chain, l2 L2Chain, lgr log.Logger, syncCfg *Config) (result *FindHeadsResult, err error) {
      …
      for {
         …
         if syncCfg.SkipSyncStartCheck && highestL2WithCanonicalL1Origin.Hash == n.Hash {
            lgr.Info("Found highest L2 block with canonical L1 origin. Skip further sanity check and jump to the safe head")
            n = result.Safe
            continue
         }
         // Pull L2 parent for next iteration
         parent, err := l2.L2BlockRefByHash(ctx, n.ParentHash)
         if err != nil {
            return nil, fmt.Errorf("failed to fetch L2 block by hash %v: %w", n.ParentHash, err)
         }

         // Check the L1 origin relation
         if parent.L1Origin != n.L1Origin {
            // sanity check that the L1 origin block number is coherent
            if parent.L1Origin.Number+1 != n.L1Origin.Number {
               return nil, fmt.Errorf("l2 parent %s of %s has L1 origin %s that is not before %s", parent, n, parent.L1Origin, n.L1Origin)
            }
            // sanity check that the later sequence number is 0, if it changed between the L2 blocks
            if n.SequenceNumber != 0 {
               return nil, fmt.Errorf("l2 block %s has parent %s with different L1 origin %s, but non-zero sequence number %d", n, parent, parent.L1Origin, n.SequenceNumber)
            }
            // if the L1 origin is known to be canonical, then the parent must be too
            if l1Block.Hash == n.L1Origin.Hash && l1Block.ParentHash != parent.L1Origin.Hash {
               return nil, fmt.Errorf("parent L2 block %s has origin %s but expected %s", parent, parent.L1Origin, l1Block.ParentHash)
            }
         } else {
            if parent.SequenceNumber+1 != n.SequenceNumber {
               return nil, fmt.Errorf("sequence number inconsistency %d <> %d between l2 blocks %s and %s", parent.SequenceNumber, n.SequenceNumber, parent, n)
            }
         }

         n = parent

         // once we found the block at seq nr 0 that is more than a full seq window behind the common chain post-reorg, then use the parent block as safe head.
         if ready {
            result.Safe = n
            return result, nil
         }
      }
   }
```
### op-node RPC Sync

This sync scenario occurs when you have a trusted l2 rpc node. You can communicate directly with the rpc, sending short-range sync requests, similar to method 2. If configured, RPC will be prioritized over P2P sync in reverse chain synchronization.

#### Key Code

`op-node/node/node.go`

Initialize rpcSync; if rpcSyncClient is set, assign it to rpcSync.


```go
   func (n *OpNode) initRPCSync(ctx context.Context, cfg *Config) error {
      rpcSyncClient, rpcCfg, err := cfg.L2Sync.Setup(ctx, n.log, &cfg.Rollup)
      if err != nil {
         return fmt.Errorf("failed to setup L2 execution-engine RPC client for backup sync: %w", err)
      }
      if rpcSyncClient == nil { // if no RPC client is configured to sync from, then don't add the RPC sync client
         return nil
      }
      syncClient, err := sources.NewSyncClient(n.OnUnsafeL2Payload, rpcSyncClient, n.log, n.metrics.L2SourceCache, rpcCfg)
      if err != nil {
         return fmt.Errorf("failed to create sync client: %w", err)
      }
      n.rpcSync = syncClient
      return nil
   }
```

Initialize the node; if `rpcSync` is not null, start the rpcSync event loop.


```go
   func (n *OpNode) Start(ctx context.Context) error {
      n.log.Info("Starting execution engine driver")

      // start driving engine: sync blocks by deriving them from L1 and driving them into the engine
      if err := n.l2Driver.Start(); err != nil {
         n.log.Error("Could not start a rollup node", "err", err)
         return err
      }

      // If the backup unsafe sync client is enabled, start its event loop
      if n.rpcSync != nil {
         if err := n.rpcSync.Start(); err != nil {
            n.log.Error("Could not start the backup sync client", "err", err)
            return err
         }
         n.log.Info("Started L2-RPC sync service")
      }

      return nil
   }
```

`op-node/sources/sync_client.go`

Once a signal (block number) is received in the `s.requests` channel, the `fetchUnsafeBlockFromRpc` function is called to fetch the corresponding block information from the RPC node.


```go
   // eventLoop is the main event loop for the sync client.
   func (s *SyncClient) eventLoop() {
      defer s.wg.Done()
      s.log.Info("Starting sync client event loop")

      backoffStrategy := &retry.ExponentialStrategy{
         Min:       1000 * time.Millisecond,
         Max:       20_000 * time.Millisecond,
         MaxJitter: 250 * time.Millisecond,
      }

      for {
         select {
         case <-s.resCtx.Done():
            s.log.Debug("Shutting down RPC sync worker")
            return
         case reqNum := <-s.requests:
            _, err := retry.Do(s.resCtx, 5, backoffStrategy, func() (interface{}, error) {
               // Limit the maximum time for fetching payloads
               ctx, cancel := context.WithTimeout(s.resCtx, time.Second*10)
               defer cancel()
               // We are only fetching one block at a time here.
               return nil, s.fetchUnsafeBlockFromRpc(ctx, reqNum)
            })
            if err != nil {
               if err == s.resCtx.Err() {
                  return
               }
               s.log.Error("failed syncing L2 block via RPC", "err", err, "num", reqNum)
               // Reschedule at end of queue
               select {
               case s.requests <- reqNum:
               default:
                  // drop syncing job if we are too busy with sync jobs already.
               }
            }
         }
      }
   }
```

Next, let's see where the signals are sent to the `s.requests` channel. In the same file, the `RequestL2Range` function introduces a range of blocks that need to be synchronized. The function then sends out these tasks one by one using a `for` loop.

```go
   func (s *SyncClient) RequestL2Range(ctx context.Context, start, end eth.L2BlockRef) error {
      // Drain previous requests now that we have new information
      for len(s.requests) > 0 {
         select { // in case requests is being read at the same time, don't block on draining it.
         case <-s.requests:
         default:
            break
         }
      }

      endNum := end.Number
      if end == (eth.L2BlockRef{}) {
         n, err := s.rollupCfg.TargetBlockNumber(uint64(time.Now().Unix()))
         if err != nil {
            return err
         }
         if n <= start.Number {
            return nil
         }
         endNum = n
      }

      // TODO(CLI-3635): optimize the by-range fetching with the Engine API payloads-by-range method.

      s.log.Info("Scheduling to fetch trailing missing payloads from backup RPC", "start", start, "end", endNum, "size", endNum-start.Number-1)

      for i := start.Number + 1; i < endNum; i++ {
         select {
         case s.requests <- i:
         case <-ctx.Done():
            return ctx.Err()
         }
      }
      return nil
   }
```

In the outer `OpNode` type's implementation of the `RequestL2Range` method, it's clear to see that `rpcSync` type of reverse chain synchronization is prioritized.

```go
   func (n *OpNode) RequestL2Range(ctx context.Context, start, end eth.L2BlockRef) error {
      if n.rpcSync != nil {
         return n.rpcSync.RequestL2Range(ctx, start, end)
      }
      if n.p2pNode != nil && n.p2pNode.AltSyncEnabled() {
         if unixTimeStale(start.Time, 12*time.Hour) {
            n.log.Debug("ignoring request to sync L2 range, timestamp is too old for p2p", "start", start, "end", end, "start_time", start.Time)
            return nil
         }
         return n.p2pNode.RequestL2Range(ctx, start, end)
      }
      n.log.Debug("ignoring request to sync L2 range, no sync method available", "start", start, "end", end)
      return nil
   }
```

## Summary
After understanding these synchronization methods, we now know how unsafe payloads (blocks) are actually transmitted. Different sync modules correspond to the transmission of block data in different scenarios. So, how are unsafe blocks in the entire network step-by-step transformed into safe blocks, and then finalized? These topics will be discussed in other sections.
