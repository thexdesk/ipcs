## ipcs

[![GoDoc](https://godoc.org/github.com/hinshun/ipcs?status.svg)](https://godoc.org/github.com/hinshun/ipcs)
[![Build Status](https://travis-ci.org/hinshun/ipcs.svg?branch=master)](https://travis-ci.org/hinshun/ipcs)
[![Go Report Card](https://goreportcard.com/badge/github.com/hinshun/ipcs)](https://goreportcard.com/report/github.com/hinshun/ipcs)

Containerd meets [IPFS](github.com/ipfs/go-ipfs). Peer-to-peer distribution of content blobs.

## Getting started

Converting a manifest from DockerHub to p2p manifest:

```sh
# Term 1: Start a IPFS daemon
$ make ipfs

# Term 2: Start a rootless containerd backed by ipcs.
$ make containerd

# Term 3: Convert alpine to a p2p manifest
$ make convert
```

Converting two manifests from DockerHub to p2p manifests, and then comparing the number of shared IPLD nodes (layers chunked into 262KiB blocks):

```sh
# Term 1: Start a IPFS daemon
$ make ipfs

# Term 2: Start a rootless containerd backed by ipcs.
$ make containerd

# Term 3: Convert ubuntu:bionic and ubuntu:xenial into p2p manifests, then bucket IPLD nodes into nodes unique to each image, and nodes inside intersect.
$ make compare
```

## Design

IPFS backed container image distribution is not new. Here is a non-exhaustive list of in-the-wild implementations:
- https://github.com/docker/distribution/pull/2906
- https://github.com/miguelmota/ipdr
- https://github.com/jvassev/image2ipfs/

P2P container image distribution is also implemented with different P2P networks:
- https://github.com/dragonflyoss/Dragonfly
- https://github.com/uber/kraken

The previous IPFS implementations all utilize the [Docker Registry HTTP API V2](https://docs.docker.com/registry/spec/api/) to distribute. However, the connection between containerd pulling the image and registry is not peer-to-peer, and if the registry was ran as a sidecar the content would be duplicated twice in the local system. Instead, I chose to implement it as a containerd content plugin for the following reasons:
- Containerd natively uses IPFS as a `content.Store`, no duplication.
- Allow p2p and non-p2p manifests to live together.
- Potentially do [file-granularity chunking](https://github.com/AkihiroSuda/filegrain) by introducing new layer mediatype.
- Fulfilling the `content.Store` interface will allow using `ipcs` to also back the [buildkit](https://github.com/moby/buildkit) cache.

IPFS imposes a 4 MiB limit for blocks because it may be run in a public network with adversarial peers. Since its not able to verify hashes until all the content has arrived, an attacker can send gibberish flooding connections and consuming bandwidth. Chunking data into smaller blocks also aids in deduplication:

![Chunking blocks in IPFS](chunking.png)

IPCS implements containerd's `content.Store` interface and can be built as a golang plugin to override containerd's default local store. A converter implementation is also provided that converts a regular OCI image manifest to a manifest where every descriptor is replaced with the descriptor of the root DAG node added to IPFS. The root node is the merkle root of the 262KiB chunks of the layer.

![Converting to P2P](containerd-ipfs.png)

Although the IPFS daemon or its network may already have the bytes for all image's P2P content, containerd has a [boltdb](https://github.com/boltdb/bolt) metadata store wrapping the underlying `content.Store`.

A image pull, starting from the client side goes through the following layers:
- `proxy.NewContentStore` (content.Store)
- content.ContentClient (gRPC client)
- content.NewService (gRPC server: plugin.GRPCPlugin "content")
- content.newContentStore (content.Store: plugin.ServicePlugin, services.ContentService)
- metadata.NewDB (bolt *metadata.DB: plugin.MetadataPlugin "bolt")
- ipcs.NewContentStore (content.Store: plugin.ContentPlugin, "ipcs")

So in the case of this project `ipcs`, a pull is simply flushing through its `content.Store` layers to register the image in containerd's metadata stores. Note that the majority of the blocks don't need to be downloaded into IPFS's local storage in order to complete a pull, and can be delayed until unpacking the layers into snapshots.

## Results

Collected data on: `6/4/2019`

Systems:
- m5.large x 3
- 8.0 GiB Memory
- 2 vCPUs
- Up to 10 Gigabit (Throttled by [AWS network credits](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/burstable-performance-instances.html))
- Linux kernel 4.4.0
- Ubuntu 16.04.6 LTS
- Containerd v1.2.6
- IPFS v0.4.21

Comparison:
- Pull from DockerHub / Private docker registries
- Shard content chunks evenly to 3 nodes such that each node has roughly 33% of IPFS blocks.

Image | Total size (bytes) | IPFS blocks | DockerHub pull (secs) | IPFS pull (secs) | Diff (Hub/IPFS)
----- | ------------------ | ----------- | --------------------- | ---------------- | ---------------
docker.io/library/alpine:latest | 2759178 | 14 | 1.430587576 | 0.96249954 | 148.63%
docker.io/library/ubuntu:latest | 28861894 | 38 | 2.079848393 | 6.93721404 | 29.98%
docker.io/library/golang:latest | 296160075 | 380 | 4.817960124 | 55.26376992 | 8.85%
docker.io/ipfs/go-ipfs:latest | 23545678 | 103 | 1.182348947 | 8.134937865 | 14.53%

IPFS's performance seems to slow down drastically as the number of nodes (size of total image) goes up. As seen from `make compare`, there also doesn't seem to be any improvements in deduplication between IPFS chunks as opposed to OCI layers:

```sh
$ GO111MODULE=on IPFS_PATH=./tmp/ipfs go run ./cmd/compare docker.io/library/alpine:latest docker.io/library/ubuntu:latest docker.io/library/golang:latest docker.io/ipfs/go-ipfs:latest
// ...
2019/06/04 13:39:55 Found 1381 blocks
docker.io/ipfs/go-ipfs:latest: 46891351
docker.io/library/alpine:latest: 5516903
docker.io/library/golang:latest: 828096081
docker.io/library/ubuntu:latest: 57723854
// Zero block intersection, they are very different images though.
```

Serious usage of p2p container image distribution should consider Dragonfly and Kraken, because IPFS suffers from performance issues:
- [ ] Bitswap transfer is slow [ipfs/go-ipfs#5723](https://github.com/ipfs/go-ipfs/issues/5723)
- [ ] IPFS uses excessive bandwidth [ipfs/go-ipfs#3429](https://github.com/ipfs/go-ipfs/issues/3429)
- [ ] Slow pins when there are many pins [ipfs/go-ipfs#5221](https://github.com/ipfs/go-ipfs/issues/5221) 

Related benchmarking:
- https://github.com/ipfs/go-ipfs/issues/5226
- https://github.com/ipfs/go-ipfs/issues/6382
- https://github.com/ipfs/go-ipfs/issues/3131
- https://github.com/ipfs/go-ipfs/issues/3316
- https://github.com/ipfs/go-ipfs/issues/3317
- https://github.com/ipfs/go-ipfs/issues/3860
- https://github.com/ipfs/go-ipfs/issues/5037
- https://github.com/ipfs/go-ipfs/issues/5420

## Next steps

Explore deduplication by adding each layer's uncompressed, untared files into IPFS to get chunked-file-granular deduplication. IPFS's Unixfs (UNIX/POSIX fs features implemented via IPFS) needs the following:
- Support for tar file metadata (uid, gid, modtime, xattrs, executable bit, etc):
  - [ ] https://github.com/ipfs/unixfs-v2/issues/1
- Support for hard links, character/block devices, fifo:
  - [ ] https://github.com/ipfs/go-ipfs/issues/1642
- Implementation of `diff.Comparer` and `diff.Applier` to apply custom IPFS layer mediatype to containerd's tmpmount. 

Explore IPFS-FUSE mounted layers for lazy container rootfs:
- Same requirements as above.
- Possible snapshotter interface changes
  - [ ] https://github.com/containerd/containerd/issues/2968
  - [ ] https://github.com/containerd/containerd/issues/2943
  - https://docs.google.com/document/d/1TmNWnmbx-fd2wUhujuUtAeSUuulWZyhggAKCSc1F3AE/edit
