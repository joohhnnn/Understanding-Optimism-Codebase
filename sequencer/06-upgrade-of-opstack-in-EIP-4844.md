# Upgrade of OPStack in EIP-4844

Ethereum's EIP-4844 represents a significant transformation for layer 2, as it substantially reduces the costs for layer 2 when using L1 for data availability (DA). This article will not delve into the specifics of EIP-4844 but will provide a brief overview to set the context for our exploration of the updates within OP-Stack.

## EIP-4844

TL;DR:
EIP-4844 introduces a data format called "blob," which does not participate in EVM execution but is stored in the consensus layer. Each data blob has a lifespan of 4096 epochs (about 18 days). Blobs exist on the L1 mainnet, carried by a new type of transaction known as type3, with each block capable of accommodating up to six blobs, and each transaction able to carry six blobs.

For further reading, please refer to the following articles:
[Ethereum Evolved: Dencun Upgrade Part 5, EIP-4844](https://consensys.io/blog/ethereum-evolved-dencun-upgrade-part-5-eip-4844)
[EIP-4844, Blobs, and Blob Gas: What you need to know](https://www.blocknative.com/blog/eip-4844-blobs-and-blob-gas-what-you-need-to-know)
[Proto-Danksharding](https://notes.ethereum.org/@vbuterin/proto_danksharding_faq#Proto-Danksharding-FAQ)

## Application of OP-Stack

After adopting BLOBs to replace CALLDATA as the data storage method for rollups, the fees plummeted dramatically.
![image](https://hackmd.io/_uploads/BJCSEyngA.png)

In this update of OP-Stack, the main changes in logic involve converting data previously sent via calldata into blob format and sending it to L1 as blob-type transactions. Additionally, this update involves parsing blobs when retrieving data sent to rollups from L1. Here are the main components involved in this upgrade:

1. **submitter** — Responsible for sending rollup data to L1.
2. **fetcher** — Syncs data from L1 to L2, including old rollup data and deposit transactions.
3. **blob-related definitions and implementation** — How to retrieve and structure blob data, etc.
4. **other related design parts** — Such as client support for signing blob type transactions, designs related to fault proof, etc.

---

⚠️⚠️⚠️Please note that all code discussed in this article is based on initial PR designs and may differ from the code running in production environments.

---

### Blob-related Definitions and Encoding/Decoding Implementation

[Pull Request(8131) blob definition](https://github.com/ethereum-optimism/optimism/pull/8131/files#diff-30107b16d72d6e958093d83b5d736522a7994cab064187562605c82174400cd5)
[Pull Request(8767) encoding & decoding](https://github.com/ethereum-optimism/optimism/commit/78ecdf523026d0afa45c519524a15b83cbe162c8#diff-30107b16d72d6e958093d83b5d736522a7994cab064187562605c82174400cd5R86)

#### Defining blob

```
BlobSize = 4096 * 32
  
type Blob [BlobSize]byte
```

#### blob encoding

[**official specs about blob encoding**](https://github.com/ethereum-optimism/specs/blob/main/specs/protocol/derivation.md#blob-encoding)
It's important to note that these specs correspond to the latest version of the code, while the code snippets below are from an initial simplified version. The main difference is that the Blob type is divided into 4096 field elements, each constrained by the size of a "specific modulus," meaning math.log2(BLS_MODULUS) = 254.8570894..., which implies that each field element is no larger than 254 bits or 31.75 bytes. The initial demo code used only 31 bytes, thus not utilizing 0.75 bytes of space. In contrast, the latest code version employs four field elements working together to utilize the 0.75 bytes of space fully, thus increasing data efficiency.
Here is part of Pull Request(8767) with code snippets:
By iterating 4096 times, it reads a total of 31*4096 bytes of data, which are then added to the blob.

```
func (b *Blob) FromData(data Data) error {
	if len(data) > MaxBlobDataSize {
		return fmt.Errorf("data is too large for blob. len=%v", len(data))
	}
	b.Clear()
	// encode 4-byte little-endian length value into topmost 4 bytes (out of 31) of first field
	// element
	binary.LittleEndian.PutUint32(b[1:5], uint32(len(data)))
	// encode first 27 bytes of input data into remaining bytes of first field element
	offset := copy(b[5:32], data)
	// encode (up to) 31 bytes of remaining input data at a time into the subsequent field element
	for i := 1; i < 4096; i++ {
		offset += copy(b[i*32+1:i*32+32], data[offset:])
		if offset == len(data) {
			break
		}
	}
	if offset < len(data) {
		return fmt.Errorf("failed to fit all data into blob. bytes remaining: %v", len(data)-offset)
	}
	return nil
}
```

#### blob decoding

blob data decoding follows the same principles as the data encoding mentioned above.

```
func (b *Blob) ToData() (Data, error) {
	data := make(Data, 4096*32)
	for i := 0; i < 4096; i++ {
		if b[i*32] != 0 {
			return nil, fmt.Errorf("invalid blob, found non-zero high order byte %x of field element %d", b[i*32], i)
		}
		copy(data[i*31:i*31+31], b[i*32+1:i*32+32])
	}
	// extract the length prefix & trim the output accordingly
	dataLen := binary.LittleEndian.Uint32(data[:4])
	data = data[4:]
	if dataLen > uint32(len(data)) {
		return nil, fmt.Errorf("invalid blob, length prefix out of range: %d", dataLen)
	}
	data = data[:dataLen]
	return data, nil
}
```

### Submitter

[Pull Request(8769)](https://github.com/ethereum-optimism/optimism/pull/8769)

#### flag configuration

```
switch c.DataAvailabilityType {
case flags.CalldataType:
case flags.BlobsType:
default:
    return fmt.Errorf("unknown data availability type: %v", c.DataAvailabilityType)
}
```

#### BatchSubmitter

BatchSubmitter's functionality has expanded from just sending calldata to deciding whether to send calldata or blob-type data, depending on the situation. Blob-type data is encoded within blobTxCandidate using the previously mentioned FromData (blob-encode) function.

```
func (l *BatchSubmitter) sendTransaction(txdata txData, queue *txmgr.Queue[txData], receiptsCh chan txmgr.TxReceipt[txData]) error {
	// Do the gas estimation offline. A value of 0 will cause the [txmgr] to estimate the gas limit.
	data := txdata.Bytes()

	var candidate *txmgr.TxCandidate
	if l.Config.UseBlobs {
		var err error
		if candidate, err = l.blobTxCandidate(data); err != nil {
			// We could potentially fall through and try a calldata tx instead, but this would
			// likely result in the chain spending more in gas fees than it is tuned for, so best
			// to just fail. We do not expect this error to trigger unless there is a serious bug
			// or configuration issue.
			return fmt.Errorf("could not create blob tx candidate: %w", err)
		}
	} else {
		candidate = l.calldataTxCandidate(data)
	}

	intrinsicGas, err := core.IntrinsicGas(candidate.TxData, nil, false, true, true, false)
	if err != nil {
		// we log instead of return an error here because txmgr can do its own gas estimation
		l.Log.Error("Failed to calculate intrinsic gas", "err", err)
	} else {
		candidate.GasLimit = intrinsicGas
	}

	queue.Send(txdata, *candidate, receiptsCh)
	return nil
}

func (l *BatchSubmitter) blobTxCandidate(data []byte) (*txmgr.TxCandidate, error) {
	var b eth.Blob
	if err := b.FromData(data); err != nil {
		return nil, fmt.Errorf("data could not be converted to blob: %w", err)
	}
	return &txmgr.TxCandidate{
		To:    &l.RollupConfig.BatchInboxAddress,
		Blobs: []*eth.Blob{&b},
	}, nil
}
```

### Fetcher

[Pull Request(9098)](https://github.com/ethereum-optimism/optimism/pull/9098/files#diff-1fd8727490dbd2214b6d0c247eb222f2ac4098d259b4a45e9c0caea7fb2d3e08)

#### GetBlob

GetBlob is responsible for retrieving blob data, its main logic includes using 4096 field elements to construct a complete blob and verifying its correctness through commitment. Additionally, GetBlob also participates in the [upper layer L1Retrieval logic process.](https://github.com/joohhnnn/Understanding-Optimism-Codebase/blob/main/sequencer/04-how-derivation-works.md)

```
func (p *PreimageOracle) GetBlob(ref eth.L1BlockRef, blobHash eth.IndexedBlobHash) *eth.Blob {
	// Send a hint for the blob commitment & blob field elements.
	blobReqMeta := make([]byte, 16)
	binary.BigEndian.PutUint64(blobReqMeta[0:8], blobHash.Index)
	binary.BigEndian.PutUint64(blobReqMeta[8:16], ref.Time)
	p.hint.Hint(BlobHint(append(blobHash.Hash[:], blobReqMeta...)))

	commitment := p.oracle.Get(preimage.Sha256Key(blobHash.Hash))

	// Reconstruct the full blob from the 4096 field elements.
	blob := eth.Blob{}
	fieldElemKey := make([]byte, 80)
	copy(fieldElemKey[:48], commitment)
	for i := 0; i < params.BlobTxFieldElementsPerBlob; i++ {
		binary.BigEndian.PutUint64(fieldElemKey[72:], uint64(i))
		fieldElement := p.oracle.Get(preimage.BlobKey(crypto.Keccak256(fieldElemKey)))

		copy(blob[i<<5:(i+1)<<5], fieldElement[:])
	}

	blobCommitment, err := blob.ComputeKZGCommitment()
	if err != nil || !bytes.Equal(blobCommitment[:], commitment[:]) {
		panic(fmt.Errorf("invalid blob commitment: %w", err))
	}

	return &blob
}
```

### Miscellaneous

Besides the main modules mentioned above, there are others such as the client sign module responsible for signing type3 transactions, and modules related to fault proof, which will be detailed in the next section. Here, we won't elaborate further.

[Pull Request(5452), related to fault proof](https://github.com/ethereum-optimism/optimism/commit/4739b0f8bfe2f3848af3f1a5661a038c5d602b2f#diff-790daa91002e5c07497fdc2d7c2149b551d77ccec1b1906cc70f575b7c7bad65)

[Pull Request(9182), related to client sign](https://github.com/ethereum-optimism/optimism/pull/9185/files#diff-8046655b02fcced5322724e2cd61ece649a9d79ba09405093f9ed70b2087e47d)

