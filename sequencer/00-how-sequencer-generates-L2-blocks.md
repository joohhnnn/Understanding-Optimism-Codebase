
# Sequencer Workflow

Sequencer plays a core role in Layer 2 (L2) solutions, mainly responsible for transaction aggregation, L1 data derivation, L2 block generation, L1 batch data submission, and proposing the L2 state root in L1. In this article, we will delve deeply into the working principles of the Sequencer and the associated code implementation. In this part, we primarily discuss the L2 block generation process.

## L2 Block Generation

On a more macroscopic level, the sequencer, during the L2 block generation process, essentially creates a template block payload that contains only deposits. This payload is then sent to the Execution Layer (EL), where EL extracts transactions from the txpool and then wraps the payload to generate the actual block.

### Code Flow

After the operation node (opnode) starts up, the Driver initiates an event loop. Within this event loop, we have defined the `sequencerCh` channel and the `planSequencerAction` method.

```go
sequencerTimer := time.NewTimer(0)
var sequencerCh <-chan time.Time
planSequencerAction := func() {
    delay := s.sequencer.PlanNextSequencerAction()
    sequencerCh = sequencerTimer.C
    if len(sequencerCh) > 0 { // Make sure the channel is cleared before resetting
        <-sequencerCh
    }
    sequencerTimer.Reset(delay)
}
```

In the `planSequencerAction` method, we reset the timer for receiving channel signals. Meanwhile, the `PlanNextSequencerAction` method is used to calculate the delay time for `RunNextSequencerAction`.

#### Delay Time Explanation

Here, "delay time" is an important concept. It dictates the amount of time to wait before executing the next serialization action. By dynamically calculating the delay time, we can control the frequency and timing of serialization more flexibly, thereby optimizing the system's performance.

### Event Loop Circular Structure

In the for loop of the event loop, a series of checks are performed initially. For example, we check whether the sequencer is enabled and whether the L1 state is ready, to determine if the next sequencer operation can be triggered.

```go
for {    
    // Main condition: Check if the Sequencer is enabled and if the L1 state is ready
    // In this if statement, we check several key conditions to determine whether sequencing can proceed, including:
    // - Whether the Sequencer is enabled
    // - Whether the Sequencer is stopped
    // - Whether the L1 state is ready
    // - Whether the derivation pipeline engine is ready
    if s.driverConfig.SequencerEnabled && !s.driverConfig.SequencerStopped &&
        s.l1State.L1Head() != (eth.L1BlockRef{}) && s.derivation.EngineReady() {

        // Checking the safe lag
        // In this code segment, we monitor the lag between the safe and unsafe L2 heads to determine whether to pause the creation of new blocks.
        if s.driverConfig.SequencerMaxSafeLag > 0 && s.derivation.SafeL2Head().Number+s.driverConfig.SequencerMaxSafeLag <= s.derivation.UnsafeL2Head().Number {
            if sequencerCh != nil {
                s.log.Warn(
                    "Delay creating new block since safe lag exceeds limit",
                    "safe_l2", s.derivation.SafeL2Head(),
                    "unsafe_l2", s.derivation.UnsafeL2Head(),
                )
                sequencerCh = nil
            }
        // Updating Sequencer operations
        // If the sequencer is building onto a new block and the L1 state is ready, we update the trigger for the next sequencer action.
        } else if s.sequencer.BuildingOnto().ID() != s.derivation.UnsafeL2Head().ID() {
            planSequencerAction()
        }
    // Default condition: In all other cases, we set sequencerCh to nil, meaning no new sequencer actions are planned.
    } else {
        sequencerCh = nil
    }
}
```

### Conclusion

In the loop structure of the eventloop, we conducted a series of checks to determine whether the next sequencer operation can be triggered.

During the checking process, the timer was set for the first time through `planSequencerAction`.

Next, let's take a look at the following part of the code:

	select {
	case <-sequencerCh:
		payload, err := s.sequencer.RunNextSequencerAction(ctx)
		if err != nil {
			s.log.Error("Sequencer critical error", "err", err)
			return
		}
		if s.network != nil && payload != nil {
			// Publishing of unsafe data via p2p is optional.
			// Errors are not severe enough to change/halt sequencing but should be logged and metered.
			if err := s.network.PublishL2Payload(ctx, payload); err != nil {
				s.log.Warn("failed to publish newly created block", "id", payload.ID(), "err", err)
				s.metrics.RecordPublishingError()
			}
		}
		planSequencerAction() // schedule the next sequencer action to keep the sequencing looping

This part of the code is triggered when the timer, set previously, reaches the predetermined time and sends out a signal. It first attempts to execute the next sequencing action. If this action succeeds, it tries to disseminate the newly created payload through the network. Regardless of the outcome, it eventually invokes the `planSequencerAction` function to schedule the next sequencing action, thereby establishing a continuous loop to manage the sequencing actions.

Next, let us examine the contents of the triggered `RunNextSequencerAction` function.

	// RunNextSequencerAction starts new block building work, or seals existing work,
	// and is best timed by first awaiting the delay returned by PlanNextSequencerAction.
	// If a new block is successfully sealed, it will be returned for publishing, nil otherwise.
	//
	// Only critical errors are bubbled up, other errors are handled internally.
	// Internally starting or sealing of a block may fail with a derivation-like error:
	//   - If it is a critical error, the error is bubbled up to the caller.
	//   - If it is a reset error, the ResettableEngineControl used to build blocks is requested to reset, and a backoff applies.
	//     No attempt is made at completing the block building.
	//   - If it is a temporary error, a backoff is applied to reattempt building later.
	//   - If it is any other error, a backoff is applied and building is cancelled.
	//
	// Upon L1 reorgs that are deep enough to affect the L1 origin selection, a reset-error may occur,
	// to direct the engine to follow the new L1 chain before continuing to sequence blocks.
	// It is up to the EngineControl implementation to handle conflicting build jobs of the derivation
	// process (as verifier) and sequencing process.
	// Generally it is expected that the latest call interrupts any ongoing work,
	// and the derivation process does not interrupt in the happy case,
	// since it can consolidate previously sequenced blocks by comparing sequenced inputs with derived inputs.
	// If the derivation pipeline does force a conflicting block, then an ongoing sequencer task might still finish,
	// but the derivation can continue to reset until the chain is correct.
	// If the engine is currently building safe blocks, then that building is not interrupted, and sequencing is delayed.
	func (d *Sequencer) RunNextSequencerAction(ctx context.Context) (*eth.ExecutionPayload, error) {
		if onto, buildingID, safe := d.engine.BuildingPayload(); buildingID != (eth.PayloadID{}) {
			if safe {
				d.log.Warn("avoiding sequencing to not interrupt safe-head changes", "onto", onto, "onto_time", onto.Time)
				// approximates the worst-case time it takes to build a block, to reattempt sequencing after.
				d.nextAction = d.timeNow().Add(time.Second * time.Duration(d.config.BlockTime))
				return nil, nil
			}
			payload, err := d.CompleteBuildingBlock(ctx)
			if err != nil {
				if errors.Is(err, derive.ErrCritical) {
					return nil, err // bubble up critical errors.
				} else if errors.Is(err, derive.ErrReset) {
					d.log.Error("sequencer failed to seal new block, requiring derivation reset", "err", err)
					d.metrics.RecordSequencerReset()
					d.nextAction = d.timeNow().Add(time.Second * time.Duration(d.config.BlockTime)) // hold off from sequencing for a full block
					d.CancelBuildingBlock(ctx)
					d.engine.Reset()
				} else if errors.Is(err, derive.ErrTemporary) {
					d.log.Error("sequencer failed temporarily to seal new block", "err", err)
					d.nextAction = d.timeNow().Add(time.Second)
					// We don't explicitly cancel block building jobs upon temporary errors: we may still finish the block.
					// Any unfinished block building work eventually times out, and will be cleaned up that way.
				} else {
					d.log.Error("sequencer failed to seal block with unclassified error", "err", err)
					d.nextAction = d.timeNow().Add(time.Second)
					d.CancelBuildingBlock(ctx)
				}
				return nil, nil
			} else {
				d.log.Info("sequencer successfully built a new block", "block", payload.ID(), "time", uint64(payload.Timestamp), "txs", len(payload.Transactions))
				return payload, nil
			}
		} else {
			err := d.StartBuildingBlock(ctx)
			if err != nil {
				if errors.Is(err, derive.ErrCritical) {
					return nil, err
				} else if errors.Is(err, derive.ErrReset) {
					d.log.Error("sequencer failed to seal new block, requiring derivation reset", "err", err)
					d.metrics.RecordSequencerReset()
					d.nextAction = d.timeNow().Add(time.Second * time.Duration(d.config.BlockTime)) // hold off from sequencing for a full block
					d.engine.Reset()
				} else if errors.Is(err, derive.ErrTemporary) {
					d.log.Error("sequencer temporarily failed to start building new block", "err", err)
					d.nextAction = d.timeNow().Add(time.Second)
				} else {
					d.log.Error("sequencer failed to start building new block with unclassified error", "err", err)
					d.nextAction = d.timeNow().Add(time.Second)
				}
			} else {
				parent, buildingID, _ := d.engine.BuildingPayload() // we should have a new payload ID now that we're building a block
				d.log.Info("sequencer started building new block", "payload_id", buildingID, "l2_parent_block", parent, "l2_parent_block_time", parent.Time)
			}
			return nil, nil
		}
	}

This code defines a method named `RunNextSequencerAction`, which is part of the Sequencer structure. The purpose of this method is to manage the block creation and encapsulation process, determining the next steps based on the current state and any encountered errors.

Here is the primary workflow and components of the method:

### Check the current block creation status:
- It uses `d.engine.BuildingPayload()` to check if there is a block currently being created.

### Handle the block that is being created:
- If there is a block in the process of being created, it checks whether it is safe to continue the creation. If so, it will retry later. If not, it attempts to complete the block creation.

### Error handling:
- Various errors can be encountered while trying to complete the block creation. These errors are categorized and handled appropriately:
  
  - **Critical errors:** These errors are passed on to the caller.
  - **Reset errors:** These cause the sequencer to reset, delaying subsequent sequencing attempts.
  - **Temporary errors:** These only cause a brief delay before trying again.
  - **Other errors:** These cancel the current block creation task and retry later.

### Successful block creation:
- If the block is successfully created, it logs a message and returns the newly created block along with a nil error.

### Start a new block creation task:
- If there are no blocks being created currently, it initiates a new block creation task. This involves a similar error-handling process as above.

### Log recordings:
- Throughout the method, various log messages are recorded based on different situations and outcomes to help track the sequencer's status and behavior.

Let's highlight the crucial steps, primarily divided into two parts: one is completing the full build, and the other is initiating a new block build.

First, let's look at the process of starting a new block build.


	func (d *Sequencer) StartBuildingBlock(ctx context.Context) error {
	…
	attrs, err := d.attrBuilder.PreparePayloadAttributes(fetchCtx, l2Head, l1Origin.ID())
	if err != nil {
		return err
	}

    …
	attrs.NoTxPool = uint64(attrs.Timestamp) > l1Origin.Time+d.config.MaxSequencerDrift

	…
	// Start a payload building process.
	errTyp, err := d.engine.StartPayload(ctx, l2Head, attrs, false)
	if err != nil {
		return fmt.Errorf("failed to start building on top of L2 chain %s, error (%d): %w", l2Head, errTyp, err)
	}
	…
}


In this segment of code, the `RunNextSequencerAction` method and its role in the block creation and encapsulation process are as follows:

## Method Details and Explanation

In this section, we will delve into the method of creating a new block and its components.

### Important Concepts in Optimism

Before we delve into `PreparePayloadAttributes`, we first need to understand two important concepts in the Optimism network: **Sequencing Window** and **Sequencing Epoch**.

#### Sequencing Window

In the merged Ethereum network, the fixed block time for L1 is 12 seconds, while for L2, it is 2 seconds. Based on this setting, we can define the concept of a “Sequencing Window” and elucidate it through an example:

- **Example**: If we set a “Sequencing Window” to encompass 5 L1 blocks, then the total time for this window would be 60 seconds (12 seconds/block × 5 blocks = 60 seconds). Within this 60-second timeframe, theoretically, 30 L2 blocks can be produced (60 seconds/2 seconds = 30).

#### Sequencing Epoch

A “Sequencing Epoch” is a series of L2 blocks derived based on a specific “Sequencing Window”.

- **Example**: In a “Sequencing Window” that encompasses 5 L1 blocks, it means it covers a 60-second timeframe (12 seconds/block × 5 blocks). Within this period, theoretically, 30 L2 blocks can be generated (60 seconds ÷ 2 seconds/block = 30 blocks).

### Adapting to Network Changes

In certain circumstances, to maintain the vitality of the network, we can cope with the situation of skipped L1 slots or temporary loss of connection with L1 by increasing the length of the “epoch.” Conversely, to prevent the L2 timestamps from gradually getting ahead of L1, we might need to shorten the “epoch” time for adjustments.

Through such a design, the system can flexibly and efficiently adjust the block generation strategy, ensuring the stability and security of the network.

### Function Detailed Explanation

In the function below, we can see that the passed-in epoch parameter is `l1Origin.ID()`. This aligns with our definition of epoch numbering. The function is responsible for preparing all the necessary attributes for creating a new L2 block.


```
	attrs, err := d.attrBuilder.PreparePayloadAttributes(fetchCtx, l2Head, l1Origin.ID())
```

	func (ba *FetchingAttributesBuilder) PreparePayloadAttributes(ctx context.Context, l2Parent eth.L2BlockRef, epoch eth.BlockID) (attrs *eth.PayloadAttributes, err error) {
		var l1Info eth.BlockInfo
		var depositTxs []hexutil.Bytes
		var seqNumber uint64

	sysConfig, err := ba.l2.SystemConfigByL2Hash(ctx, l2Parent.Hash)
	if err != nil {
		return nil, NewTemporaryError(fmt.Errorf("failed to retrieve L2 parent block: %w", err))
	}

	// If the L1 origin changed this block, then we are in the first block of the epoch. In this
	// case we need to fetch all transaction receipts from the L1 origin block so we can scan for
	// user deposits.
	if l2Parent.L1Origin.Number != epoch.Number {
		info, receipts, err := ba.l1.FetchReceipts(ctx, epoch.Hash)
		if err != nil {
			return nil, NewTemporaryError(fmt.Errorf("failed to fetch L1 block info and receipts: %w", err))
		}
		if l2Parent.L1Origin.Hash != info.ParentHash() {
			return nil, NewResetError(
				fmt.Errorf("cannot create new block with L1 origin %s (parent %s) on top of L1 origin %s",
					epoch, info.ParentHash(), l2Parent.L1Origin))
		}

		deposits, err := DeriveDeposits(receipts, ba.cfg.DepositContractAddress)
		if err != nil {
			// deposits may never be ignored. Failing to process them is a critical error.
			return nil, NewCriticalError(fmt.Errorf("failed to derive some deposits: %w", err))
		}
		// apply sysCfg changes
		if err := UpdateSystemConfigWithL1Receipts(&sysConfig, receipts, ba.cfg); err != nil {
			return nil, NewCriticalError(fmt.Errorf("failed to apply derived L1 sysCfg updates: %w", err))
		}

		l1Info = info
		depositTxs = deposits
		seqNumber = 0
	} else {
		if l2Parent.L1Origin.Hash != epoch.Hash {
			return nil, NewResetError(fmt.Errorf("cannot create new block with L1 origin %s in conflict with L1 origin %s", epoch, l2Parent.L1Origin))
		}
		info, err := ba.l1.InfoByHash(ctx, epoch.Hash)
		if err != nil {
			return nil, NewTemporaryError(fmt.Errorf("failed to fetch L1 block info: %w", err))
		}
		l1Info = info
		depositTxs = nil
		seqNumber = l2Parent.SequenceNumber + 1
	}

	// Sanity check the L1 origin was correctly selected to maintain the time invariant between L1 and L2
	nextL2Time := l2Parent.Time + ba.cfg.BlockTime
	if nextL2Time < l1Info.Time() {
		return nil, NewResetError(fmt.Errorf("cannot build L2 block on top %s for time %d before L1 origin %s at time %d",
			l2Parent, nextL2Time, eth.ToBlockID(l1Info), l1Info.Time()))
	}

	l1InfoTx, err := L1InfoDepositBytes(seqNumber, l1Info, sysConfig, ba.cfg.IsRegolith(nextL2Time))
	if err != nil {
		return nil, NewCriticalError(fmt.Errorf("failed to create l1InfoTx: %w", err))
	}

	txs := make([]hexutil.Bytes, 0, 1+len(depositTxs))
	txs = append(txs, l1InfoTx)
	txs = append(txs, depositTxs...)

	return &eth.PayloadAttributes{
		Timestamp:             hexutil.Uint64(nextL2Time),
		PrevRandao:            eth.Bytes32(l1Info.MixDigest()),
		SuggestedFeeRecipient: predeploys.SequencerFeeVaultAddr,
		Transactions:          txs,
		NoTxPool:              true,
		GasLimit:              (*eth.Uint64Quantity)(&sysConfig.GasLimit),
	}, nil
}


As illustrated in the code, the `PreparePayloadAttributes` is tasked with preparing the payload attributes for the new block. Initially, it determines whether there is a need to fetch new L1 deposits and system configuration data based on the parent block information of L1 and L2. Following this, it crafts a special system transaction encompassing details pertinent to the L1 block and system configurations. This distinct transaction, along with potential L1 deposit transactions, constitute a set of transactions to be incorporated into the payload of the new L2 block. The function ensures the coherence of time and the correct allocation of serial numbers, ultimately returning a `PayloadAttributes` structure containing all this information for the creation of a new L2 block. However, at this point, a preliminary payload is prepared, incorporating only the deposit transactions from L1. Subsequently, `StartPayload` is invoked to commence the next phase of payload construction.

Having acquired the attributes, we proceed further down.


	attrs.NoTxPool = uint64(attrs.Timestamp) > l1Origin.Time+d.config.MaxSequencerDrift

Determining whether it is necessary to produce an empty block, note that even here, the empty block will at least include L1 information deposits and any user deposits. If it is required to produce an empty block, we handle it by setting NoTxPool to true, which will result in the sequencer excluding any transactions from the transaction pool.

Next, the `StartPayload` is called to initiate the building of this payload.


	errTyp, err := d.engine.StartPayload(ctx, l2Head, attrs, false)
	if err != nil {
		return fmt.Errorf("failed to start building on top of L2 chain %s, error (%d): %w", l2Head, errTyp, err)
	}

#### StartPayload Function

The `StartPayload` mainly triggers a ForkchoiceUpdate and updates some of the building states in the EngineQueue, such as the buildingID, etc. Subsequently, when `RunNextSequencerAction` is run again, it will find the ID that is being built based on this ID.

```
	func (eq *EngineQueue) StartPayload(ctx context.Context, parent eth.L2BlockRef, attrs *eth.PayloadAttributes, updateSafe bool) (errType BlockInsertionErrType, err error) {
		if eq.isEngineSyncing() {
			return BlockInsertTemporaryErr, fmt.Errorf("engine is in progess of p2p sync")
		}
		if eq.buildingID != (eth.PayloadID{}) {
			eq.log.Warn("did not finish previous block building, starting new building now", "prev_onto", eq.buildingOnto, "prev_payload_id", eq.buildingID, "new_onto", parent)
			// TODO: maybe worth it to force-cancel the old payload ID here.
		}
		fc := eth.ForkchoiceState{
			HeadBlockHash:      parent.Hash,
			SafeBlockHash:      eq.safeHead.Hash,
			FinalizedBlockHash: eq.finalized.Hash,
		}
		id, errTyp, err := StartPayload(ctx, eq.engine, fc, attrs)
		if err != nil {
			return errTyp, err
		}
		eq.buildingID = id
		eq.buildingSafe = updateSafe
		eq.buildingOnto = parent
		return BlockInsertOK, nil
	}
```
```
   func StartPayload(ctx context.Context, eng Engine, fc eth.ForkchoiceState, attrs *eth.PayloadAttributes) (id eth.PayloadID, errType BlockInsertionErrType, err error) {
	…
	fcRes, err := eng.ForkchoiceUpdate(ctx, &fc, attrs)
	…
   }
```

In this function, the ForkchoiceUpdate is called internally, where we can see that a new Payload ID is created, and the ID being built along with other related parameters are updated.

### ForkchoiceUpdate Function

Following that, the `ForkchoiceUpdate` function is called to handle the updates to the ForkChoice. This function is a wrapper function that calls `engine_forkchoiceUpdatedV1` to trigger the EL to generate a new block.

The `ForkchoiceUpdate` function is a wrapper method for the call, which internally handles the call to the engine layer (op-geth) through the engine API. Here, `engine_forkchoiceUpdatedV1` is called to have the EL produce a block.


	var result eth.ForkchoiceUpdatedResult
	err := s.client.CallContext(fcCtx, &result, "engine_forkchoiceUpdatedV1", fc, attributes)


This function internally calls the `engine_forkchoiceUpdatedV1` method to handle the updates to the Fork Choice and the creation of new Payloads.

### The ForkchoiceUpdated Function in op-geth

Next, we turn our focus to the implementation of the `forkchoiceUpdated` function in op-geth.

In op-geth, the function responsible for handling this request is the `forkchoiceUpdated` function. This function first obtains and verifies various blocks related to the provided fork choice state. Then, it creates a new payload (i.e., a new block) based on this information and optional payload attributes. If the payload creation succeeds, it will return a valid response containing the new payload ID; otherwise, it will return an error. 

Here is the key code:

	if payloadAttributes != nil {
		if api.eth.BlockChain().Config().Optimism != nil && payloadAttributes.GasLimit == nil {
			return engine.STATUS_INVALID, engine.InvalidPayloadAttributes.With(errors.New("gasLimit parameter is required"))
		}
		transactions := make(types.Transactions, 0, len(payloadAttributes.Transactions))
		for i, otx := range payloadAttributes.Transactions {
			var tx types.Transaction
			if err := tx.UnmarshalBinary(otx); err != nil {
				return engine.STATUS_INVALID, fmt.Errorf("transaction %d is not valid: %v", i, err)
			}
			transactions = append(transactions, &tx)
		}
		args := &miner.BuildPayloadArgs{
			Parent:       update.HeadBlockHash,
			Timestamp:    payloadAttributes.Timestamp,
			FeeRecipient: payloadAttributes.SuggestedFeeRecipient,
			Random:       payloadAttributes.Random,
			Withdrawals:  payloadAttributes.Withdrawals,
			NoTxPool:     payloadAttributes.NoTxPool,
			Transactions: transactions,
			GasLimit:     payloadAttributes.GasLimit,
		}
		id := args.Id()
		// If we already are busy generating this work, then we do not need
		// to start a second process.
		if api.localBlocks.has(id) {
			return valid(&id), nil
		}
		payload, err := api.eth.Miner().BuildPayload(args)
		if err != nil {
			log.Error("Failed to build payload", "err", err)
			return valid(nil), engine.InvalidPayloadAttributes.With(err)
		}
		api.localBlocks.put(id, payload)
		return valid(&id), nil
	}

Here, we first load the payload that we created earlier in op-node into the 'args' variable, and then pass 'args' into the 'BuildPayload' function.

	// buildPayload builds the payload according to the provided parameters.
	func (w *worker) buildPayload(args *BuildPayloadArgs) (*Payload, error) {
		// Build the initial version with no transaction included. It should be fast
		// enough to run. The empty payload can at least make sure there is something
		// to deliver for not missing slot.
		empty, _, err := w.getSealingBlock(args.Parent, args.Timestamp, args.FeeRecipient, args.Random, args.Withdrawals, true, args.Transactions, args.GasLimit)
		if err != nil {
			return nil, err
		}
		// Construct a payload object for return.
		payload := newPayload(empty, args.Id())
		if args.NoTxPool { // don't start the background payload updating job if there is no tx pool to pull from
			return payload, nil
		}

		// Spin up a routine for updating the payload in background. This strategy
		// can maximum the revenue for including transactions with highest fee.
		go func() {
			// Setup the timer for re-building the payload. The initial clock is kept
			// for triggering process immediately.
			timer := time.NewTimer(0)
			defer timer.Stop()

			// Setup the timer for terminating the process if SECONDS_PER_SLOT (12s in
			// the Mainnet configuration) have passed since the point in time identified
			// by the timestamp parameter.
			endTimer := time.NewTimer(time.Second * 12)

			for {
				select {
				case <-timer.C:
					start := time.Now()
					block, fees, err := w.getSealingBlock(args.Parent, args.Timestamp, args.FeeRecipient, args.Random, args.Withdrawals, false, args.Transactions, args.GasLimit)
					if err == nil {
						payload.update(block, fees, time.Since(start))
					}
					timer.Reset(w.recommit)
				case <-payload.stop:
					log.Info("Stopping work on payload", "id", payload.id, "reason", "delivery")
					return
				case <-endTimer.C:
					log.Info("Stopping work on payload", "id", payload.id, "reason", "timeout")
					return
				}
			}
		}()
		return payload, nil
	}

### Initialization Phase:

First, it quickly builds an initial version of the empty payload (that is, a block that does not contain any transactions) using the provided parameters (but not including any transactions). If an error is encountered at this step, it will return that error.

#### Building the Return Payload Object:

Next, it uses the just-created empty block to create a payload object that will be returned to the caller. If the `args.NoTxPool` parameter is true, it means there is no transaction pool to fetch transactions from, the function will end and return the current payload object.

#### Background Payload Updating:

If `args.NoTxPool` is false, then a background goroutine is launched to periodically update the payload to include more transactions and update the state. This background process has two timers:
- One to control when to recreate the payload to include new transactions.
- Another to set a timeout, after which the background process will stop working.

Each time the timer triggers, it tries to obtain a new block to update the payload; if successful, it updates the data in the payload object. The background process stops if the payload is delivered or the time is out.

In this way, the `buildPayload` function ensures a quickly available initial payload while the background process optimizes the payload as much as possible by including more transactions. This strategy aims to maximize the income obtained by including high-fee transactions.

So how does it work when `args.NoTxPool` is false?
The answer is hidden in the `getSealingBlock` function.

	func (w *worker) getSealingBlock(parent common.Hash, timestamp uint64, coinbase common.Address, random common.Hash, withdrawals types.Withdrawals, noTxs bool, transactions types.Transactions, gasLimit *uint64) (*types.Block, *big.Int, error) {
		req := &getWorkReq{
			params: &generateParams{
				timestamp:   timestamp,
				forceTime:   true,
				parentHash:  parent,
				coinbase:    coinbase,
				random:      random,
				withdrawals: withdrawals,
				noUncle:     true,
				noTxs:       noTxs,
				txs:         transactions,
				gasLimit:    gasLimit,
			},
			result: make(chan *newPayloadResult, 1),
		}
		select {
		case w.getWorkCh <- req:
			result := <-req.result
			if result.err != nil {
				return nil, nil, result.err
			}
			return result.block, result.fees, nil
		case <-w.exitCh:
			return nil, nil, errors.New("miner closed")
		}
	}


In this section, we see that the `mainLoop` function receives new Payload creation requests by listening to the `getWorkCh` channel. Once a request is received, it triggers the `generateWork` function to initiate the new Payload creation process.

	case req := <-w.getWorkCh:
		block, fees, err := w.generateWork(req.params)
		req.result <- &newPayloadResult{
			err:   err,
			block: block,
			fees:  fees,
		}

### GenerateWork Function

The `GenerateWork` function is the final step in the new Payload creation process. It is responsible for preparing the ground and creating the new block.

```go


	// generateWork generates a sealing block based on the given parameters.
	func (w *worker) generateWork(genParams *generateParams) (*types.Block, *big.Int, error) {
		work, err := w.prepareWork(genParams)
		if err != nil {
			return nil, nil, err
		}
		defer work.discard()
		if work.gasPool == nil {
			work.gasPool = new(core.GasPool).AddGas(work.header.GasLimit)
		}

		for _, tx := range genParams.txs {
			from, _ := types.Sender(work.signer, tx)
			work.state.SetTxContext(tx.Hash(), work.tcount)
			_, err := w.commitTransaction(work, tx)
			if err != nil {
				return nil, nil, fmt.Errorf("failed to force-include tx: %s type: %d sender: %s nonce: %d, err: %w", tx.Hash(), tx.Type(), from, tx.Nonce(), err)
			}
			work.tcount++
		}

		// forced transactions done, fill rest of block with transactions
		if !genParams.noTxs {
			interrupt := new(atomic.Int32)
			timer := time.AfterFunc(w.newpayloadTimeout, func() {
				interrupt.Store(commitInterruptTimeout)
			})
			defer timer.Stop()

			err := w.fillTransactions(interrupt, work)
			if errors.Is(err, errBlockInterruptedByTimeout) {
				log.Warn("Block building is interrupted", "allowance", common.PrettyDuration(w.newpayloadTimeout))
			}
		}
		block, err := w.engine.FinalizeAndAssemble(w.chain, work.header, work.state, work.txs, work.unclelist(), work.receipts, genParams.withdrawals)
		if err != nil {
			return nil, nil, err
		}
		return block, totalFees(block, work.receipts), nil
	}
```

In this function, we can see the detailed block creation process, including the handling of transactions and the final assembly of the block.

## Block Completion and Confirmation Process

In this section, we will continue to explore the block production process under the Sequencer mode. This stage mainly involves the completion and confirmation process of the block. Below, we will analyze each step and the function's role in detail.

### Block Construction and Optimization

At the beginning stage, we first build a new block in the memory pool. Here, special attention is paid to the application of the `NoTxPool` parameter, which was set earlier in the Sequencer. This part of the code is responsible for the preliminary construction of the block and the subsequent optimization work.

The key step in this is	

	if !genParams.noTxs {
		interrupt := new(atomic.Int32)
		timer := time.AfterFunc(w.newpayloadTimeout, func() {
			interrupt.Store(commitInterruptTimeout)
		})
		defer timer.Stop()

		err := w.fillTransactions(interrupt, work)
		if errors.Is(err, errBlockInterruptedByTimeout) {
			log.Warn("Block building is interrupted", "allowance", common.PrettyDuration(w.newpayloadTimeout))
		}
	}

In this part, we finally use the `NoTxPool` parameter that was set earlier in the sequencer, and then start building a new block in the memory pool (here the transactions in the memory pool come from itself and other nodes. However, since gossip is turned off by default, there is no transaction communication between other nodes, which is why the sequencer's memory pool is private).

Up to this point, the block has been generated in the sequencer node. However, the `buildPayload` function creates an initial, structurally correct block and continuously optimizes it in a background process to increase its transaction content and potential miner revenue. Yet its validity and endorsement are dependent on the subsequent network consensus process. That is, it still needs subsequent steps to stop this update and determine a final block.

Next, let's return to the sequencer.
At the current stage, we have confirmed that the `buildingID` in the `EngineQueue` has been set, and the block derived from this payload has been generated in `op-geth`.
Next, due to the timer set at the very beginning, the sequencer triggers and calls the `RunNextSequencerAction` method again. 
It enters the judgment, but this time, our `buildingID` already exists, so it enters the `CompleteBuildingBlock` stage.


	if onto, buildingID, safe := d.engine.BuildingPayload(); buildingID != (eth.PayloadID{}) {
			…
			payload, err := d.CompleteBuildingBlock(ctx)
			…
		}

CompleteBuildingBlock calls the ConfirmPayload method internally

	// ConfirmPayload ends an execution payload building process in the provided Engine, and persists the payload as the canonical head.
	// If updateSafe is true, then the payload will also be recognized as safe-head at the same time.
	// The severity of the error is distinguished to determine whether the payload was valid and can become canonical.
	func ConfirmPayload(ctx context.Context, log log.Logger, eng Engine, fc eth.ForkchoiceState, id eth.PayloadID, updateSafe bool) (out *eth.ExecutionPayload, errTyp BlockInsertionErrType, err error) {
		payload, err := eng.GetPayload(ctx, id)
		…
		…
		status, err := eng.NewPayload(ctx, payload)
		…
		…
		fcRes, err := eng.ForkchoiceUpdate(ctx, &fc, nil)
		…
		return payload, BlockInsertOK, nil
	}

Here you can refer to the illustration provided by oplabs. 

![ENGINE](../resources/engine.svg)

This flowchart mainly describes the process of creating blocks.

#### The Rollup driver actually does not really create blocks. Instead, it guides the execution engine to do so through the Engine API. In each iteration of the above block derivation loop, the rollup driver will make a payload attribute object and send it to the execution engine. The execution engine then converts the payload attribute object into a block and adds it to the chain. The basic sequence of the Rollup driver is as follows:

1. Call `engine_forkChoiceUpdatedV1` using the payload attribute object. We will skip the details of the fork choice state parameter for now - just know that one of its fields is `headBlockHash`, which is set to the block hash at the tip of the L2 chain. The Engine API returns a payload ID.
2. Call `engine_getPayloadV1` using the payload ID returned in step 1. The Engine API returns a payload object containing the block hash as one of its fields.
3. Call `engine_newPayloadV1` using the payload returned in step 2.
4. Call `engine_forkChoiceUpdatedV1` with the `headBlockHash` set to the block hash returned in step 2. Now, the tip of the L2 chain is the block created in step 1.

The `engine_forkChoiceUpdatedV1` in the first step is the process we started building from the beginning, while steps two, three, and four are in the `ConfirmPayload` method.

The second step, `GetPayload` method, retrieves the `ExecutionPayload` of the block we built in the first step.

	// Resolve returns the latest built payload and also terminates the background
	// thread for updating payload. It's safe to be called multiple times.
	func (payload *Payload) Resolve() *engine.ExecutionPayloadEnvelope {
		payload.lock.Lock()
		defer payload.lock.Unlock()

		select {
		case <-payload.stop:
		default:
			close(payload.stop)
		}
		if payload.full != nil {
			return engine.BlockToExecutableData(payload.full, payload.fullFees)
		}
		return engine.BlockToExecutableData(payload.empty, big.NewInt(0))
	}

The `GetPayload` method stops the block's reconstruction by sending a signal to the `payload.stop` channel in the coroutine initiated in the first step. Meanwhile, it sends the latest block data (ExecutionPayload) back to the sequencer (op-node).

The third step, `NewPayload` method,


	func (api *ConsensusAPI) newPayload(params engine.ExecutableData) (engine.PayloadStatusV1, error) {
		…
		block, err := engine.ExecutableDataToBlock(params)
		if err != nil {
			log.Debug("Invalid NewPayload params", "params", params, "error", err)
			return engine.PayloadStatusV1{Status: engine.INVALID}, nil
		}
		…
		if err := api.eth.BlockChain().InsertBlockWithoutSetHead(block); err != nil {
			log.Warn("NewPayloadV1: inserting block failed", "error", err)

			api.invalidLock.Lock()
			api.invalidBlocksHits[block.Hash()] = 1
			api.invalidTipsets[block.Hash()] = block.Header()
			api.invalidLock.Unlock()

			return api.invalid(err, parent.Header()), nil
		}
		…
	}

At this point, based on the payload parameters that have been finally confirmed, a block is constructed. Following this, the block is inserted into our blockchain.

The fourth step, `ForkchoiceUpdate` method,


	func (api *ConsensusAPI) forkchoiceUpdated(update engine.ForkchoiceStateV1, payloadAttributes *engine.PayloadAttributes) (engine.ForkChoiceResponse, error) {
		…
		…
		if update.FinalizedBlockHash != (common.Hash{}) {
			if merger := api.eth.Merger(); !merger.PoSFinalized() {
				merger.FinalizePoS()
			}
			// If the finalized block is not in our canonical tree, somethings wrong
			finalBlock := api.eth.BlockChain().GetBlockByHash(update.FinalizedBlockHash)
			if finalBlock == nil {
				log.Warn("Final block not available in database", "hash", update.FinalizedBlockHash)
				return engine.STATUS_INVALID, engine.InvalidForkChoiceState.With(errors.New("final block not available in database"))
			} else if rawdb.ReadCanonicalHash(api.eth.ChainDb(), finalBlock.NumberU64()) != update.FinalizedBlockHash {
				log.Warn("Final block not in canonical chain", "number", block.NumberU64(), "hash", update.HeadBlockHash)
				return engine.STATUS_INVALID, engine.InvalidForkChoiceState.With(errors.New("final block not in canonical chain"))
			}
			// Set the finalized block
			api.eth.BlockChain().SetFinalized(finalBlock.Header())
		}
		…
	}

Through the `SetFinalized` process, the block generated from the previous steps is finalized. This method marks a specific block as "finalized." In blockchain networks, when a block is labeled as "finalized," it signifies that the block and all preceding blocks are irreversible, forever becoming a part of the blockchain. This is a vital security feature ensuring that once a block is finalized, it cannot be replaced by another fork.

With this, the basic construction of an L2 block is completed. The subsequent task is to record this new L2 information in the sequencer. Let's return to the `ConfirmPayload` function to continue.

		payload, errTyp, err := ConfirmPayload(ctx, eq.log, eq.engine, fc, eq.buildingID, eq.buildingSafe)
		if err != nil {
			return nil, errTyp, fmt.Errorf("failed to complete building on top of L2 chain %s, id: %s, error (%d): %w", eq.buildingOnto, eq.buildingID, errTyp, err)
		}
		ref, err := PayloadToBlockRef(payload, &eq.cfg.Genesis)
		if err != nil {
			return nil, BlockInsertPayloadErr, NewResetError(fmt.Errorf("failed to decode L2 block ref from payload: %w", err))
		}

		eq.unsafeHead = ref
		eq.engineSyncTarget = ref
		eq.metrics.RecordL2Ref("l2_unsafe", ref)
		eq.metrics.RecordL2Ref("l2_engineSyncTarget", ref)


We can observe that the payload is parsed into `PayloadToBlockRef` (PayloadToBlockRef extracts the essential L2BlockRef information from an execution payload, falling back to genesis information if necessary.), such as `unsafeHead`. These data will be used in subsequent steps, such as block propagation.

With this, the complete process of block generation under the sequencer mode comes to an end. In the next chapter, we will continue to discuss how the sequencer propagates this block to other nodes, such as verifiers, after the block has been generated.