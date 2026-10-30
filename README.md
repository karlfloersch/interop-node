# interop-node

A minimal Docker Compose setup for running OP Mainnet + Unichain, or OP Sepolia + Unichain Sepolia.

Each network runs:

- **op-supernode**: one consensus process for both chains (shared L1 connection, one RPC).
- **op-reth** for each chain: the execution client, run as a pruned full node.

Nodes sync from peers (`execution-layer` sync), not by re-deriving everything from L1.

## Quickstart

```sh
cp .env.example .env
# Fill in the L1 RPC + beacon URLs, and set DATA_DIR to a large disk.
docker compose up -d
```

`COMPOSE_PROFILES` in `.env` selects the network: `sepolia`, `mainnet`, or `mainnet,sepolia`.

## Requirements

- An L1 execution RPC and an L1 beacon API (the beacon must serve blobs).
- Fast disk (NVMe/SSD). Budget at least 2 TB per network. Chains keep growing, so leave headroom.
- 16+ cores and 64 GB RAM are comfortable for one network.

## Endpoints

By default ports bind to `127.0.0.1`. Set `RPC_BIND=0.0.0.0` to expose them.

| Service            | Profile | Host port |
|--------------------|---------|-----------|
| OP Mainnet RPC     | mainnet | 8545      |
| Unichain RPC       | mainnet | 8546      |
| Supernode RPC      | mainnet | 9545      |
| OP Sepolia RPC     | sepolia | 8547      |
| Unichain Sepolia RPC | sepolia | 8548 |
| Supernode RPC      | sepolia | 9546      |

Per-chain op-node RPCs are namespaced under the supernode by chain ID, e.g. `http://localhost:9546/11155420`.

## Checking sync

```sh
# Execution head
curl -s -H 'content-type: application/json' localhost:8547 \
  -d '{"jsonrpc":"2.0","id":1,"method":"eth_blockNumber","params":[]}'

# Per-chain consensus status (unsafe / safe / finalized)
curl -s -H 'content-type: application/json' localhost:9546 \
  -d '{"jsonrpc":"2.0","id":1,"method":"supernode_syncStatus","params":[]}'

docker compose logs -f
```

op-reth first downloads headers back to genesis, then bodies, then executes. Expect it to take many
hours to days, depending on hardware.

## OP Mainnet: one-time Bedrock import

OP Mainnet history before Bedrock can't be executed by op-reth. Before first start, import the Bedrock
state snapshot (see the
[op-reth guide](https://docs.optimism.io/node-operators/op-reth/run/faq/sync-op-mainnet)):

```sh
# Put world_trie_state.jsonl (decompressed) in $DATA_DIR/op-mainnet/, then:
docker compose run --rm --no-deps op-mainnet-reth \
  init-state --without-ovm --chain optimism --datadir /data /data/world_trie_state.jsonl
```

Unichain and the Sepolia chains need no extra steps.

## Upgrading

Image versions are pinned in `docker-compose.yml`. Override them with `OP_RETH_IMAGE` / `OP_SUPERNODE_IMAGE`
in `.env`, then `docker compose up -d`.
