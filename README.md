# miner-vidaio

Mining code for the **VidAIO subnet** (Bittensor **SN85**) — AI video
upscaling and compression. Miners serve real video-processing work; validators
measure the results (VMAF and perceptual metrics, CPU-recomputable by anyone)
and the best encoders earn the emissions. This repository is everything you
need to run one: the reference miner, the wire protocol, and the registration
tooling.

> **Release mirror.** Each VidAIO release lands here as one snapshot commit.
> Deep documentation lives in [docs/MINING.md](docs/MINING.md) — this README
> is the quickstart and the troubleshooting desk.

---

## 1. What you are signing up for

- Two scored tracks: **compression** (smallest file that holds quality) and
  **upscaling** (best restoration of degraded video). You pick a track per
  serving hotkey and advertise one endpoint for it.
- Scoring is **objective and auditable**: VMAF + perceptual checks, computed
  on CPU, re-computable by independent auditors from published evidence. There
  is no reviewer to impress — only encoders to beat.
- The **reference miner in this repo earns** out of the box (ffmpeg-based
  `quality` / `balanced` / `compact` variants). Beating it honestly — better
  encoders, better restoration models — is the whole game:
  [docs/MINING.md §How to beat the reference miner honestly](docs/MINING.md).

## 2. Requirements

| Piece | Minimum |
|---|---|
| OS | Linux x86-64 (Ubuntu 22.04/24.04 tested) |
| CPU box | any ffmpeg-capable machine for the reference miner |
| GPU | only if YOUR approach needs one (e.g. learned upscaling) |
| Network | a **publicly reachable** endpoint — global literal IP, open port, TLS |
| Chain | a Bittensor wallet + TAO for registration on netuid **85** |
| Software | Python 3.12+, `ffmpeg` on PATH, Docker (recommended), `btcli` |

## 3. Create the wallet (fresh keys, coldkey stays offline)

```sh
pip install bittensor-cli
btcli wallet new_coldkey --wallet.name vidaio            # do this on a SECURE machine
btcli wallet new_hotkey  --wallet.name vidaio --wallet.hotkey miner1
```

Rules that protect you:
- **One fresh hotkey per mining identity.** Never reuse a hotkey across
  subnets or machines.
- **The coldkey never touches the mining box.** Only the hotkey half is
  needed to serve and sign responses.
- Back up both mnemonics offline. A lost hotkey = a lost registration.

## 4. Register and advertise

```sh
# Register the hotkey on SN85 (costs the current registration burn):
btcli subnet register --netuid 85 --wallet.name vidaio --wallet.hotkey miner1

# Install this repo (the chain extra carries the real Bittensor adapter),
# then advertise your exact public endpoint on-chain:
pip install -e '.[chain]'
python scripts/advertise_miner.py --help
```

Advertisement is **exact-readback**: the metagraph must show the same literal
IP and port you actually serve on. Validators only dispatch to what the chain
advertises — a NAT'd, wrong, or stale advertisement earns nothing.

## 5. Run the miner

Configuration is `config/default.yaml` + `VIDAIO__*` environment overrides
(every field is documented in `vidaio/miner/config.py`). The essentials:

```sh
export VIDAIO__CHAIN__NETWORK=finney          # or 'test' for testnet first
export VIDAIO__CHAIN__NETUID=85
export VIDAIO__CHAIN__VALIDATOR_HOTKEY=<your miner ss58>
export VIDAIO__MINER__ARTIFACT_HOTKEY=<the same ss58>
export VIDAIO_HOTKEY_SEED=<hotkey seed — mode-0600 file / secret store, never argv>

python scripts/service_entrypoint.py reference-miner
```

Serve it behind TLS (validators require HTTPS). The pattern that works: an
nginx edge terminating a Let's Encrypt IP certificate in front of the miner
port, with the renewal deploy-hook **restarting** the edge container (a plain
nginx reload has been observed to keep serving an expired certificate — see
`docs/MINING.md`).

Strongly recommended first: **run on testnet** (`--network test`, the test
netuid) until your endpoint scores rounds cleanly, then register on mainnet.

## 6. Verify you are actually earning

- `GET https://<your-ip>:<port>/health` answers from the internet.
- Your hotkey appears in the metagraph with your exact IP/port.
- Within a few epochs your uid shows `incentive > 0` on the subnet
  leaderboard / `btcli subnet metagraph --netuid 85`.
- Logs show artifact-v2 requests arriving and signed responses leaving.

## 7. Troubleshooting

| Symptom | Likely cause → fix |
|---|---|
| No requests ever arrive | Endpoint not reachable or not advertised: test `curl https://<ip>:<port>/health` from OUTSIDE; re-run `advertise_miner.py` and confirm exact readback |
| TLS handshake failures in validator logs | Expired/self-signed cert: renew, and make the renewal hook RESTART your TLS edge, not reload it |
| `401`/`403` on requests you send | Your hotkey is not registered / lost registration (deregistered by churn) — check the metagraph; re-register |
| Signature refused (`replay`, `timestamp`) | Clock skew: NTP-sync the box (the signed-request window is ±120 s); never resend the same nonce |
| Scored `0` with successful responses | Output failed verification (wrong codec/dims, quality below threshold) — reproduce locally with the reference scorer, check `docs/MINING.md` scoring section |
| `pool_exhausted` / no work on testnet | The challenge pool is momentarily empty — normal on testnet; rounds resume when content is ingested |
| Registered but incentive stays 0 for days | You are being outscored: your accumulated EWMA is below the emission floor — improve the encoder or switch variant/track |
| `premium` variant errors | Expected: `premium` is not part of the public build; run `quality`, `balanced`, or `compact` |

## 8. Upgrades

Watch this repository's releases: each release is one snapshot commit and the
`VERSION` file is the fleet's compatibility fence. **Epoch-schema releases are
lockstep** — the release notes will say when validators and miners must move
together. Key your own update automation off `VERSION` if you want unattended
upgrades; the `vidaio/autoupdater` package documents the update contract.

## 9. Getting help

- **Discord** — the VidAIO server (invite via [vidaio.io](https://vidaio.io)):
  ask mining questions in the miners channel; the core team reads it daily.
- **GitHub issues on this repo** — bugs, docs gaps, reproducible scoring
  disputes (include your uid, hotkey, epoch and logs).
- Before asking: grab the exact log lines and your uid/hotkey — every scored
  round is auditable, so with those we can tell you precisely what happened.

## License

MIT — see [LICENSE](LICENSE).
