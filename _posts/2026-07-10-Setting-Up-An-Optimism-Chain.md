---
layout: post
title: "Setting Up Your Own OP Stack Chain"
description: "A practical walkthrough for deploying your own Optimism L2 rollup against Sepolia using op-deployer, Docker, and the standard OP Stack components."
tags: Ethereum OP-Stack Optimism L2
---

I recently went through the process of deploying a custom L2 chain built on the
[OP Stack](https://docs.optimism.io/stack/getting-started). This post documents
the full journey — from generating wallets to a running sequencer with a batcher
and proposer submitting data back to L1.

The target environment is Sepolia as the L1, but the same steps apply to any
EVM-compatible L1.

## What is the OP Stack?

The OP Stack is the open-source codebase that powers Optimism and the entire
Superchain ecosystem. It gives you a production-grade optimistic rollup out of
the box: transactions are executed off-chain, compressed into batches, and posted
to L1 for data availability. Fraud proofs (via the dispute game) protect the
bridge.

At a high level the components you will run are:

| Component | Role |
|-----------|------|
| `op-reth` | Execution client (EL) — processes transactions, holds state |
| `op-node` | Consensus / rollup node (CL) — drives the sequencer, derives L2 from L1 |
| `op-batcher` | Compresses L2 batches and submits them to the L1 inbox |
| `op-proposer` | Proposes L2 output roots on L1 for the bridge to use |

## Prerequisites

| Dependency | Install |
|------------|---------|
| [Docker](https://docs.docker.com/get-docker/) + Compose | Docker Desktop |
| [Foundry](https://getfoundry.sh) (`cast`) | `curl -L https://foundry.paradigm.xyz \| bash` |
| [jq](https://github.com/jqlang/jq) | `brew install jq` |
| `op-deployer` | see below |

### Installing op-deployer

`op-deployer` is the tool that deploys all the L1 contracts (bridge, dispute
game factory, system config, etc.) and generates the genesis configuration for
your L2.

Download the binary from the [OP releases page](https://github.com/ethereum-optimism/optimism/releases),
find the latest release that includes `op-deployer`, and grab the asset for your
platform.

```bash
tar -xvzf path/to/op-deployer-archive.tar.gz
chmod +x op-deployer

# Intel Mac
sudo mv op-deployer /usr/local/bin/op-deployer

# Apple Silicon Mac
sudo mv op-deployer /opt/homebrew/bin/op-deployer
```

On macOS you may get a quarantine warning. Clear it with:

```bash
sudo xattr -dr com.apple.quarantine $(which op-deployer)
```

Verify:

```bash
op-deployer --version
```

## Step 1: Generate wallets

The OP Stack uses separate keys for each operational role. You can generate them
with `cast wallet new` or any key management tool you prefer.

| Role | Purpose |
|------|---------|
| `deployer` | Deploys L1 contracts (one-time, ~2–3 ETH on Sepolia) |
| `admin` | General admin key |
| `proxy_admin_owner` | Upgrades proxy contracts |
| `system_config_owner` | Updates `SystemConfig` on L1 |
| `unsafe_block_signer` | Signs unsafe (pre-confirmation) blocks on the sequencer |
| `batcher` | Submits compressed batches to L1 |
| `proposer` | Proposes L2 output roots on L1 |
| `challenger` | Challenges invalid proposals in the dispute game |

Store private keys securely (e.g. in a secrets manager). Never commit them to
version control.

Also generate a JWT secret for the authenticated Engine API between op-node and
op-reth:

```bash
openssl rand -hex 32 > jwt.txt
```

## Step 2: Fund wallets

Before deploying, top up these accounts with Sepolia ETH:

- **deployer** — ~2–3 ETH to cover contract deployment gas
- **batcher** — ongoing operational costs; replenish regularly
- **proposer** — ongoing operational costs; replenish regularly

The other keys don't need balance upfront.

## Step 3: Initialise the deployment intent

`op-deployer` works with an *intent* file that describes what you want to deploy.
Initialise a working directory for it:

```bash
op-deployer init \
  --l1-chain-id 11155111 \
  --l2-chain-ids <YOUR_CHAIN_ID> \
  --workdir .deployer \
  --intent-type standard-overrides
```

This creates `.deployer/intent.toml`. The auto-generated `l1ContractsLocator`,
`l2ContractsLocator`, and `opcmAddress` values point to the correct on-chain
contract sets for the OP Stack version — do not change them.

## Step 4: Configure intent.toml

Open `.deployer/intent.toml` and fill in the role addresses from step 1:

```toml
[[chains]]
  id = "<YOUR_CHAIN_ID_HEX>"

  baseFeeVaultRecipient      = "<fee-recipient-address>"
  l1FeeVaultRecipient        = "<fee-recipient-address>"
  sequencerFeeVaultRecipient = "<fee-recipient-address>"
  operatorFeeVaultRecipient  = "<fee-recipient-address>"

  [chains.roles]
    l1ProxyAdminOwner  = "<proxy_admin_owner_address>"
    l2ProxyAdminOwner  = "<proxy_admin_owner_address>"
    systemConfigOwner  = "<system_config_owner_address>"
    unsafeBlockSigner  = "<unsafe_block_signer_address>"
    batcher            = "<batcher_address>"
    proposer           = "<proposer_address>"
    challenger         = "<challenger_address>"
```

If you want to pre-fund an address in the L2 genesis (useful for contract
deployments on day one), add an alloc entry:

```toml
# 100,000 ETH in wei, hex-encoded
[chains.genesisAlloc."<ADDRESS>"]
  balance = "0x152D02C7E14AF6800000"
```

## Step 5: Deploy L1 contracts

```bash
op-deployer apply \
  --workdir .deployer \
  --l1-rpc-url $L1_RPC_URL \
  --private-key $DEPLOYER_PRIVATE_KEY
```

This deploys the full L1 contract set: `OptimismPortal`, `L1CrossDomainMessenger`,
`L1StandardBridge`, `DisputeGameFactory`, `SystemConfig`, and more. On success
the deployment state lands in `.deployer/state.json`.

## Step 6: Generate genesis and rollup config

Extract the L2 genesis block and rollup configuration from the deployment state:

```bash
op-deployer inspect genesis --workdir .deployer <YOUR_CHAIN_ID> > sequencer/genesis.json
op-deployer inspect rollup  --workdir .deployer <YOUR_CHAIN_ID> > sequencer/rollup.json
```

Depending on the op-node version you are running, you may need to strip fields
that were added in newer protocol versions. For op-node v1.19.x:

```bash
python3 -c "
import json
with open('sequencer/rollup.json') as f:
    d = json.load(f)
d['genesis']['system_config'].pop('minBaseFee', None)
d['genesis']['system_config'].pop('daFootprintGasScalar', None)
with open('sequencer/rollup.json', 'w') as f:
    json.dump(d, f, indent=2)
"
```

## Step 7: Start the sequencer

The sequencer runs two containers that communicate over the Engine API:

- **op-reth** (execution layer) — handles EVM execution and state
- **op-node** (consensus layer) — drives block production and derives from L1

A minimal `docker-compose.yml` for the sequencer:

```yaml
services:
  op-reth:
    image: ghcr.io/paradigmxyz/op-reth:latest
    volumes:
      - ./data:/data
      - ./genesis.json:/genesis.json
      - ./jwt.txt:/jwt.txt
    command: >
      node
      --chain /genesis.json
      --datadir /data
      --authrpc.jwtsecret /jwt.txt
      --authrpc.addr 0.0.0.0
      --authrpc.port 8551
      --http
      --http.addr 0.0.0.0
      --http.port 8545
      --http.api eth,net,web3,debug,txpool
    ports:
      - "8545:8545"

  op-node:
    image: us-docker.pkg.dev/oplabs-tools-artifacts/images/op-node:latest
    depends_on: [op-reth]
    volumes:
      - ./rollup.json:/rollup.json
      - ./jwt.txt:/jwt.txt
    command: >
      op-node
      --rollup.config /rollup.json
      --l1 $L1_RPC_URL
      --l1.beacon $L1_BEACON_URL
      --l2 http://op-reth:8551
      --l2.jwt-secret /jwt.txt
      --sequencer.enabled
      --sequencer.l1-confs 4
      --p2p.sequencer.key $P2P_SEQUENCER_KEY
      --p2p.advertise.ip $P2P_ADVERTISE_IP
      --rpc.addr 0.0.0.0
      --rpc.port 8547
```

Start it:

```bash
docker compose up -d
```

Check that blocks are being produced:

```bash
cast block-number --rpc-url http://localhost:8545
# wait a few seconds
cast block-number --rpc-url http://localhost:8545
# the number should increase
```

## Step 8: Start the batcher

The batcher reads L2 blocks from the sequencer, compresses them into frames, and
posts them to the L1 batcher inbox address.

```yaml
services:
  op-batcher:
    image: us-docker.pkg.dev/oplabs-tools-artifacts/images/op-batcher:latest
    command: >
      op-batcher
      --l1-eth-rpc $L1_RPC_URL
      --l2-eth-rpc http://sequencer:8545
      --rollup-rpc http://sequencer:8547
      --private-key $BATCHER_PRIVATE_KEY
      --max-channel-duration 25
      --sub-safety-margin 4
```

Verify it is working:

```bash
docker compose logs -f op-batcher
# Look for: "Submitted batch"
```

## Step 9: Start the proposer

The proposer periodically reads the latest safe L2 block from the sequencer,
computes an output root, and submits it to the `DisputeGameFactory` on L1. The
bridge uses these roots to process withdrawals.

```yaml
services:
  op-proposer:
    image: us-docker.pkg.dev/oplabs-tools-artifacts/images/op-proposer:latest
    command: >
      op-proposer
      --l1-eth-rpc $L1_RPC_URL
      --rollup-rpc http://sequencer:8547
      --private-key $PROPOSER_PRIVATE_KEY
      --game-factory-address $DISPUTE_GAME_FACTORY_ADDRESS
      --proposal-interval 120s
```

The `DISPUTE_GAME_FACTORY_ADDRESS` is in `.deployer/state.json` under
`disputeGameFactoryProxy`.

Verify:

```bash
docker compose logs -f op-proposer
# Look for: "Proposing output root"
```

## Putting it all together

Once all three services are running you have a fully functional L2:

- **Sequencer** accepts transactions and produces blocks
- **Batcher** keeps L1 data availability up to date
- **Proposer** publishes output roots so withdrawals can be processed after the
  challenge window

The L2 is live at `http://localhost:8545`. You can add it to MetaMask or any
EVM wallet using your chain ID and start deploying contracts.

## Tips

**Batcher balance matters.** If the batcher wallet runs out of ETH the chain will
keep producing blocks but data availability will fall behind. Set up monitoring
and alerts on the batcher balance.

**Genesis hash drift.** If you modify `genesis.json` after deployment (e.g. to
pre-fund accounts or add precompiles), op-reth will compute a different genesis
block hash than what `rollup.json` records. Spin up op-reth in a temporary
container, query block 0, and patch `rollup.json` manually:

```bash
HASH=$(cast block 0 --rpc-url http://localhost:8545 --field hash)
# update .genesis.l2.hash in rollup.json with $HASH
```

**op-deployer is idempotent.** You can re-run `op-deployer apply` safely — it
checks what is already deployed and skips completed steps.

**Local development without Sepolia ETH.** Point op-deployer at a local Anvil
node forking Sepolia (`anvil --fork-url <sepolia-rpc>`). Deployment is instant
and free, which is great for iteration.
