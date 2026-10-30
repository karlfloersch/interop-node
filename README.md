# interop-node

A minimal Docker Compose setup for running OP Mainnet + Unichain, or OP Sepolia + Unichain Sepolia.

OP Mainnet and Unichain are activating interop together, so they run as a pair:

- **op-supernode**: one consensus process for both chains, so it can check cross-chain messages.
- **op-reth** for each chain: the execution client that serves RPC.

Nodes sync blocks from peers (`execution-layer` sync). They don't re-derive the whole chain from L1.

Tested with op-supernode v1.1.0 and op-reth v2.8.0.

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

| Service          | Profile | HTTP | WS   | Metrics |
|------------------|---------|------|------|---------|
| OP Mainnet       | mainnet | 8545 | 8645 | 9001    |
| Unichain         | mainnet | 8546 | 8646 | 9002    |
| Supernode        | mainnet | 9545 | –    | 7300    |
| OP Sepolia       | sepolia | 8547 | 8647 | 9003    |
| Unichain Sepolia | sepolia | 8548 | 8648 | 9004    |
| Supernode        | sepolia | 9546 | –    | 7301    |

- **op-reth HTTP/WS:** bound to `RPC_BIND`, which defaults to `127.0.0.1`. HTTP includes `debug` and
  `trace`. If you set `RPC_BIND=0.0.0.0`, Docker publishes these ports past host firewalls, so put a proxy
  or allowlist in front.
- **Supernode RPC and all metrics:** always bound to `127.0.0.1`. The supernode RPC includes
  peer-management methods, so keep it private.
- **P2P ports:** open inbound TCP+UDP.
  - mainnet: 30303–30304 and 9222–9223
  - sepolia: 30305–30306 and 9224–9225

## Is it synced?

The commands below use the mainnet ports; on Sepolia, use 8547/8548 and 9546 with chain IDs 11155420/1301.

```sh
# Latest block: its timestamp should be within a few seconds of now.
curl -s -H 'content-type: application/json' localhost:8545 \
  -d '{"jsonrpc":"2.0","id":1,"method":"eth_getBlockByNumber","params":["latest",false]}'

# Per-chain consensus status (/10 = OP Mainnet, /130 = Unichain).
curl -s -H 'content-type: application/json' localhost:9545/10 \
  -d '{"jsonrpc":"2.0","id":1,"method":"optimism_syncStatus","params":[]}'

docker compose logs -f op-mainnet-reth mainnet-supernode
```

op-reth downloads headers, then bodies, then executes blocks. Until it is executing near the tip:

- The RPC reports block `0`.
- `eth_syncing` returns stage progress instead of `false`.
- `safe` and `finalized` return null.
- The supernode already shows the network tip as `unsafe_l2`, so don't use that to judge sync.

Restarting during the headers stage restarts header download.

From scratch, Sepolia takes hours to a day or two. Mainnet from a snapshot takes hours.

## Crediting deposits

- **Credit on `finalized`, or on `safe` at the least.** Never credit on `latest` or on WS subscriptions,
  which follow the unsafe head and can reorg. After interop activates, `safe` also means "cross-chain
  messages verified".
- **On a freshly synced node, wait before crediting.** Its first `safe`/`finalized` heads are set 12 hours
  behind the synced tip, taken from peers rather than derived from L1. Wait until `safe_l2` and
  `finalized_l2` in `optimism_syncStatus` have advanced past the block where sync finished.

## OP Mainnet: before first start

op-reth can't execute OP Mainnet's pre-Bedrock history, so it can't start from an empty datadir.

1. From [datadirs.optimism.io](https://datadirs.optimism.io/), download
   `latest-op-mainnet-reth-full-storage-v2` (or the `-archive-storage-v2` variant for archive nodes).
2. Verify it against its `.sha256sum`.
3. Extract it into `./data/op-mainnet` (or your `DATA_DIR`). Follow the
   [restore guide](https://docs.optimism.io/node-operators/guides/management/restore-from-snapshot).

The snapshot's pruning sets how much older history you have. A full snapshot doesn't contain older
receipts, even if `RETH_HISTORY` keeps them from then on.

If you already started OP Mainnet without a snapshot, run `docker compose down` and
`sudo rm -rf ./data/op-mainnet`, then restore.

Unichain and the Sepolia chains need no extra steps.

## Monitoring

Most failures stall a chain quietly. The containers keep running, so Docker won't restart anything.

- **Alerts:** alert when a chain's latest block timestamp falls behind wall-clock time, when its
  `safe_l2` stops advancing, and on disk usage of `DATA_DIR`. Metrics are on the ports above.
- **L1 outages:** both chains share one L1 connection. If L1 or the beacon is down or rate-limited, the
  unsafe head keeps following peers, but `safe` and `finalized` stop advancing.
- **Troubleshooting:** check `docker compose logs mainnet-supernode` (or `sepolia-supernode`). An L1 or
  beacon URL problem is the usual cause.

## Operating

- **Stop:** `docker compose down`. op-reth gets 5 minutes to flush to disk.
- **Upgrade:** change the image tags (or set `OP_RETH_IMAGE` / `OP_SUPERNODE_IMAGE` in `.env`), then
  `docker compose pull && docker compose up -d`.
- **Resync one chain:** `docker compose down`, then `sudo rm -rf ./data/<chain>`, then `docker compose up -d`.
  For OP Mainnet, restore the snapshot again before `up`.
- **History mode:** set `RETH_HISTORY` before the first start. op-reth saves its prune settings in
  `<datadir>/reth.toml` and keeps them even if `RETH_HISTORY` is later empty. Check the
  `Pruner initialized prune_config=` log line.
- **Offline more than ~18 days, or restoring an old snapshot:** the L1 beacon may no longer have the blobs
  the node needs. Add `- --l1.beacon-fallbacks=<blob archive URL>` to the supernode command (see the
  [blobs guide](https://docs.optimism.io/node-operators/guides/management/blobs)).

## Interop activation

These images don't schedule interop yet. Upgrade to the releases announced for the interop hardfork before
it activates; see the [interop prep notice](https://docs.optimism.io/notices/interop-prep).

After activation, the paired chains depend on each other for safety:

- If one chain's node lags or is resyncing, the other chain's `safe` and `finalized` heads stop advancing.
  Unsafe heads and RPC keep working.
- Keep both chains healthy.
- Resync a chain by resyncing the whole network, not one chain at a time.
