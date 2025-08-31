# How Batcher Works

In this section, we will explore what exactly is `batcher` ⚙️. The official specs include an introduction to batcher ([source](https://github.com/ethereum-optimism/optimism/blob/develop/specs/batcher.md)).

Before diving in, let's raise some questions to truly understand the role and working mechanism of `batcher`:
- What is `batcher`, and why is it called `batcher`?
- How does `batcher` actually operate in the code?

## Prerequisites

- In the rollup mechanism, to achieve decentralization features such as censorship resistance, we must transmit all data (transactions) occurring on layer 2 to layer 1. This way, we can leverage the security of layer 1 while constructing the complete layer 2 data from layer 1, making layer 2 truly effective.
- [Epochs and the Sequencing Window](https://github.com/ethereum-optimism/optimism/blob/develop/specs/overview.md#epochs-and-the-sequencing-window): An `epoch` can be simply understood as the time frame within which a new layer 1 block (`N+1`) is generated. The epoch number equals the number of the layer 1 block `N`, and all layer 2 blocks generated within the time frame `N -> N+1` belong to `epoch N`. So, when should the layer 2 data be uploaded to layer 1 to be effective? The size of the `Sequencing Window` gives us the answer, i.e., data related to block N/epoch N must be uploaded to layer 1 before block `N + size`.
- Batch/Batcher Transaction: A `batch` can be simply understood as the transactions required to build each layer 2 block. A `Batcher Transaction` is the transaction sent to layer 1 after multiple batches are processed and combined.
- [Channel](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#channel): A `channel` can be understood as a combination of `batches`, combined for better compression rates, thereby reducing the cost of data availability.
- [Frame](https://github.com/ethereum-optimism/optimism/blob/develop/specs/glossary.md#channel-frame): A `frame` can be understood as a partition of a `channel`. Sometimes, to achieve better compression, the `channel` data might be too large to be sent to layer 1 by `batcher` in one go, so it needs to be sliced and sent in multiple transmissions.

## What is Batcher

In rollup, a role is needed to transmit layer 2 information to layer 1, and sending new transactions immediately is expensive and difficult to manage. Therefore, we need to formulate a rational batch uploading strategy. This is where `batcher` comes in. Batcher is a unique entity (currently managed by the sequencer's private key) that sends `Batcher Transactions` to a specific address to transmit layer 2 information.

Batcher collects `unsafe` block data to acquire multiple batches. Here, each block corresponds to one batch. When enough batches are collected for efficient compression, they are turned into a `channel` and sent to layer 1 in the form of `frames` to complete the upload of layer 2 information.

## Code Implementation

In this part, we will deeply delve into the mechanism and implementation principle from a code perspective.

### Entry Point

`op-batcher/batcher/driver.go`

The `Start` function is called to initiate the `loop` cycle. In the loop, three main tasks are handled:
- When the timer triggers, all new, not yet loaded `L2blocks` are loaded, then the `publishStateToL1` function is triggered to publish `state` to layer 1.
- Handle `receipts`, recording success or failure status.
- Handle shutdown requests.

> **Source Code**: [op-batcher/batcher/driver.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-batcher/batcher/driver.go#L126)

```go
    func (l *BatchSubmitter) Start() error {
        l.log.Info("Starting Batch Submitter")

        l.mutex.Lock()
        defer l.mutex.Unlock()

        if l.running {
            return errors.New("batcher is already running")
        }
        l.running = true

        l.shutdownCtx, l.cancelShutdownCtx = context.WithCancel(context.Background())
        l.killCtx, l.cancelKillCtx = context.WithCancel(context.Background())
        l.state.Clear()
        l.lastStoredBlock = eth.BlockID{}

        l.wg.Add(1)
        go l.loop()

        l.log.Info("Batch Submitter started")

        return nil
    }
```

> **Source Code**: [op-batcher/batcher/driver.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-batcher/batcher/driver.go#L286)

```go
    func (l *BatchSubmitter) loop() {
        defer l.wg.Done()

        ticker := time.NewTicker(l.PollInterval)
        defer ticker.Stop()

        receiptsCh := make(chan txmgr.TxReceipt[txData])
        queue := txmgr.NewQueue[txData](l.killCtx, l.txMgr, l.MaxPendingTransactions)

        for {
            select {
            case <-ticker.C:
                if err := l.loadBlocksIntoState(l.shutdownCtx); errors.Is(err, ErrReorg) {
                    err := l.state.Close()
                    if err != nil {
                        l.log.Error("error closing the channel manager to handle a L2 reorg", "err", err)
                    }
                    l.publishStateToL1(queue, receiptsCh, true)
                    l.state.Clear()
                    continue
                }
                l.publishStateToL1(queue, receiptsCh, false)
            case r := <-receiptsCh:
                l.handleReceipt(r)
            case <-l.shutdownCtx.Done():
                err := l.state.Close()
                if err != nil {
                    l.log.Error("error closing the channel manager", "err", err)
                }
                l.publishStateToL1(queue, receiptsCh, true)
                return
            }
        }
    }
```

### Loading Latest Block Data
`op-batcher/batcher/driver.go`

The `loadBlocksIntoState` function calls `calculateL2BlockRangeToStore` to get the range of newly generated `unsafeblocks` derived from the latest `safeblock` since the last `batch transaction` was sent. It then iterates over this range, and for each `unsafe` block, calls the `loadBlockIntoState` function to fetch it from L2 and loads it into the internal `block queue` via the `AddL2Block` function, awaiting further processing.

> **Source Code**: [op-batcher/batcher/driver.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-batcher/batcher/driver.go#L191)

```go
    func (l *BatchSubmitter) loadBlocksIntoState(ctx context.Context) error {
        start, end, err := l.calculateL2BlockRangeToStore(ctx)
        ……
        var latestBlock *types.Block
        // Add all blocks to "state"
        for i := start.Number + 1; i < end.Number+1; i++ {
            block, err := l.loadBlockIntoState(ctx, i)
            if errors.Is(err, ErrReorg) {
                l.log.Warn("Found L2 reorg", "block_number", i)
                l.lastStoredBlock = eth.BlockID{}
                return err
            } else if err != nil {
                l.log.Warn("failed to load block into state", "err", err)
                return err
            }
            l.lastStoredBlock = eth.ToBlockID(block)
            latestBlock = block
        }
        ……
    }
```

> **Source Code**: [op-batcher/batcher/driver.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-batcher/batcher/driver.go#L227)

```go
    func (l *BatchSubmitter) loadBlockIntoState(ctx context.Context, blockNumber uint64) (*types.Block, error) {
        ……
        block, err := l.L2Client.BlockByNumber(ctx, new(big.Int).SetUint64(blockNumber))
        ……
        if err := l.state.AddL2Block(block); err != nil {
            return nil, fmt.Errorf("adding L2 block to state: %w", err)
        }
        ……
        return block, nil
    }
```

### Process Loaded Block Data and Send to Layer 1
`op-batcher/batcher/driver.go`

The `publishTxToL1` function uses the `TxData` function to process the previously loaded data and calls the `sendTransaction` function to send it to L1.

> **Source Code**: [op-batcher/batcher/driver.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-batcher/batcher/driver.go#L356)

```go
    func (l *BatchSubmitter) publishTxToL1(ctx context.Context, queue *txmgr.Queue[txData], receiptsCh chan txmgr.TxReceipt[txData]) error {
        // send all available transactions
        l1tip, err := l.l1Tip(ctx)
        if err != nil {
            l.log.Error("Failed to query L1 tip", "error", err)
            return err
        }
        l.recordL1Tip(l1tip)

        // Collect next transaction data
        txdata, err := l.state.TxData(l1tip.ID())
        if err == io.EOF {
            l.log.Trace("no transaction data available")
            return err
        } else if err != nil {
            l.log.Error("unable to get tx data", "err", err)
            return err
        }

        l.sendTransaction(txdata, queue, receiptsCh)
        return nil
    }
```

#### Detailed Explanation of TxData
`op-batcher/batcher/channel_manager.go`

The `TxData` function mainly handles two tasks:
- It looks for the first `channel` containing a `frame`. If one exists and passes inspection, it uses `nextTxData` to fetch and return the data.
- If no such `channel` exists, it first calls `ensureChannelWithSpace` to check if the `channel` has remaining space. Then it uses `processBlocks` to construct data from the previously loaded `block queue` into the `outchannel's composer` for compression.
- `outputFrames` slices the data in the `outchannel composer` into suitably sized `frames`.
- Finally, it returns the newly constructed data using the `nextTxData` function.

`EnsureChannelWithSpace` ensures that `currentChannel` is a `channel` with space to accommodate more data (i.e., `channel.IsFull` returns `false`). If `currentChannel` is null or full, a new `channel` is created.

> **Source Code**: [op-batcher/batcher/channel_manager.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-batcher/batcher/channel_manager.go#L136)

```go
    func (s *channelManager) TxData(l1Head eth.BlockID) (txData, error) {
        s.mu.Lock()
        defer s.mu.Unlock()
        var firstWithFrame *channel
        for _, ch := range s.channelQueue {
            if ch.HasFrame() {
                firstWithFrame = ch
                break
            }
        }

        dataPending := firstWithFrame != nil && firstWithFrame.HasFrame()
        s.log.Debug("Requested tx data", "l1Head", l1Head, "data_pending", dataPending, "blocks_pending", len(s.blocks))

        // Short circuit if there is a pending frame or the channel manager is closed.
        if dataPending || s.closed {
            return s.nextTxData(firstWithFrame)
        }

        // No pending frame, so we have to add new blocks to the channel

        // If we have no saved blocks, we will not be able to create valid frames
        if len(s.blocks) == 0 {
            return txData{}, io.EOF
        }

        if err := s.ensureChannelWithSpace(l1Head); err != nil {
            return txData{}, err
        }

        if err := s.processBlocks(); err != nil {
            return txData{}, err
        }

        // Register current L1 head only after all pending blocks have been
        // processed. Even if a timeout will be triggered now, it is better to have
        // all pending blocks be included in this channel for submission.
        s.registerL1Block(l1Head)

        if err := s.outputFrames(); err != nil {
            return txData{}, err
        }

        return s.nextTxData(s.currentChannel)
    }

```

The `processBlocks` function internally adds the `blocks` from the `block queue` into the current `channel` via `AddBlock`.

> **Source Code**: [op-batcher/batcher/channel_manager.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-batcher/batcher/channel_manager.go#L215)

```go
    func (s *channelManager) processBlocks() error {
        var (
            blocksAdded int
            _chFullErr  *ChannelFullError // throw away, just for type checking
            latestL2ref eth.L2BlockRef
        )
        for i, block := range s.blocks {
            l1info, err := s.currentChannel.AddBlock(block)
            if errors.As(err, &_chFullErr) {
                // current block didn't get added because channel is already full
                break
            } else if err != nil {
                return fmt.Errorf("adding block[%d] to channel builder: %w", i, err)
            }
            s.log.Debug("Added block to channel", "channel", s.currentChannel.ID(), "block", block)

            blocksAdded += 1
            latestL2ref = l2BlockRefFromBlockAndL1Info(block, l1info)
            s.metr.RecordL2BlockInChannel(block)
            // current block got added but channel is now full
            if s.currentChannel.IsFull() {
                break
            }
        }
```

The `AddBlock` function first uses `BlockToBatch` to extract the `batch` from the `block`, and then compresses and stores the data using the `AddBatch` function.

> **Source Code**: [op-batcher/batcher/channel_builder.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-batcher/batcher/channel_builder.go#L192)

```go
    func (c *channelBuilder) AddBlock(block *types.Block) (derive.L1BlockInfo, error) {
        if c.IsFull() {
            return derive.L1BlockInfo{}, c.FullErr()
        }

        batch, l1info, err := derive.BlockToBatch(block)
        if err != nil {
            return l1info, fmt.Errorf("converting block to batch: %w", err)
        }

        if _, err = c.co.AddBatch(batch); errors.Is(err, derive.ErrTooManyRLPBytes) || errors.Is(err, derive.CompressorFullErr) {
            c.setFullErr(err)
            return l1info, c.FullErr()
        } else if err != nil {
            return l1info, fmt.Errorf("adding block to channel out: %w", err)
        }
        c.blocks = append(c.blocks, block)
        c.updateSwTimeout(batch)

        if err = c.co.FullErr(); err != nil {
            c.setFullErr(err)
            // Adding this block still worked, so don't return error, just mark as full
        }

        return l1info, nil
    }
```

After obtaining the `txdata`, the entire data set is sent to Layer 1 using the `sendTransaction` function.

## Conclusion
In this section, we have learned what `batcher` is and how it operates. You can view the current behavior of the `batcher` at this [address](https://etherscan.io/address/0x6887246668a3b87f54deb3b94ba47a6f63f32985).
