# daisugi-devnet-0 spec

:::info
:mega: daisugi-devnet-0 launched on **2026-11-18 15:00:00 UTC**. The Daisugi fork activates at epoch 10 (**16:04:00 UTC**).
:::

:::info
:checkered_flag: **Scope.** Post-quantum transaction devnet: native frame transactions (EIP-8141) with keyed nonces (EIP-8250) from genesis, and recursive STARK proof aggregation of post-quantum signatures (EIP-8288) from the Daisugi fork. Daisugi is a CL fork on top of **Electra**; Fulu, Gloas, Heze and the BPO forks are not scheduled.
:::

:::warning
:boom: Daisugi is experimental and not a scheduled mainnet fork. The EIP-8288 format follows the Nethermind prototype where the EIP is still open (see [Open questions](#Open-questions)).
:::

## EIP list for daisugi-devnet-0

| EIP | Title | Active from |
|-----|-------|-------------|
| [EIP-8141](https://eips.ethereum.org/EIPS/eip-8141) | Frame Transaction (type `0x06`) | EL genesis (`eip8141PrototypeTime`) |
| [EIP-8250](https://eips.ethereum.org/EIPS/eip-8250) | Keyed Nonces | EL genesis |
| [EIP-8288](https://eips.ethereum.org/EIPS/eip-8288) | Recursive STARK proof of post-quantum signatures | Daisugi fork (`eip8288PrototypeTime`) |

The reference implementation is Nethermind [`feature/eip8288-lean`](https://github.com/NethermindEth/nethermind/tree/feature/eip8288-lean) ([#15955](https://github.com/NethermindEth/nethermind/pull/15955)). EIP-8141 follows the rules of that commit: the signature hash commits to signature lengths ([EIPs#12502](https://github.com/ethereum/EIPs/pull/12502)) and `fees` is `[max_fee, max_priority_fee_per_gas]`.

## Network parameters

| Parameter | Value |
|-----------|-------|
| Network name | `daisugi-devnet-0` |
| Chain ID | `701563864` (`0x29d103d8`) |
| Preset | mainnet |
| Slot time | 12 s (`SLOT_DURATION_MS: 12000`) |
| Genesis time | `1795014000` (2026-11-18 15:00:00 UTC) |
| EL genesis block | `0x3c3a46744395b7beae03e0f994ec80c46d1b8d7646aced7e7b0501dcded7765d` (timestamp `1795013940`) |
| Genesis validators root | `0xbb8095d1b2aadfbc87d5981cd3783ce5e4cbb85e81de5a557593ac64c12eba98` |
| `GENESIS_FORK_VERSION` | `0x10683924` |
| `ELECTRA_FORK_EPOCH` | `0` |
| `DAISUGI_FORK_VERSION` | `0xd0683924` |
| `DAISUGI_FORK_EPOCH` | `10` (slot 320) |
| `eip8141PrototypeTime` | `1795013940` (EL genesis) |
| `eip8288PrototypeTime` | `1795017840` (2026-11-18 16:04:00 UTC) |
| Fulu, Gloas, Heze, BPO 1-2 | not scheduled (`18446744073709551615`) |
| Gas limit | `60000000` |
| `MAX_PAYLOAD_SIZE` | `20971520` |
| Validators | 4000, each 2048 ETH with `0x02` withdrawal credentials |
| Deposit contract | `0x00000000219ab540356cBB839Cbe05303d7705Fa` |

Config files (`config.yaml`, `genesis.json`, `genesis.ssz`, bootnodes): [network-configs/devnet-0/metadata](https://github.com/ethpandaops/daisugi-devnets/tree/master/network-configs/devnet-0/metadata).

### Genesis predeploys

Both with nonce 1, balance 0 and empty storage:

| Address | Code | Purpose |
|---------|------|---------|
| `0x81413f0cF12e9b6a49B1D0439E081c577D57FfFf` | `0x60083614600a575f5ffd5b5f3560c01c4211601657005b5f5ffd` | EIP-8141 expiry verifier |
| `0x8250968C12e01A19d6F667b9B2F3b3A4d0e51cB7` | `0x60006000fd` | EIP-8250 `NONCE_MANAGER` |

## Consensus layer changes

Daisugi extends the **Electra** containers. It does not include Fulu (no PeerDAS, no `proposer_lookahead`) or Gloas.

- `ExecutionPayload` gains `recursive_stark_proof: ByteList[8388608]` as its last field.
- `ExecutionPayloadHeader` gains `recursive_stark_proof_root: Root` as its last field (the `hash_tree_root` of the proof list).
- `BeaconBlockBody`, `BeaconBlock`, `SignedBeaconBlock`, `BeaconState` and the blinded/builder variants are the Electra containers with these two types.
- A block without post-quantum dependencies carries an empty proof.
- Beacon API: `version` / `Eth-Consensus-Version` is `daisugi`.
- Fork digest: computed from `DAISUGI_FORK_VERSION` and the genesis validators root, without the Fulu blob-parameter mix-in.
- Block hash check: the CL rebuilds the EL header from `keccak256(starkProof)` and a dependency hash it derives from the block's frame transactions, and rejects the block on a mismatch.

Go types for tooling: [go-eth2-client#69](https://github.com/ethpandaops/go-eth2-client/pull/69).

## Engine API

The Electra method versions stay in use after the fork: `engine_newPayloadV4`, `engine_getPayloadV4`, `engine_forkchoiceUpdatedV3`.

- `ExecutionPayload` JSON gains `starkProof` (hex bytes) from the Daisugi fork on.
- `engine_getPayloadBodies*` responses carry the proof; the CL checks it against the stored header root.
- The EL derives the header commitments itself: `keccak256(stark_proof)` and `block_deps_hash`.

## Execution layer notes

- Post-quantum transactions are submitted as proof wrappers with `eth_sendProofWrapper`.
- Accepted signature schemes: leanSPHINCS (`0x10`) and generic leanSTARK (`0x11`).
- Nethermind must run with `--TxPool.FrameTxMaxVerifyGas=500000` and `--TxPool.BlobsSupport=Disabled`. Without the second flag, post-quantum transactions do not propagate between nodes.

## Clients

| Client | Source | Image |
|--------|--------|-------|
| Nethermind | [`feature/eip8288-lean`](https://github.com/NethermindEth/nethermind/tree/feature/eip8288-lean) @ `af50ed19` | `ethpandaops/nethermind:daisugi-devnet-0` |
| Lighthouse | v8.2.2 (`e423a667`) + the [`eip8288-lean`](https://github.com/asanso/AA-PQ/tree/main/aggregation/frames/consensus/eip8288-lean) patch series from asanso/AA-PQ @ `4b0a11e` | `ethpandaops/lighthouse:daisugi-devnet-0` |

The devnet runs 4 validator nodes (1000 validators each) and 1 bootnode, all Nethermind + Lighthouse. No other client implements Daisugi yet.

## Open questions

- **`DEPS_HASH`.** EIP-8288 leaves the hash function open; the devnet uses Keccak-256, as Nethermind does.
- **leanSTARK scheme `0x11`.** Accepted beside leanSPHINCS `0x10`, which deviates from the EIP. Its entries are hashed in a second list under the `eip8288/leanstark-deps` domain.

## Tooling

| Tool | Status |
|------|--------|
| [Dora](https://dora.daisugi-devnet-0.ethpandaops.io) | Daisugi fork and STARK proof size/root per block ([dora#938](https://github.com/ethpandaops/dora/pull/938)) |
| [Checkpoint sync](https://checkpoint-sync.daisugi-devnet-0.ethpandaops.io) | serves Daisugi states and blocks ([checkpointz#269](https://github.com/ethpandaops/checkpointz/pull/269)) |
| [Forky](https://forky.daisugi-devnet-0.ethpandaops.io), [Forkmon](https://forkmon.daisugi-devnet-0.ethpandaops.io), [Tracoor](https://tracoor.daisugi-devnet-0.ethpandaops.io) | enabled |
| [Faucet](https://faucet.daisugi-devnet-0.ethpandaops.io), [RPC](https://rpc.daisugi-devnet-0.ethpandaops.io), [Beacon API](https://beacon.daisugi-devnet-0.ethpandaops.io) | enabled |
| Genesis generator | `DAISUGI_FORK_EPOCH` support in [ethereum-genesis-generator#340](https://github.com/ethpandaops/ethereum-genesis-generator/pull/340), image `ethpandaops/ethereum-genesis-generator:daisugi-devnets` |
| Assertoor, Syncoor, Xatu, Buildoor | disabled, no Daisugi support |

## Testing focus

- **Fork transition**: Electra to Daisugi at epoch 10, finality through the fork.
- **Post-quantum transactions**: valid proof wrappers are included; corrupt, early (pre-fork) and replayed wrappers are rejected.
- **Proof blocks**: blocks with dependencies carry a proof of roughly 250 kB; all nodes agree on the proof and the block hash.
- **Propagation**: a wrapper sent to one node reaches the others before inclusion.
- **Sync**: a node synced from genesis and one synced from a checkpoint reach the same head.

## Local testing

Kurtosis example with ethereum-package `5795b77e65235ad223d4d3b413b2739190bf60cc`. The package uses the generator's built-in CL config template, so the generator image must append `DAISUGI_FORK_VERSION` and `DAISUGI_FORK_EPOCH` to `config.yaml`:

```yaml
participants:
  - el_type: nethermind
    el_image: ethpandaops/nethermind:daisugi-devnet-0
    el_extra_params:
      - --TxPool.FrameTxMaxVerifyGas=500000
      - --TxPool.BlobsSupport=Disabled
    cl_type: lighthouse
    cl_image: ethpandaops/lighthouse:daisugi-devnet-0
    count: 4

network_params:
  preset: mainnet
  seconds_per_slot: 12
  genesis_gaslimit: 60000000
  max_payload_size: 20971520
  withdrawal_type: "0x02"
  validator_balance: 2048
  electra_fork_epoch: 0
  fulu_fork_epoch: 18446744073709551615
  gloas_fork_epoch: 18446744073709551615

ethereum_genesis_generator_params:
  image: ethpandaops/ethereum-genesis-generator:daisugi-devnets
  extra_env:
    DAISUGI_FORK_EPOCH: "1"

additional_services:
  - dora
  - checkpointz

dora_params:
  image: ethpandaops/dora:daisugi-support-latest

checkpointz_params:
  image: ethpandaops/checkpointz:daisugi-support
```

---

Devnet repository: https://github.com/ethpandaops/daisugi-devnets
