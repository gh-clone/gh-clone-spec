# Plan for gh-clone-search

### Tier 2: Version Control Storage & Code Search (Spokes & Blackbird)
- [ ] Define the Spokes internal gRPC interface (`Fetch`, `Push`, `UpdateRef`, `Checksum`).
- [ ] Implement a Raft state machine in Rust for Git replica consensus, managing leader election across storage racks.
- [ ] Configure HAProxy edge proxies to intelligently route Git-over-SSH and Git-over-HTTPS to the active Spokes primary node.
- [ ] Implement a distributed Git reference-locking mechanism (via Redis/ZooKeeper) to prevent concurrent push race conditions.
- [ ] Write a specialized `.pack` file parser in Rust to extract and validate commits during the push receive-pack phase.
- [ ] Implement a 3-way merge algorithm using `libgit2` in-memory.
- [ ] Build a memory-arena allocator for the merge engine to handle large AST diffing without runtime Garbage Collection pauses.
- [ ] Implement strict conflict marker (`<<<<<<< HEAD`) detection during the virtual tree traversal.
- [ ] Create an asynchronous background task queue to compute and cache Pull Request diffs using the Patience/Histogram diff algorithm.
- [ ] Build the Blackbird indexer daemon: A Rust worker permanently tailing the Kafka `push` topic.
- [ ] Integrate `tree-sitter` grammars into the indexer for the top 10 languages (C, C++, JS, Python, Go, Rust, Ruby, Java, TS, PHP).
- [ ] Extract semantic symbol definitions (classes, functions, constants) by executing compiled tree-sitter queries against file blobs.
- [ ] Generate byte-level sparse trigrams from source code documents to power structural regex matching.
- [ ] Construct Finite State Transducers (FSTs) to map incoming regex queries into their optimal trigram intersection paths.
- [ ] Implement a query planner to intersect trigram FSTs and filter file candidates *before* executing full `hyperscan` regex engines.
- [ ] Implement a document scoring algorithm based on the PageRank of intra-repository import/dependency graphs.
- [ ] Store tokenized code chunks in an LSM-tree (RocksDB) partitioned efficiently by repository ID.
- [ ] Implement incremental code indexing: only hash, parse, and index the specific blobs that changed in a given commit.
- [ ] Expose an internal gRPC search endpoint that scatter-gathers user queries across the sharded Blackbird cluster and aggregates results.

