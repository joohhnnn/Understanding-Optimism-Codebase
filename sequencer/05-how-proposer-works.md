# Introduction to op-proposer

In this chapter, we will explore what exactly `op-proposer` is 🌊.

Firstly, let's share some resources from the official specs ([source](https://github.com/ethereum-optimism/optimism/blob/develop/specs/proposals.md)).

In a nutshell, the role of `proposer` is to periodically send the state root from layer 2 to layer 1. This allows for trustless execution of transactions originating from layer 2 directly on layer 1, such as withdrawals or message communications.

we will use a layer 1 withdrawal transaction originating from layer 2 as an example to discuss the role of `op-proposer` in the entire process.

## Withdrawal Process

In Optimism, a withdrawal is a transaction from L2 (e.g., OP Mainnet, OP Goerli) to L1 (e.g., Ethereum mainnet, Goerli), possibly with or without assets. It can roughly be divided into four transactions:
- The withdrawal initiation transaction submitted by the user on L2;
- The `proposer` uploads the `state root` from L2 to L1 via a transaction, for use in subsequent steps by the user in L1;
- The withdrawal proof transaction submitted by the user in L1, based on the `Merkle Patricia Trie`, proving the legitimacy of the withdrawal;
- After the error challenge period has passed, the final withdrawal transaction is executed on L1 by the user, claiming any additional assets, etc.

For more details, you can refer to the official description of this part ([source](https://community.optimism.io/docs/protocol/withdrawal-flow/#)).

## What is proposer
The `proposer` serves as a connector when parts of L2 data are needed in L1. It sends this portion of L2 data (state root) to a contract in L1. The contract in L1 can then directly utilize this data via contract calls.

⚠️ Note: Many people think that only after the `proposer` sends the `state root` does it mean these blocks are `finalized`. This understanding is `incorrect`. `Safe` blocks in L1 are considered `finalized` after **two** `epochs (64 blocks)`. The `proposer` uploads `finalized` block data, it does not become `finalized` after upload.

### Difference between proposer and batcher
Previously, we discussed the `batcher` part, which also transfers L2 data to L1. You might wonder, if the `batcher` has already moved the data to L1, why do we need another `proposer` to move it again?

#### Inconsistent Block State
When the `batcher` sends data, the block state is still in an `unsafe` state, it cannot be directly used, and you can't determine when the block becomes a `finalized` state based on `batcher` transactions.
When the `proposer` sends data, it signifies that the relevant blocks have reached the `finalized` stage, and the data can be highly trusted and used.

#### Different Data Format and Size
The `batcher` stores almost complete transaction details, including `gas price, data` etc., in `layer 1`.
The `proposer` only sends the block's `state root` to L1. This `state root` can later be used in conjunction with [merkle-tree](https://medium.com/techskill-brew/merkle-tree-in-blockchain-part-5-blockchain-basics-4e25b61179a2) design.
The `batcher` transfers massive data, while the `proposer` transfers only a small amount. Therefore, `batcher` data is more suitable for being placed in `calldata`, which is cheap but can't be directly used in contracts. The `proposer` data is stored in `contract storage`, where the data amount is small, the cost is not high, and it can be used in contract interactions.

#### Differences Between `calldata` and `storage` in Ethereum
In Ethereum, the main differences between `calldata` and `storage` can be categorized into three aspects:

1. **Persistence**:
   - `storage`: Persistent storage; data is saved permanently.
   - `calldata`: Temporary storage; data disappears after the function execution.

2. **Cost**:
   - `storage`: Expensive due to permanent data storage.
   - `calldata`: Cheaper, as it's only for temporary storage.

3. **Accessibility**:
   - `storage`: Accessible across multiple functions or transactions.
   - `calldata`: Only accessible during the current function execution.

## Code Implementation
In this section, we will delve into the mechanisms and implementation principles from a code perspective.

### Entry Point of the Program
`op-proposer/proposer/l2_output_submitter.go`

The `Start` function is called to initiate the `loop`. Within the loop, the function `FetchNextOutputInfo` is primarily responsible for checking whether the next block should send a `proposal` transaction. If it should, the `sendTransaction` function is directly called to send it to L1. Otherwise, the loop continues to the next iteration.

> **Source Code**: [op-proposer/proposer/l2_output_submitter.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-proposer/proposer/l2_output_submitter.go#L381)

```go
    func (l *L2OutputSubmitter) loop() {
        defer l.wg.Done()

        ctx := l.ctx

        ticker := time.NewTicker(l.pollInterval)
        defer ticker.Stop()
        for {
            select {
            case <-ticker.C:
                output, shouldPropose, err := l.FetchNextOutputInfo(ctx)
                if err != nil {
                    break
                }
                if !shouldPropose {
                    break
                }
                cCtx, cancel := context.WithTimeout(ctx, 10*time.Minute)
                if err := l.sendTransaction(cCtx, output); err != nil {
                    l.log.Error("Failed to send proposal transaction",
                        "err", err,
                        "l1blocknum", output.Status.CurrentL1.Number,
                        "l1blockhash", output.Status.CurrentL1.Hash,
                        "l1head", output.Status.HeadL1.Number)
                    cancel()
                    break
                }
                l.metr.RecordL2BlocksProposed(output.BlockRef)
                cancel()

            case <-l.done:
                return
            }
        }
    }
```

### Fetching Output
`op-proposer/proposer/l2_output_submitter.go`

The `FetchNextOutputInfo` function retrieves the next block number for sending a `proposal` by calling the `l2ooContract` contract. It then compares this block number with the current L2 block number to determine whether a `proposal` transaction should be sent. If it should be sent, the `fetchOutput` function is called to generate the `output`.

> **Source Code**: [op-proposer/proposer/l2_output_submitter.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-proposer/proposer/l2_output_submitter.go#L241)

```go
    func (l *L2OutputSubmitter) FetchNextOutputInfo(ctx context.Context) (*eth.OutputResponse, bool, error) {
        cCtx, cancel := context.WithTimeout(ctx, l.networkTimeout)
        defer cancel()
        callOpts := &bind.CallOpts{
            From:    l.txMgr.From(),
            Context: cCtx,
        }
        nextCheckpointBlock, err := l.l2ooContract.NextBlockNumber(callOpts)
        if err != nil {
            l.log.Error("proposer unable to get next block number", "err", err)
            return nil, false, err
        }
        // Fetch the current L2 heads
        cCtx, cancel = context.WithTimeout(ctx, l.networkTimeout)
        defer cancel()
        status, err := l.rollupClient.SyncStatus(cCtx)
        if err != nil {
            l.log.Error("proposer unable to get sync status", "err", err)
            return nil, false, err
        }

        // Use either the finalized or safe head depending on the config. Finalized head is default & safer.
        var currentBlockNumber *big.Int
        if l.allowNonFinalized {
            currentBlockNumber = new(big.Int).SetUint64(status.SafeL2.Number)
        } else {
            currentBlockNumber = new(big.Int).SetUint64(status.FinalizedL2.Number)
        }
        // Ensure that we do not submit a block in the future
        if currentBlockNumber.Cmp(nextCheckpointBlock) < 0 {
            l.log.Debug("proposer submission interval has not elapsed", "currentBlockNumber", currentBlockNumber, "nextBlockNumber", nextCheckpointBlock)
            return nil, false, nil
        }

        return l.fetchOutput(ctx, nextCheckpointBlock)
    }
```

`fetchOutput` function internally uses `OutputV0AtBlock` to retrieve and process the `output` response body.

`op-node/sources/l2_client.go`

The `OutputV0AtBlock` function takes the previously identified block hash for sending a `proposal` to obtain the block header. It then derives the data needed for `OutputV0` based on this block header. The role of the `StorageHash (withdrawal_storage_root)` obtained through the `GetProof` function is to significantly reduce the size of the entire Merkle tree proof process if only `state` data related to `L2ToL1MessagePasserAddr` is needed.

> **Source Code**: [op-node/sources/l2_client.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-node/sources/l2_client.go#L170)

```go
    func (s *L2Client) OutputV0AtBlock(ctx context.Context, blockHash common.Hash) (*eth.OutputV0, error) {
        head, err := s.InfoByHash(ctx, blockHash)
        if err != nil {
            return nil, fmt.Errorf("failed to get L2 block by hash: %w", err)
        }
        if head == nil {
            return nil, ethereum.NotFound
        }

        proof, err := s.GetProof(ctx, predeploys.L2ToL1MessagePasserAddr, []common.Hash{}, blockHash.String())
        if err != nil {
            return nil, fmt.Errorf("failed to get contract proof at block %s: %w", blockHash, err)
        }
        if proof == nil {
            return nil, fmt.Errorf("proof %w", ethereum.NotFound)
        }
        // make sure that the proof (including storage hash) that we retrieved is correct by verifying it against the state-root
        if err := proof.Verify(head.Root()); err != nil {
            return nil, fmt.Errorf("invalid withdrawal root hash, state root was %s: %w", head.Root(), err)
        }
        stateRoot := head.Root()
        return &eth.OutputV0{
            StateRoot:                eth.Bytes32(stateRoot),
            MessagePasserStorageRoot: eth.Bytes32(proof.StorageHash),
            BlockHash:                blockHash,
        }, nil
    }
```

### Sending Output

`op-proposer/proposer/l2_output_submitter.go`

Within the `sendTransaction` function, the `proposeL2OutputTxData` function is indirectly called to use the `ABI of the L1 contract` to match our `output` with the `input format` of the contract function. The `sendTransaction` function then sends the packaged data to L1, interacting with the `L2OutputOracle contract`.

> **Source Code**: [op-proposer/proposer/l2_output_submitter.go (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/op-proposer/proposer/l2_output_submitter.go#L313)

```go
    func proposeL2OutputTxData(abi *abi.ABI, output *eth.OutputResponse) ([]byte, error) {
        return abi.Pack(
            "proposeL2Output",
            output.OutputRoot,
            new(big.Int).SetUint64(output.BlockRef.Number),
            output.Status.CurrentL1.Hash,
            new(big.Int).SetUint64(output.Status.CurrentL1.Number))
    }
```

`packages/contracts-bedrock/src/L1/L2OutputOracle.sol`

The `L2OutputOracle contract` validates this `state root from the L2 block` and stores it in the `contract's storage`.

> **Source Code**: [packages/contracts-bedrock/src/L1/L2OutputOracle.sol (v1.1.4)](https://github.com/ethereum-optimism/optimism/blob/v1.1.4/packages/contracts-bedrock/src/L1/L2OutputOracle.sol#L178)

```solidity
    /// @notice Accepts an outputRoot and the timestamp of the corresponding L2 block.
    ///         The timestamp must be equal to the current value returned by `nextTimestamp()` in
    ///         order to be accepted. This function may only be called by the Proposer.
    /// @param _outputRoot    The L2 output of the checkpoint block.
    /// @param _l2BlockNumber The L2 block number that resulted in _outputRoot.
    /// @param _l1BlockHash   A block hash which must be included in the current chain.
    /// @param _l1BlockNumber The block number with the specified block hash.
    function proposeL2Output(
        bytes32 _outputRoot,
        uint256 _l2BlockNumber,
        bytes32 _l1BlockHash,
        uint256 _l1BlockNumber
    )
        external
        payable
    {
        require(msg.sender == proposer, "L2OutputOracle: only the proposer address can propose new outputs");

        require(
            _l2BlockNumber == nextBlockNumber(),
            "L2OutputOracle: block number must be equal to next expected block number"
        );

        require(
            computeL2Timestamp(_l2BlockNumber) < block.timestamp,
            "L2OutputOracle: cannot propose L2 output in the future"
        );

        require(_outputRoot != bytes32(0), "L2OutputOracle: L2 output proposal cannot be the zero hash");

        if (_l1BlockHash != bytes32(0)) {
            // This check allows the proposer to propose an output based on a given L1 block,
            // without fear that it will be reorged out.
            // It will also revert if the blockheight provided is more than 256 blocks behind the
            // chain tip (as the hash will return as zero). This does open the door to a griefing
            // attack in which the proposer's submission is censored until the block is no longer
            // retrievable, if the proposer is experiencing this attack it can simply leave out the
            // blockhash value, and delay submission until it is confident that the L1 block is
            // finalized.
            require(
                blockhash(_l1BlockNumber) == _l1BlockHash,
                "L2OutputOracle: block hash does not match the hash at the expected height"
            );
        }

        emit OutputProposed(_outputRoot, nextOutputIndex(), _l2BlockNumber, block.timestamp);

        l2Outputs.push(
            Types.OutputProposal({
                outputRoot: _outputRoot,
                timestamp: uint128(block.timestamp),
                l2BlockNumber: uint128(_l2BlockNumber)
            })
        );
    }
```

## Summary
The overall implementation logic of the `proposer` is relatively straightforward. It involves periodically running a loop to fetch the next L2 block that needs to send a `proposal` from L1 and comparing it with the local L2 block, and is responsible for processing the data and sending it to L1. Most of the other transaction flows during the withdrawal process are handled by the `SDK`, and you can read more about this in our previously released official description of the withdrawal process ([source](https://community.optimism.io/docs/protocol/withdrawal-flow/#)).
To observe the actual behavior of the `proposer` on the mainnet, you can check this [proposer address](https://etherscan.io/address/0x473300df21d047806a082244b417f96b32f13a33).
