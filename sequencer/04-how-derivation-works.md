# How Opstack Derives Layer2 from Layer1
Before reading this article, I highly recommend you first read the introduction about derivation in the `optimism/specs` ([source](https://github.com/ethereum-optimism/optimism/blob/develop/specs/derivation.md#deriving-payload-attributes)). If you find yourself confused after reading that article, that's okay. Just remember the feeling because, after reading our analysis, you'll find that the official article is very concise and covers all the essential points.

Let's dive into the main topic. We all know that the nodes running on layer2 can fetch data from the DA layer (layer1) and build complete block data. Today, we will discuss how this is implemented in the `codebase`.

## Questions You Should Have

If you were to design such a system, how would you go about it? What questions would you have? Here are some questions that may help you better understand this article:
- How does the entire system operate when you launch a new node?
- Do you need to query all the l1 block data one by one? How is this triggered?
- What data do you need after getting l1 block data?
- During the derivation process, how does the block state change? How does it go from `unsafe` to `safe` and then to `finalized`?
- What are the opaque data structures like `batch/channel/frame` in the official specs? (You can understand this in detail in the previous chapter `03-how-batcher-works`)

## What is Derivation?

Before understanding `derivation`, let's discuss the basic rollup mechanism of optimism, using a simple l2 transfer transaction as an example.

When you issue a transfer transaction on the Optimism network, this transaction is "forwarded" to the `sequencer` node. The sequencer sorts it and encapsulates it into a block, which can be understood as block production. We call this block containing your transaction `Block A`. The current state of `Block A` is `unsafe`. Then after a certain time interval (e.g., 4 minutes), the `batcher` module in the `sequencer` sends all the collected transactions (including your transfer) to l1 in a single transaction, producing Block X on l1. The state of Block A remains `unsafe`. When any node executes the `derivation` program, it fetches Block X data from l1 and updates `the local l2 unsafe Block A`. The state of `Block A` becomes `safe`. After `two epochs (64 blocks)` on l1, the l2 node marks Block A as a `finalized` block.

Derivation is the process of transitioning `unsafe` blocks to `safe` and finally to `finalized` by continuously running the `derivation` program.

## Deep Dive into Code

Ahoy, Captain, let's deep dive🤿

### Obtaining the Data of Batch Transactions Sent by Batcher

First, let's see how to check whether there is `batch transactions` data in the block when we know a new l1 block.
Here we first sort out the modules we need and then look at these modules.
- First, determine what the next l1 block number is.
- Parse the data from the next block.

#### Determine the Next Block Number
`op-node/rollup/derive/l1_traversal.go`

By querying the current `origin.Number + 1` block height, we obtain the latest l1 block. If this block does not exist, i.e., `error` matches `ethereum.NotFound`, it means that the current block height is the latest, and the next block has not yet been produced on l1. If successfully retrieved, the latest block number is recorded in `l1t.block`.

> **Source Code**: [op-node/rollup/derive/l1_traversal.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-node/rollup/derive/l1_traversal.go#L60)

```go
    func (l1t *L1Traversal) AdvanceL1Block(ctx context.Context) error {
        origin := l1t.block
        nextL1Origin, err := l1t.l1Blocks.L1BlockRefByNumber(ctx, origin.Number+1)
        if errors.Is(err, ethereum.NotFound) {
            l1t.log.Debug("can't find next L1 block info (yet)", "number", origin.Number+1, "origin", origin)
            return io.EOF
        } else if err != nil {
            return NewTemporaryError(fmt.Errorf("failed to find L1 block info by number, at origin %s next %d: %w", origin, origin.Number+1, err))
        }
        if l1t.block.Hash != nextL1Origin.ParentHash {
            return NewResetError(fmt.Errorf("detected L1 reorg from %s to %s with conflicting parent %s", l1t.block, nextL1Origin, nextL1Origin.ParentID()))
        }

        ……

        l1t.block = nextL1Origin
        l1t.done = false
        return nil
    }
```

#### Parsing the Block Data
`op-node/rollup/derive/calldata_source.go`

Firstly, we use `InfoAndTxsByHash` to fetch all `transactions` from the block we just acquired. Then we pass these `transactions`, along with our `batcherAddr` and our `config`, into the `DataFromEVMTransactions` function.
Why do we pass these parameters? Because when we are filtering these transactions, we need to ensure the accuracy (authority) of the `batcher` address and the recipient address. After `DataFromEVMTransactions` receives these parameters, it loops through each transaction to filter based on the address accuracy, thereby identifying the correct `batch transactions`.

> **Source Code**: [op-node/rollup/derive/calldata_source.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-node/rollup/derive/calldata_source.go#L62)

```go
    func NewDataSource(ctx context.Context, log log.Logger, cfg *rollup.Config, fetcher L1TransactionFetcher, block eth.BlockID, batcherAddr common.Address) DataIter {
        _, txs, err := fetcher.InfoAndTxsByHash(ctx, block.Hash)
        if err != nil {
            return &DataSource{
                open:        false,
                id:          block,
                cfg:         cfg,
                fetcher:     fetcher,
                log:         log,
                batcherAddr: batcherAddr,
            }
        } else {
            return &DataSource{
                open: true,
                data: DataFromEVMTransactions(cfg, batcherAddr, txs, log.New("origin", block)),
            }
        }
    }
```

> **Source Code**: [op-node/rollup/derive/calldata_source.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-node/rollup/derive/calldata_source.go#L107)

```go
    func DataFromEVMTransactions(config *rollup.Config, batcherAddr common.Address, txs types.Transactions, log log.Logger) []eth.Data {
        var out []eth.Data
        l1Signer := config.L1Signer()
        for j, tx := range txs {
            if to := tx.To(); to != nil && *to == config.BatchInboxAddress {
                seqDataSubmitter, err := l1Signer.Sender(tx) // optimization: only derive sender if To is correct
                if err != nil {
                    log.Warn("tx in inbox with invalid signature", "index", j, "err", err)
                    continue // bad signature, ignore
                }
                // some random L1 user might have sent a transaction to our batch inbox, ignore them
                if seqDataSubmitter != batcherAddr {
                    log.Warn("tx in inbox with unauthorized submitter", "index", j, "err", err)
                    continue // not an authorized batch submitter, ignore
                }
                out = append(out, tx.Data())
            }
        }
        return out
    }
```

### From Data to SafeAttribute: Making Unsafe Blocks Safe

In this section, we first parse the `data` obtained in the previous step into a `frame` and add it to the `FrameQueue`'s `frames` array. Then, we extract a `frame` from the `frames` array and initialize it into a `channel`, which is then added to the `channelbank`. We wait for all `frames` in that `channel` to be added. Afterward, we extract `batch` information from the `channel` and add it to the `BatchQueue`. Finally, we add the `batch` from the `BatchQueue` to the `AttributesQueue` to construct `safeAttributes`, update the `safeblock` in the `enginequeue`, and ultimately update the EL layer's `safeblock` through the invocation of the `ForkchoiceUpdate` function.

#### Data -> Frame
`op-node/rollup/derive/frame_queue.go`

This function uses the `NextData` function to obtain the data from the previous step. It then parses this data and adds it to the `FrameQueue`'s `frames` array, returning the first `frame` in the array.

> **Source Code**: [op-node/rollup/derive/frame_queue.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-node/rollup/derive/frame_queue.go#L36)

```go
    func (fq *FrameQueue) NextFrame(ctx context.Context) (Frame, error) {
        // Find more frames if we need to
        if len(fq.frames) == 0 {
            if data, err := fq.prev.NextData(ctx); err != nil {
                return Frame{}, err
            } else {
                if new, err := ParseFrames(data); err == nil {
                    fq.frames = append(fq.frames, new...)
                } else {
                    fq.log.Warn("Failed to parse frames", "origin", fq.prev.Origin(), "err", err)
                }
            }
        }
        // If we did not add more frames but still have more data, retry this function.
        if len(fq.frames) == 0 {
            return Frame{}, NotEnoughData
        }

        ret := fq.frames[0]
        fq.frames = fq.frames[1:]
        return ret, nil
    }
```

#### Frame -> Channel
`op-node/rollup/derive/channel_bank.go`

The `NextData` function is responsible for reading and returning the `raw data` from the first `channel` in the current `channel bank`. It also calls `NextFrame` to fetch the `frame` and load it into the `channel`.

> **Source Code**: [op-node/rollup/derive/channel_bank.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-node/rollup/derive/channel_bank.go#L158)

```go
    func (cb *ChannelBank) NextData(ctx context.Context) ([]byte, error) {
        // Do the read from the channel bank first
        data, err := cb.Read()
        if err == io.EOF {
            // continue - We will attempt to load data into the channel bank
        } else if err != nil {
            return nil, err
        } else {
            return data, nil
        }

        // Then load data into the channel bank
        if frame, err := cb.prev.NextFrame(ctx); err == io.EOF {
            return nil, io.EOF
        } else if err != nil {
            return nil, err
        } else {
            cb.IngestFrame(frame)
            return nil, NotEnoughData
        }
    }
```

#### Channel -> Batch
`op-node/rollup/derive/channel_in_reader.go`

The `NextBatch` function mainly decodes the previously fetched `raw data` into data with a `batch` structure and returns it. The role of the `WriteChannel` function is to provide a function and assign it to `nextBatchFn`. The purpose of this function is to create a reader, decode data with a `batch` structure from the reader, and return it.

> **Source Code**: [op-node/rollup/derive/channel_in_reader.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-node/rollup/derive/channel_in_reader.go#L63)

```go
    func (cr *ChannelInReader) NextBatch(ctx context.Context) (*BatchData, error) {
        if cr.nextBatchFn == nil {
            if data, err := cr.prev.NextData(ctx); err == io.EOF {
                return nil, io.EOF
            } else if err != nil {
                return nil, err
            } else {
                if err := cr.WriteChannel(data); err != nil {
                    return nil, NewTemporaryError(err)
                }
            }
        }

        // TODO: can batch be non nil while err == io.EOF
        // This depends on the behavior of rlp.Stream
        batch, err := cr.nextBatchFn()
        if err == io.EOF {
            cr.NextChannel()
            return nil, NotEnoughData
        } else if err != nil {
            cr.log.Warn("failed to read batch from channel reader, skipping to next channel now", "err", err)
            cr.NextChannel()
            return nil, NotEnoughData
        }
        return batch.Batch, nil
    }
```

**Note❗️ Here, the batch generated by the `NextBatch` function is not directly used; instead, it's first added to `batchQueue` for unified management and use. Additionally, `NextBatch` is actually called by the `func (bq *BatchQueue) NextBatch()` function in the `op-node/rollup/derive/batch_queue.go` directory.**
#### Batch -> SafeAttributes
**Supplementary Information:**
1. In layer2 blocks, the first transaction in the block is always an `anchor transaction`, which can be understood as containing some L1 information. If this layer2 block also happens to be the first block in an epoch, it will also contain `deposit` transactions from layer1 ([example of the first block in an epoch](https://optimistic.etherscan.io/txs?block=110721915]).
2. The term "batch" here should not be confused with the batch transactions sent by the batcher. For example, let's name the batch transaction sent by the batcher as batchA, while the term "batch" we discuss here is batchB. The relationship between batchA and batchB is one of containment; batchA may contain a large number of transactions that could be constructed into batchB, batchBB, batchBBB, etc. BatchB corresponds to a block's transactions in layer2, while batchA corresponds to a large number of layer2 block transactions.

`op-node/rollup/derive/attributes_queue.go`
- The `NextAttributes` function, after receiving the current L2's safe block header, passes this header and the batch we obtained in the previous step to the `createNextAttributes` function to construct `safeAttributes`.
- Something to note in `createNextAttributes` is that it internally calls the `PreparePayloadAttributes` function, which mainly handles anchor and `deposit` transactions. Finally, the transactions from `batch` and the transactions returned by `PreparePayloadAttributes` are concatenated and returned.

The `createNextAttributes` function internally calls `PreparePayloadAttributes`.

> **Source Code**: [op-node/rollup/derive/attributes_queue.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-node/rollup/derive/attributes_queue.go#L51)

```go
    func (aq *AttributesQueue) NextAttributes(ctx context.Context, l2SafeHead eth.L2BlockRef) (*eth.PayloadAttributes, error) {
        // Get a batch if we need it
        if aq.batch == nil {
            batch, err := aq.prev.NextBatch(ctx, l2SafeHead)
            if err != nil {
                return nil, err
            }
            aq.batch = batch
        }

        // Actually generate the next attributes
        if attrs, err := aq.createNextAttributes(ctx, aq.batch, l2SafeHead); err != nil {
            return nil, err
        } else {
            // Clear out the local state once we will succeed
            aq.batch = nil
            return attrs, nil
        }

    }
```

> **Source Code**: [op-node/rollup/derive/attributes_queue.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-node/rollup/derive/attributes_queue.go#L74)

```go
    func (aq *AttributesQueue) createNextAttributes(ctx context.Context, batch *BatchData, l2SafeHead eth.L2BlockRef) (*eth.PayloadAttributes, error) {
        
        ……
        attrs, err := aq.builder.PreparePayloadAttributes(fetchCtx, l2SafeHead, batch.Epoch())
        ……

        return attrs, nil
    }
```

```go
    func (aq *AttributesQueue) createNextAttributes(ctx context.Context, batch *BatchData, l2SafeHead eth.L2BlockRef) (*eth.PayloadAttributes, error) {
        // sanity check parent hash
        if batch.ParentHash != l2SafeHead.Hash {
            return nil, NewResetError(fmt.Errorf("valid batch has bad parent hash %s, expected %s", batch.ParentHash, l2SafeHead.Hash))
        }
        // sanity check timestamp
        if expected := l2SafeHead.Time + aq.config.BlockTime; expected != batch.Timestamp {
            return nil, NewResetError(fmt.Errorf("valid batch has bad timestamp %d, expected %d", batch.Timestamp, expected))
        }
        fetchCtx, cancel := context.WithTimeout(ctx, 20*time.Second)
        defer cancel()
        attrs, err := aq.builder.PreparePayloadAttributes(fetchCtx, l2SafeHead, batch.Epoch())
        if err != nil {
            return nil, err
        }

        // we are verifying, not sequencing, we've got all transactions and do not pull from the tx-pool
        // (that would make the block derivation non-deterministic)
        attrs.NoTxPool = true
        attrs.Transactions = append(attrs.Transactions, batch.Transactions...)

        aq.log.Info("generated attributes in payload queue", "txs", len(attrs.Transactions), "timestamp", batch.Timestamp)

        return attrs, nil
    }
```

#### SafeAttributes -> Safe Block
At this step, the `safehead` in the `engine queue` is first set to `safe`. However, this doesn't mean the block has become `safe`; it still needs to be updated in the EL (Execution Layer) through `ForkchoiceUpdate`.

`op-node/rollup/derive/engine_queue.go`

The `tryNextSafeAttributes` function internally assesses the relationship between the current `safehead` and `unsafehead`. If everything is normal, it triggers the `consolidateNextSafeAttributes` function. This function sets the `safeHead` in the `engine queue` as the `safe` block constructed from the `safeAttributes` we obtained in the previous step. It also sets `needForkchoiceUpdate` to `true`, triggering the subsequent `ForkchoiceUpdate` to change the block status in the EL to `safe`, thus truly converting an `unsafe` block into a `safe` block. The final function, `postProcessSafeL2`, adds the `safehead` to the `finalizedL1` queue for subsequent `finalization` use.

> **Source Code**: [op-node/rollup/derive/engine_queue.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-node/rollup/derive/engine_queue.go#L548)

```go
    func (eq *EngineQueue) tryNextSafeAttributes(ctx context.Context) error {
        ……
        if eq.safeHead.Number < eq.unsafeHead.Number {
            return eq.consolidateNextSafeAttributes(ctx)
        } 
        ……
    }

    func (eq *EngineQueue) consolidateNextSafeAttributes(ctx context.Context) error {
        ……
        payload, err := eq.engine.PayloadByNumber(ctx, eq.safeHead.Number+1)
        ……
        ref, err := PayloadToBlockRef(payload, &eq.cfg.Genesis)
        ……
        eq.safeHead = ref
        eq.needForkchoiceUpdate = true
        eq.postProcessSafeL2()
        ……
        return nil
    }
```

### Finalizing Safe Blocks
A safe block is not truly secure; it needs to undergo further finalization to become `finalized`. When a block's status changes to `safe`, the count starts from the originating `L1 (batcher transaction)`. After two `L1 epochs (64 blocks)`, this `safe` block can be updated to a `finalized` status.

`op-node/rollup/derive/engine_queue.go`

The `tryFinalizePastL2Blocks` function internally checks blocks in the `finalized queue` against a 64-block criterion. If it passes the check, `tryFinalizeL2` is called to finalize the setting in the `engine queue` and update the `needForkchoiceUpdate` marker.

> **Source Code**: [op-node/rollup/derive/engine_queue.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-node/rollup/derive/engine_queue.go#L328)

```go
    func (eq *EngineQueue) tryFinalizePastL2Blocks(ctx context.Context) error {
        ……
        eq.log.Info("processing L1 finality information", "l1_finalized", eq.finalizedL1, "l1_origin", eq.origin, "previous", eq.triedFinalizeAt) //const finalityDelay untyped int = 64

        // Sanity check we are indeed on the finalizing chain, and not stuck on something else.
        // We assume that the block-by-number query is consistent with the previously received finalized chain signal
        ref, err := eq.l1Fetcher.L1BlockRefByNumber(ctx, eq.origin.Number)
        if err != nil {
            return NewTemporaryError(fmt.Errorf("failed to check if on finalizing L1 chain: %w", err))
        }
        if ref.Hash != eq.origin.Hash {
            return NewResetError(fmt.Errorf("need to reset, we are on %s, not on the finalizing L1 chain %s (towards %s)", eq.origin, ref, eq.finalizedL1))
        }
        eq.tryFinalizeL2()
        return nil
    }
```

> **Source Code**: [op-node/rollup/derive/engine_queue.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-node/rollup/derive/engine_queue.go#L363)

```go
    func (eq *EngineQueue) tryFinalizeL2() {
        if eq.finalizedL1 == (eth.L1BlockRef{}) {
            return // if no L1 information is finalized yet, then skip this
        }
        eq.triedFinalizeAt = eq.origin
        // default to keep the same finalized block
        finalizedL2 := eq.finalized
        // go through the latest inclusion data, and find the last L2 block that was derived from a finalized L1 block
        for _, fd := range eq.finalityData {
            if fd.L2Block.Number > finalizedL2.Number && fd.L1Block.Number <= eq.finalizedL1.Number {
                finalizedL2 = fd.L2Block
                eq.needForkchoiceUpdate = true
            }
        }
        eq.finalized = finalizedL2
        eq.metrics.RecordL2Ref("l2_finalized", finalizedL2)
    }
```

### Loop Triggering
The `eventLoop` function in `op-node/rollup/driver/state.go` is responsible for triggering the entry point of the entire loop process. It mainly indirectly executes the `Step` function found in `op-node/rollup/derive/engine_queue.go`.

> **Source Code**: [op-node/rollup/derive/engine_queue.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-node/rollup/derive/engine_queue.go#L245)

```go
func (eq *EngineQueue) Step(ctx context.Context) error {
	if eq.needForkchoiceUpdate {
		return eq.tryUpdateEngine(ctx)
	}
	// Trying unsafe payload should be done before safe attributes
	// It allows the unsafe head can move forward while the long-range consolidation is in progress.
	if eq.unsafePayloads.Len() > 0 {
		if err := eq.tryNextUnsafePayload(ctx); err != io.EOF {
			return err
		}
		// EOF error means we can't process the next unsafe payload. Then we should process next safe attributes.
	}
	if eq.isEngineSyncing() {
		// Make pipeline first focus to sync unsafe blocks to engineSyncTarget
		return EngineP2PSyncing
	}
	if eq.safeAttributes != nil {
		return eq.tryNextSafeAttributes(ctx)
	}
	outOfData := false
	newOrigin := eq.prev.Origin()
	// Check if the L2 unsafe head origin is consistent with the new origin
	if err := eq.verifyNewL1Origin(ctx, newOrigin); err != nil {
		return err
	}
	eq.origin = newOrigin
	eq.postProcessSafeL2() // make sure we track the last L2 safe head for every new L1 block
	// try to finalize the L2 blocks we have synced so far (no-op if L1 finality is behind)
	if err := eq.tryFinalizePastL2Blocks(ctx); err != nil {
		return err
	}
	if next, err := eq.prev.NextAttributes(ctx, eq.safeHead); err == io.EOF {
		outOfData = true
	} else if err != nil {
		return err
	} else {
		eq.safeAttributes = &attributesWithParent{
			attributes: next,
			parent:     eq.safeHead,
		}
		eq.log.Debug("Adding next safe attributes", "safe_head", eq.safeHead, "next", next)
		return NotEnoughData
	}

	if outOfData {
		return io.EOF
	} else {
		return nil
	}
}
```

## Summary
The whole `derivation` function may seem very complex, but if you break down each link, you'll get a good grasp of it. The reason the official specs are hard to understand is that the concepts of `batch`, `frame`, `channel`, etc., are easily confusing. Therefore, if you are still puzzled after reading this article, it's suggested that you go back and review our `03-how-batcher-works`.
