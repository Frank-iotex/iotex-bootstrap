# BlockDAO
BlockDAO manages data access to the blockchain, for example reading a raw data block, querying a tx by hash, and committing a block to the blockchain

The blockchain data consist of multiple block files (chain.db, chain-00000001.db, etc), the state trie db file (trie.db), and the index db (index.db)

We'll be focusing on block file in the following

## Legacy file (before db split activation at Ithaca height)
Each db file is a key-value db storing raw data in multiple buckets, using *block hash as the key*
1. block header in bucket named "bhr"
2. block body in bucket named "bbd"
3. block footer in bucket named "bfr"
4. block receipt in bucket named "rpt"

In particular, the very first db file (`chain.db`) stores the additional information:
1. the height and hash of the tip block is stored in bucket named "blk"
2. the height/hash mapping of all blocks in bucket named "h2h"
3. transaction log of all blocks in bucket named "syl"

When committing a block, 2 things happen sequentially:
1. block data are written into corresponding db file (like chain-00000001.db)
2. height/hash of the new block, height/hash mapping, and transaction log are written to `chain.db`

## New file (after db split activation at Ithaca height)
The quest of a new file comes from 2 considerations:

First, storing the height/hash mapping in the first file (chain.db) has led to dependency of the BlockDAO service on this special file, we want to remove that dependency such that **the blockchain is able to start up and running from any block db file**

Second, using hash as primary key and breaking a block into 3 parts consumed more storage space. We have done testing that shows using block height as (single incrementing) key and storing the whole block can reduce file size by 30%

The new file is still a key-value db, but *using block height as the key*
1. the height/hash mapping, and transaction log are written in the file itself (to remove dependency on `chain.db`)
2. in addition to height/hash of tip block, we also store the height of the starting block in this file. Doing so allows us to quickly know if a block is stored within a particular db file (starting height <= height <= tip height)

Here's the comparison

File | Primary key | Height/hash mapping | Transaction log | Start height
--- | --- | :---: | :---: | :---:
Legacy | block hash | No (except `chain.db') | No (except `chain.db') | No
New | block height | Yes | Yes | Yes

## Managing legacy and new files in BlockDAO
In order to handle each individual db file we abstract a FileDAO interface, which contains API to read/write block/action in current BlockDAO interface
```go
FileDAO interface {
    Start(ctx context.Context) error
    Stop(ctx context.Context) error
    Height() (uint64, error)
    GetBlockHash(uint64) (hash.Hash256, error)
    GetBlockHeight(hash.Hash256) (uint64, error)
    GetBlock(hash.Hash256) (*block.Block, error)
    GetBlockByHeight(uint64) (*block.Block, error)
    GetReceipts(uint64) ([]*action.Receipt, error)
    ContainsTransactionLog() bool
    TransactionLogs(uint64) (*iotextypes.TransactionLogs, error)
    PutBlock(context.Context, *block.Block) error
    DeleteTipBlock() error
}
```

and a FileDAONew interface for the new db file 

```go
FileDAONew interface {
    FileDAO
    Bottom() (uint64, error)
    ContainsHeight(uint64) bool
}
```

the fileDAO object to implement the interface

```go
// fileDAO implements FileDAO
fileDAO struct {
    currFd   FileDAO
    legacyFd FileDAO    
    newFd    map[uint64]FileDAONew
}
```

1. legacyFd is the interface that manages **all** legacy files
2. newFd is a map of FileDAONew interface, each of which points to **one** new file
3. currFd points to the newest db file (that commit block should write to)

![BlockDAO](https://github.com/iotexproject/iotex-bootstrap/blob/master/blockdao/blockdao.jpg "BlockDAO example")

