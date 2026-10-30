# interop-node

A minimal Docker Compose setup for running OP Mainnet + Unichain, or OP Sepolia + Unichain Sepolia.

OP Mainnet and Unichain are activating interop together, so they run as a pair:

- **op-supernode**: one consensus process for both chains, so it can check cross-chain messages.
- **op-reth** for each chain: the execution client that serves RPC.

Nodes sync blocks from peers (`execution-layer` sync). They don't re-derive the whole chain from L1.

## Requirements

- Docker Engine with Compose v2.
- An L1 execution RPC and an L1 beacon API. The beacon API must serve blobs.
- NVMe/SSD storage. Plan for 4 TB+ for mainnet (leaves room to restore a snapshot) and 2 TB for Sepolia.
  Archive nodes need much more.
- 16+ cores and 64 GB RAM per network.

## Quickstart

```sh
cp .env.example .env
# Edit .env: set the L1 URLs, set DATA_DIR to a large disk, and choose RETH_HISTORY.
```

**Mainnet only:** before the first start, restore an OP Mainnet snapshot (see [below](#op-mainnet-before-first-start)).

```sh
docker compose up -d
```

`COMPOSE_PROFILES` in `.env` selects the network: `mainnet` (default), `sepolia`, or `mainnet,sepolia`.

## Ports

| Service              | Profile | HTTP | WS   |
|----------------------|---------|------|------|
| OP Mainnet           | mainnet | 8545 | 8645 |
| Unichain             | mainnet | 8546 | 8646 |
| Supernode            | mainnet | 9545 | –    |
| OP Sepolia           | sepolia | 8547 | 8647 |
| Unichain Sepolia     | sepolia | 8548 | 8648 |
| Supernode            | sepolia | 9546 | –    |

- **op-reth RPC:** bound to `RPC_BIND`, which defaults to `127.0.0.1`. If you set it to `0.0.0.0`, Docker
  publishes past host firewalls. Put a proxy or allowlist in front.
- **Supernode RPC:** always bound to `127.0.0.1`. It includes peer-management methods, so keep it private.
- **P2P ports:** open inbound TCP+UDP.
  - mainnet: 30303–30304 and 9222–9223
  - sepolia: 30305–30306 and 9224–9225

## Is it synced?

```sh
# Execution head: compare with a block explorer. eth_syncing returns false at tip.
curl -s -H 'content-type: application/json' localhost:8545 \
  -d '{"jsonrpc":"2.0","id":1,"method":"eth_blockNumber","params":[]}'

# Per-chain consensus status: unsafe_l2 should be seconds behind now.
curl -s -H 'content-type: application/json' localhost:9545 \
  -d '{"jsonrpc":"2.0","id":1,"method":"supernode_syncStatus","params":[]}'

docker compose logs -f
```

op-reth downloads headers, then bodies, then executes blocks. The RPC reports block `0` until execution
starts. From scratch, Sepolia takes hours to a day or two. Mainnet from a snapshot takes hours.

Monitor that the head keeps advancing. If the L1 RPC goes down, op-reth keeps serving a stale head.
If the supernode keeps restarting, check `docker compose logs mainnet-supernode`. The usual cause is the
L1 or beacon URL.

## OP Mainnet: before first start

op-reth can't execute OP Mainnet's pre-Bedrock history, so it can't start from an empty datadir.
Restore an op-reth snapshot from [datadirs.optimism.io](https://datadirs.optimism.io/) into
`$DATA_DIR/op-mainnet`. Follow the [restore guide](https://docs.optimism.io/node-operators/guides/management/restore-from-snapshot),
including the checksum check.

The snapshot's pruning sets how much older history you have. A full snapshot doesn't contain older
receipts, even if `RETH_HISTORY` keeps them from then on.

If you already started OP Mainnet without a snapshot, run `docker compose down` and
`sudo rm -rf $DATA_DIR/op-mainnet`, then restore.

Alternative: import only the Bedrock state with `op-reth init-state --without-ovm`. See the
[op-reth guide](https://docs.optimism.io/node-operators/op-reth/run/faq/sync-op-mainnet). Then
EL sync runs from Bedrock to tip, which takes days.

Unichain and the Sepolia chains need no extra steps.

## Operating

- **Stop:** `docker compose down`. op-reth gets 5 minutes to flush to disk.
- **Upgrade:** change the image tags (or set `OP_RETH_IMAGE` / `OP_SUPERNODE_IMAGE` in `.env`), then
  `docker compose pull && docker compose up -d`.
- **Interop activation:** these images don't schedule interop yet. Upgrade to the releases announced for
  the interop hardfork before it activates.
- **Resync one chain:** `docker compose down`, then `sudo rm -rf $DATA_DIR/<chain>`, then `docker compose up -d`.
- **Offline more than ~18 days:** the L1 beacon may no longer have the blobs the node needs. Add a blob
  archive with `--l1.beacon-fallbacks` (see the
  [blobs guide](https://docs.optimism.io/node-operators/guides/management/blobs)).
