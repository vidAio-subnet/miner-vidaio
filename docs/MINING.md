# MINING — how to mine on the VidAIO subnet (SN85)

How to run a miner against this stack, what you are scored on, and how to win
honestly. The quickstart and the architecture overview are in the root
[README.md](../README.md).

> **Status.** The subnet is LIVE on Bittensor mainnet, **netuid 85** — register
> and mine today. The production default is the real chain (`chain.mode:
> bittensor`), constructing the `BittensorChainAdapter`. Before launch, real-GPU
> miners earned on both inference tracks and participated in both earning
> competition tracks across separately advertised hosts on live testnet.
> `chain.mode: report` remains the default ONLY in test/dev/local overlays.

---

## What miners do

Remote GPU backends may opt into bounded software recovery using
`VIDAIO__MINER__REMOTE_GPU_ALLOW_CPU_FALLBACK=true` (default false).
The worker must bind the response to the same input, track and variant and
honestly report `cpu:ffmpeg-fallback` with GPU acceleration false. This does not
relax deadlines, output caps, signed ingress or quality scoring. Software
recovery uses the same CFR timestamp normalization as the scorer, not a
frame-preserving re-timing of VFR input. It can produce different quality/size
results; miners should test both
tracks before enabling it. Upstream errors are logged with bounded,
credential-redacted diagnostics.

Miners transform video. The subnet (or the organic gateway, for paying customers)
sends you a task; you return a processed file; the result is measured with real
metrics (ffmpeg/libvmaf) and folded into your standing.

**Scoring and weights are CENTRAL.** You are NOT scored separately by
each validator any more. An owner-run **Scoring Authority** dispatches the
challenges, runs the real measurement, folds the EWMA once (centrally), and each
epoch publishes one immutable, on-chain-anchored **epoch log** of the authoritative
scores. Validators then converge on that log (they submit the identical weight
vector) AND independently AUDIT it. So your score is measured once, honestly, and
is **independently recomputable and audited** — every scored item can be re-run
over the real engine from the preserved audit files, and a substituted score or
weight is provably caught (see the integrity invariant below). Nothing you do is judged by
a private per-validator sampling any more.

**Two pools, one pool per hotkey identity.** The subnet runs two tracks:

- **compression** — re-encode the input smaller (byte ratio must shrink ≥ 1.25×)
  while keeping VMAF against the pristine reference above the threshold;
- **upscaling** — upscale the degraded input (discrete factors 2× / 4×), scored
  on PieAPP quality + content length under per-factor file-size caps.

A miner identity competes in exactly ONE pool, declared by the **TaskWarrant**:
the validator probes `GET /warrant` and buckets every score for your hotkey
there. To earn in both pools you run two identities (two hotkeys, two
endpoints). A missing/garbage/timeout warrant answer means you are **skipped**
for the round — the validator never defaults you into a track
([`vidaio/validator/README.md`](../vidaio/validator/README.md), the TaskWarrant
fix). The reference miner declares its pool via `miner.warrant_track`
(`vidaio/miner/config.py`).

### How you earn (plain-language tokenomics)

Full engine: [`vidaio/tokenomics/README.md`](../vidaio/tokenomics/README.md);
levers and their locked values in [`config/default.yaml`](../config/default.yaml).

- Competition/crown is **live and earning**. Outside an active result window
  (**IDLE**), `80%` goes to inference and the fixed remaining `20%` goes to the canonical
  sink. A non-breakthrough result opens a seven-day **PODIUM** window (`60%` inference /
  `40%` competition); a breakthrough opens a seven-day **CROWN** window (`10%` inference /
  `90%` competition). The inference portion always keeps its internal **0.8 compression /
  0.2 upscaling** split; neither track inherits the other's unused allocation.
- Within a track, the **top 5** miners by accumulated score take a graded
  `5:4:3:2:1` rank curve; #1 earns five times #5, and rank 6+ takes nothing
  (`top_n_per_track: 5`). Scores below the absolute `minimum_payout_score: 0.10`
  earn zero even when fewer than five miners serve the track.
- Your standing is an **EWMA** of round scores: `new = 0.75·old + 0.25·score`
  (`ewma_decay: 0.75`). One great round doesn't crown you; one bad round
  doesn't bury you — but zeros compound.
- The **retention lever was removed for v1** (owner decision): there is no longer
  any multiplier for holding vs liquidating emitted alpha. The graded top-5 curve is
  based only on rank, with no retention reshaping. `burn_proportion` stays locked at 0.
- **Buying stake buys you nothing**: `alpha_stake_weigh_factor` is locked at 0
  and has no implementation as a weight term anywhere.
- Selection dedups by IP and coldkey (lowest uid wins) — running clones of one
  operation on one box or one coldkey does not multiply slots.
- Any fixed track/podium/crown share with no eligible recipient is sent to the
  canonical sink/burn UID. It is not renormalized into another miner's pool.

### The competition track (the breakthrough crown)

> **Live.** The shipping config enables competition emissions. Economic rank
> comes only from the arithmetic mean of the exact committed score packets, with a stable
> score/hotkey/uid tie-break. Stored human `final_rank`, manual disqualification,
> eligibility, and review preferences do not affect payout. A CPU-only auditor opens the
> corresponding audit bundles and independently rebuilds the result, crown, and weights.

Besides always-on inference mining, there are compression and upscaling
**competitions**:
sealed-sandbox code submissions evaluated against held-out content
([`vidaio/competition/README.md`](../vidaio/competition/README.md)). A contender
submits **pinned code identity** — `repo_url + commit_sha + tree_sha`
(`ContenderSpec`, `vidaio/competition/interfaces.py`) — that must build via its
Dockerfile into an image honoring the run contract:

```
/bin/sh /app/run.sh <input_dir> <output_dir>
```

one output per input under the same digest-named filename, plain regular files
only, exit 0. Evaluation runs in an isolated Docker sandbox: `--network none`,
read-only rootfs, no secrets, bounded output — your code sees inputs and writes
outputs, nothing else. Enrollment is phase- and deadline-gated with an
alpha-stake gate, driven through the orchestrator's token-authed control API
(`POST /competitions/{id}/contenders`); a public self-serve enrollment surface
is [NOT BUILT] — enrollment is operator-approved today. The enrolling hotkey
signs its own request (see the registered-hotkey auth requirement below), and
enrollment requests are coordinated through the VidAIO Discord — the invite is
in the root [README](../README.md).

Competition payouts use the latest globally applied result for seven days. A result
below the inclusive 5% breakthrough floor opens a **PODIUM** window: inference receives
60% and the competition podium receives 40%. A result at or above the floor opens a
**CROWN** window: inference receives 10% and the competition podium receives 90%.
Within either competition pool the first three places receive 70/20/10; a missing or
deregistered rank is sent to the canonical chain sink rather than redistributed. The
executable comparison floor is the archived baseline rerun on the same hidden matrix.
Every contender uses the same enrollment, scoring, audit, and payout path.
`tokenomics.competition_emissions_enabled` is retained only as an emergency off
switch; emissions-on is the normal state.

---

## Requirements

- **Hardware**: the reference backends are plain ffmpeg/x264 — any
  ffmpeg-capable CPU box works. A GPU is required only if YOUR approach needs
  one (learned restoration, super-resolution, etc.); nothing in the protocol
  assumes it. `ffmpeg` on `PATH` (the reference miner shells out to it;
  `miner.ffmpeg_path`).
- **Network**: your task endpoint must be reachable by validators (and the
  gateway, if you serve organic traffic). Production validators accept globally
  routable literal axon IPs; real `axon.port` is preferred over the configured fallback,
  and IPv6 literals are bracketed correctly.
- **Serving hotkey wallet**: artifact protocol v2 signs every response. The miner role
  loads its own hotkey-only wallet through `chain.*`; `chain.validator_hotkey` and
  `miner.artifact_hotkey` must both equal the registered hotkey advertised for this
  endpoint. Never put a validator or coldkey wallet in the miner service.
  Create a FRESH hotkey per mining identity (`btcli wallet new_hotkey`),
  register it on the subnet (`btcli subnet register --netuid 85`), and
  advertise your exact public endpoint (`scripts/advertise_miner.py` — the
  metagraph must read back the same literal IP and port you serve on). The
  coldkey stays offline; only the hotkey-half lives on the mining box.
- **Registered-hotkey auth (P2)**: competition enrollment is SELF-SIGNED — the
  enrolling hotkey signs its own enrollment request (Scheme A headers, see
  `vidaio.services.hotkey_auth.sign_request_headers`), must be registered on
  the subnet, and must clear the configured minimum alpha stake. An operator
  can no longer enroll a hotkey on its behalf; a signer may only enroll itself.
- **Disk**: task work dirs under `miner.work_dir`, swept after
  `task_dir_ttl_seconds` (900 s default). Inputs up to 2 GiB
  (`miner.max_input_bytes`).
- **Ingress clock**: the server allows 60 seconds by default (configurable only within
  `(0, 300]`) from admission through complete upload staging/fsync. Slow uploads receive
  HTTP 408 and their partial task directory/capacity slot are released.

### The wire contract you must serve

Authoritative source: [`vidaio/services/protocol.py`](../vidaio/services/protocol.py).
Two routes, on your API port (default **8300**):

**`POST /v1/task/artifact`** — bounded path-free byte exchange:

```jsonc
// canonical JSON before base64url encoding into X-Vidaio-Task-Metadata
{
  "task_id": "<validator-authored id>",   // echo it back EXACTLY
  "track": "compression",                  // or "upscaling"
  "input_digest": "<sha256 hex, 64 chars>",
  "params": {"...": "track params, e.g. upscale factor / bitrate cap"},
  "deadline_seconds": 300.0                // seconds before you are scored absent
}
```

The request body is the raw input stream (`application/octet-stream`, with an exact
`Content-Length` and a running byte counter). Metadata is capped at 4 KiB before encoding
so it fits normal proxy header limits. Production uses artifact version `2`: alongside
the metadata header it carries `X-Vidaio-Validator-Hotkey`,
`X-Vidaio-Miner-Hotkey`, request timestamp, 128-bit nonce, input size, and request
signature. The canonical signature binds all task metadata, input digest/size, both
identities, timestamp, and nonce. Before reading the body, the miner checks that the
request names itself, verifies the signature, refreshes chain state, and requires exactly
one current neuron with validator permit for the signer. A stale/duplicate nonce is
rejected. Both the global live replay cache and a smaller per-validator quota are hard
bounds; exhausting either returns 503 `artifact_auth_capacity` instead of evicting an
unexpired entry or allowing one validator to consume all capacity.
Every miner start also fences timestamps at
`start_time + artifact_request_future_skew_seconds`. A request at or below the fence is
rejected before body ingress with 425 `artifact_auth_starting`; retry after the bounded
roughly `future_skew + 1 second` startup blackout using a fresh timestamp and nonce. That
prevents replay of a still-fresh request captured before an in-memory cache restart.
The cache is process-local, so one miner hotkey must terminate at one ingress process.
Multiple replicas sharing a hotkey behind a load balancer are unsafe until a shared
replay store exists: the same captured request could land once on each replica.

The response body is the raw output stream. Headers carry version 2, exact task id,
output sha256, output size, processing seconds, miner hotkey, and response signature. The
signature binds the complete signed request plus output digest/size/processing time; the
validator verifies it against the chain-attributed miner hotkey before publishing the
download. Unsigned artifact v1 exists only behind
`miner.allow_unsigned_artifact_v1: true` in isolated report/tests and production forbids
that switch.

**`GET /warrant`** — `-> {"track": "compression" | "upscaling"}`, your pool
declaration. JSON absolute-path `POST /v1/task` and its pre-versioning `/task` alias are
DEPRECATED local-test compatibility routes. They are absent by default, require
`miner.enable_legacy_path_routes: true` in a test fixture, and that opt-in is rejected
in production.

Obligations that get you zeroed or skipped if violated:

- **Echo `task_id` exactly.** The validator authors it
  (`"{challenge_id}:{uid}"`); any other id in your response is zeroed
  (`task_id_mismatch`).
- **Digest discipline.** Verify `input_digest` over the bytes received. Return the real
  output sha256 header; the validator recomputes it while streaming, also verifies task
  id/size/version, and atomically publishes only a fully bound response.
- **Deadlines.** `deadline_seconds` (default 300 s) is your budget; the
  validator's own request timeout (`miner_request_timeout_seconds`, 300 s
  default) hard-bounds it. A timeout is a
  zero-scored empty response. It cannot enlarge the miner's independent
  `artifact_ingress_timeout_seconds` upload cap.
- **Capacity honesty.** Over capacity, answer `429 busy`
  (`miner.max_concurrent_tasks`) — the round scores you absent, which beats
  queueing unbounded ffmpeg jobs and timing out on everything.
- **Auth and public-edge protection.** `miner.api_token` / `X-Miner-Token` is for a
  controlled caller fleet. Do not share one secret subnet-wide with permissionless
  validators or miners. A permissionless miner leaves that optional extra bearer unset;
  protocol v2 authenticates current validators and miner responses by hotkey. Keep the
  endpoint behind a hardened reverse proxy/edge enforcing request/connection-rate,
  timeout, and network-abuse limits. Set the fleet-wide
  `validator.miner_url_scheme=https` only when every advertised literal IP presents a
  valid public certificate; otherwise the public hop is explicitly HTTP (artifact-v2
  still authenticates both sides). The miner's own signature/replay/body/concurrency
  caps remain mandatory.
- **Cleanup.** The canonical miner deletes its task directory after the output stream;
  TTL is crash recovery. The validator deletes its private download after scoring and
  archival on every success/failure/cancellation path.

---

## Running the reference miner

The public build runs the miner role against the real chain:

```bash
python scripts/service_entrypoint.py reference-miner
```

Registration, endpoint advertisement, and hardening are covered in the
[Mainnet](#mainnet) section below.

> **Chainless simulation.** The development tree carries a one-command local
> stack (a chain simulator plus the full scoring/audit topology) used to
> exercise the whole loop offline — challenge production → your encode → three
> libvmaf runs → central EWMA fold → epoch-log finalize+anchor → validator
> converge → auditor recompute. That tooling is a development tool and is not
> included in this repository; the public entrypoint refuses simulator modes
> and runs the miner only against the real chain.

Miner config keys (section `miner`, schema `vidaio/miner/config.py`, env
override `VIDAIO__MINER__<KEY>`):

| Key | Default | Meaning |
|---|---|---|
| `http_port` / `metrics_port` | 8300 / 9106 | API and health/metrics ports |
| `warrant_track` | `compression` | THE pool this identity competes in |
| `compress_crf` / `compress_preset` | 28 / `medium` | x264 levers, compression track |
| `upscale_crf` / `upscale_preset` | 16 / `medium` | x264 levers, upscaling track |
| `max_concurrent_tasks` | 2 | Past it: 429 `busy` |
| `max_input_bytes` | 2 GiB | Bigger remote artifact bodies refused (413) before work; deprecated local path uses 422 |
| `max_output_bytes` | 4 GiB | Output rejected before response streaming if it crosses the auditor-compatible bound |
| `artifact_ingress_timeout_seconds` | 60 | Server-owned admission→receive/staging/fsync cap; constrained to `(0,300]` |
| `artifact_hotkey` | `""` | Serving miner ss58; production requires it equal the loaded `chain.validator_hotkey` wallet |
| `allow_unsigned_artifact_v1` | `false` | Explicit report/test compatibility only; production forbids unsigned v1 |
| `artifact_request_max_age_seconds` / `artifact_request_future_skew_seconds` | 120 / 5 | Signed-request freshness window and tolerated positive clock skew |
| `artifact_replay_cache_entries` | 10000 | Bounded live `(validator hotkey, nonce)` replay entries; full cache fails closed |
| `artifact_replay_cache_entries_per_validator` | 256 | Per-validator live nonce quota; must be no larger than the global cache |
| `artifact_validator_snapshot_max_age_seconds` | 300 | Oldest metagraph snapshot accepted for current-validator authorization |
| `task_dir_ttl_seconds` | 900 | Output grace window before the reaper |
| `api_token` | `null` | Optional controlled-fleet `X-Miner-Token`; do not distribute one shared secret to permissionless subnet participants |

### How scoring works against you — what gets you zeroed

Scoring engine: [`vidaio/scoring/README.md`](../vidaio/scoring/README.md).
Validity gates run FIRST and are absolute — any failure forces score 0 with a
machine-readable reason code recorded in the audit packet:

- **The rate gate** (compression): byte ratio ≥ `0.80` →
  `COMPRESSION_RATE_TOO_HIGH`. You must shrink by at least 1.25×.
- **The VMAF floor/threshold** (compression): VMAF against the PRISTINE
  reference below `threshold − 5` → `VMAF_BELOW_FLOOR`; below the threshold →
  `VMAF_BELOW_THRESHOLD`. The production default threshold is 90. Note it is
  measured against the pristine original, not the degraded input you received —
  a miner that only
  re-encodes and never restores will honestly zero on heavily degraded draws.
- **The model-delta gate**: primary VMAF vs the NEG ("no enhancement gain")
  model delta > 3.0 → `VMAF_MODEL_DELTA_EXCEEDED`. Both anti-gaming model runs use
  the **miner input** as their reference, so the gate measures what the miner added
  rather than transformations already present in its challenge payload. The 3.0
  threshold is a calibrated constant, measured against the real degradation space.
- **Dedup**: byte-identical outputs across miners (exact verified SHA-256 digest)
  use the finalized challenge-anchor
  hash plus miner hotkey under `anchor_hash_hotkey/1`; arrival timing cannot choose
  the winner. A loser is zeroed `REPLAY_DUPLICATE` only with both signed receipts,
  outputs and the content-addressed duplicate witness available to auditors.
- **Non-finite / missing metrics**: fail closed — `METRIC_MISSING` /
  `METRIC_NON_FINITE`, never a silent pass.
- **Stream validity**: frame count, duration, dimensions, PTS consistency
  (`FRAME_COUNT_MISMATCH`, `STREAM_*` codes); upscaling adds per-factor
  file-size caps (`FILE_SIZE_CAP_EXCEEDED`) and `UNSUPPORTED_SCALE_FACTOR`.
- **Perceptual-manipulation gates** (tone/grayscale/chroma:
  `TONE_MANIPULATION`, `COLOR_GRAYSCALE`, `CHROMA_UV_MANIPULATION`): deterministic
  CPU/OpenCV checks that are required in production and local release testing.
- Miner-attributable timeout, transport/429, protocol, task-id, output-digest and
  receipt failures fold an explicit zero only when the authority persists a canonical
  observation binding its signed artifact-v2 request, finalized anchor, target
  hotkey/endpoint and deadline. This stops selective non-response from freezing EWMA.
  The negative network fact remains a validator observation (not a Byzantine proof),
  but it is immutable and publicly disputable. A miner's authenticated restart fence
  that consumes the whole signed request deadline is miner-attributable and also folds
  zero. Validator-side faults—scoring worker, audit store, chain/challenge service, or
  local input—remain excused.

The upscaling track uses PIQ PieAPP. The release/auditor path runs it on CPU with
digest-pinned, preloaded weights. Miners are free to run whatever GPU stack their
approach needs; the launch scorer and every auditor remain CPU-only. CUDA scorer
mode is development-only
until a fresh Modal scorer proves exact CPU packet parity and the production guard is
changed under a versioned release.

### Reading your scores

Your weight share IS your standing after EWMA + rank curve — on mainnet, read
it straight from the metagraph (`btcli`, or any metagraph explorer). Per-round
detail (gate failures with reason codes, VMAF/rate numbers) is in the
validators' structured JSON logs and in the archived ItemScore packets, which
any third party can open and recompute from the audit store
([`vidaio/audit/README.md`](../vidaio/audit/README.md)).

---

## How to beat the reference miner honestly

The reference miner is a floor, not a bar: it re-encodes (CRF, in whichever of
h264/hevc/av1/vp9 the task asks for — x264 when it asks for nothing) and does
not restore. Measured on the dev content, its binding constraint is VMAF
against the pristine reference — it wins only the draws whose degradation a
re-encode can survive. Beat it by:

- **Better encoding**: smarter codec/preset/CRF choices per content, two-pass
  or content-adaptive rate control, better codecs where the encoding gate
  allows — every byte saved under the rate gate converts to score
  (`comp = 1 − rate`, weighted 0.7 in the compression formula).
- **Actual restoration**: deblur, denoise, tone/exposure correction, artifact
  repair before re-encoding. The challenge DAG degrades footage the way real
  pipelines do (capture → edit → delivery,
  [`vidaio/challenge/README.md`](../vidaio/challenge/README.md)); a miner that
  genuinely inverts those operators clears VMAF draws the reference miner
  honestly zeros on. This is the whole point of the subnet.
- **Capacity + reliability**: no timeouts, no 429s at your typical load, exact
  digests — absent rounds decay your EWMA.

What the gates make pointless:

- **Metric gaming** — sharpening/contrast tricks that inflate default VMAF are
  precisely what the NEG-model delta gate hunts; tone/color/chroma manipulation
  has dedicated deterministic CPU gates.
- **Replay/collusion** — cross-miner dedup zeroes copies; the task id is
  validator-authored and bound; the scored bytes are digest-pinned; the PieAPP
  sample frame derives from `sha256(content_digest || challenge_id)` — nothing
  is predictable before dispatch.
- **Enhancement shortcuts on upscaling** — file-size caps and the pass/fail
  VMAF gate bound the space; PieAPP quality is what scores.
- **Seed/DAG probing** — challenge parameters are private, committed
  (commit-reveal) BEFORE dispatch, and dispatch payloads are structurally
  leak-probed.

**The integrity invariant**: every score is an exact, archived ItemScore packet —
metrics, gate outcomes, canonicalization plan digest, scorer identity — and each
epoch's weight vector is derived from an immutable **epoch log** that merkle-commits
to those packets (root + per-item inclusion proofs) and is **anchored on chain**.
Any third party can recompute any score from the audit store
([`vidaio/audit/README.md`](../vidaio/audit/README.md)), and the validators
themselves do exactly that as **auditors**: each independently recomputes a sample
over the real engine, RE-FOLDS the earning state, re-derives the weight vector, and
POSTs a signed verdict to the **Audit Results API**
([`vidaio/auditor/README.md`](../vidaio/auditor/README.md),
[`vidaio/audit_api/README.md`](../vidaio/audit_api/README.md)). A substituted score
is `SCORE_MISMATCH`, a substituted weight is `WEIGHT_DERIVATION_MISMATCH`, a
substituted earning state is `EARNING_STATE_MISMATCH`, and a caught substitution
surfaces publicly as a **DISPUTED** epoch. There is no substitution path in this
codebase, for you or against you; and the central scorer cannot cherry-pick
corruptions after seeing your output, because the challenge commitment binds seed +
DAG + asset + scorer before dispatch and the sampling is seeded by the finalized
hash of the fixed future block `close_block + K`, which it cannot predict while
building and anchoring the log.

---

## Mainnet

The subnet is LIVE on mainnet **netuid 85** — register and mine today.
`chain.mode: bittensor` is the production default and constructs the real
`BittensorChainAdapter` (lazily importing the optional `.[chain]` bittensor deps —
a missing dep fails fast with `NotConfiguredError` pointing at the extra). The
real-chain path was exercised end to end on live testnet before launch — the
substrate/archive path, both inference tracks, and both earning competition
tracks. What follows is the miner-side operating contract.

- **Registration is standard btcli, operator-side**: `btcli subnet register`
  with your hotkey on netuid 85 (`core.netuid`) — no code path registers for
  you, and coldkeys are never touched at runtime.
- **Run the public miner least-privileged** from the release image with only its
  miner configuration, its own hotkey-only wallet, and media runtime; it does not need a
  validator wallet or S3 secrets:

  ```sh
  python scripts/service_entrypoint.py reference-miner
  ```

  Set `miner.artifact_hotkey` equal to this role's `chain.validator_hotkey` and loaded
  miner wallet. Keep `miner.allow_unsigned_artifact_v1: false` and
  `miner.enable_legacy_path_routes: false`; production rejects both compatibility
  opt-ins. Put permissionless endpoints behind the hardened edge described above.
- **Advertise the HTTP(S) endpoint after registration and startup.** Configure
  the same miner-specific `chain.validator_hotkey`/wallet environment used by the
  serving process, then publish and verify its globally routable literal address with
  the helper:

  ```sh
  python scripts/advertise_miner.py --config config/default.yaml \
    --external-ip <GLOBAL_IP> --external-port 8300 --external-scheme https
  ```

  This submits the Bittensor `serve_axon` extrinsic only for metagraph discovery;
  the miner continues serving the streamed HTTP(S) protocol, not a Synapse/Axon
  application server. Axon data carries no scheme, so the argument must match every
  validator's `miner_url_scheme`. The helper waits for finalization and exact IP/port
  readback.
- **Endpoint discovery**: the adapter reads the metagraph axon IP and port. Production
  dials only globally routable literal IPs; unspecified addresses are not dialed (and are
  separately exempt from reward IP dedup). The real axon port is preserved/preferred,
  with `validator.miner_port` only a fallback. Validate serving and reachability
  before you expect scored rounds.
- **Same signed wire contract**: artifact-v2 `POST /v1/task/artifact` + `GET /warrant`.
  A controlled reference fleet may additionally set `miner.api_token`; permissionless
  miners leave the global shared token unset and use hotkey auth plus the hardened public
  edge described above. No validator/miner shared filesystem is required.
- **Deregistration/churn**: the designed reconciliation keys earnings by
  hotkey, resolves hotkey→uid fresh at weight time; a hotkey swap purges your
  EWMA history (you restart as a new miner).

The same code path serves testnet and mainnet — only the `chain.*` endpoint and
netuid differ — so you can rehearse your full setup (registration, advertisement,
serving, scored rounds) on testnet before registering the mainnet hotkey.
