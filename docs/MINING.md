# MINING — how to mine on the VidAIO subnet (SN85)

Mining on VidAIO (Bittensor **netuid 85**, live on mainnet) means entering
**competitions**. You build a solution as a containerised entry in a Git repository,
enroll it with a request signed by your registered hotkey before the enrollment
deadline, and the validator builds it and runs it on a hidden set of clips in a sealed
GPU sandbox. Every output is scored by the public scorer, the result is published and
independently audited, and miner emissions are paid to the result. There is no
live-serving mining any more: you do not run a miner server, advertise an endpoint or
answer validator requests.

This guide is the full rulebook. The quickstart is the root [README.md](../README.md).

## Contents

- [How competitions work](#how-competitions-work)
- [How you earn](#how-you-earn)
- [Before you start: wallet, registration, stake](#before-you-start-wallet-registration-stake)
- [Entering a competition](#entering-a-competition)
- [Building and testing an entry](#building-and-testing-an-entry)
- [Sandbox and limits](#sandbox-and-limits)
- [Checklist: what loses you a clip, a batch or the whole entry](#checklist-what-loses-you-a-clip-a-batch-or-the-whole-entry)
- [What your code may do](#what-your-code-may-do)
- [Tracks](#tracks): [compression](#compression-track-compression),
  [object removal](#object-removal-track-removal),
  [frame interpolation](#frame-interpolation-track-interpolation),
  [streaming compression](#streaming-compression-track-compression-streaming-scoring),
  [faithful upscaling](#faithful-upscaling-track-upscaling-faithful-scoring),
  [generative upscaling](#generative-upscaling-track-upscaling-generative-scoring)
- [Speed](#speed)
- [Licences](#licences) and the [licence rulings so far](#licence-rulings-so-far)
- [Result rules](#result-rules)
- [Results and what is published](#results-and-what-is-published)

---

## How competitions work

VidAIO runs compression, upscaling, object-removal and frame-interpolation
**competitions**: sealed-sandbox code submissions evaluated against held-out content
([`vidaio/competition/README.md`](../vidaio/competition/README.md)). A contender
submits **pinned code identity** — `repo_url + commit_sha + tree_sha`
(`ContenderSpec`, `vidaio/competition/interfaces.py`) — that must build via its
Dockerfile into an image honoring the run contract:

```
/bin/sh /app/run.sh <input_dir> <output_dir>
```

one output per input under the same digest-named filename, plain regular files
only, exit 0. Evaluation runs in an isolated sandbox: no network, no secrets, no
environment variables, bounded output — your code sees inputs and writes outputs,
nothing else (details in [Sandbox and limits](#sandbox-and-limits)). Builds do have
network access; runs never do.

**What is fixed before you enroll.** Every competition is described by a manifest whose
digest is anchored on chain *before* enrollment opens, and the manifest is public. It
carries: the four lifecycle times, the enrollment stake floor, the quality gate
(`vmaf_threshold`), the compute envelope every contender gets
(`sandbox_resources`: CPUs, memory, and `batch_timeout_seconds`, plus `allowed_gpus`;
see [Sandbox and limits](#sandbox-and-limits)), the result rules
([`result_rules`](#result-rules)), the archived baseline you are compared against (a
competition may have none: then only its absolute score bars decide), and the ordered
**commitments to the hidden clips** (`evaluation_item_commitments`).
The clips themselves stay sealed until evaluation; the commitments let anyone prove
afterwards exactly which clips were used and that none was swapped.

**Score.** In a compression competition each clip is scored with the public compression
formula (`min(1, (0.7·(1 − rate) + 0.3·VMAF/100) / 1.12)`, zero below the VMAF gate or
on a frame-count/geometry mismatch); the other tracks have their own item scores
([Tracks](#tracks)). Your competition score is the **plain mean of your per-clip
scores** (times the speed factor when the manifest has a [speed term](#speed)); ranking
is by that mean with a stable hotkey/uid tie-break. Human review can flag a contender
for investigation but never changes a payout.

**Audited.** The shipping config enables competition emissions. Economic rank comes only
from the arithmetic mean of the exact committed score packets, with a stable
score/hotkey/uid tie-break. Stored human `final_rank`, manual disqualification,
eligibility, and review preferences do not affect payout. A CPU-only auditor opens the
corresponding audit bundles and independently rebuilds the result, crown, and weights.

## How you earn

The payout code is `vidaio/tokenomics/weights.py` (`EMISSION_PROFILES` holds the v3 and
v4 shares and splits); the defaults are in [`config/default.yaml`](../config/default.yaml),
and every competition anchors its own values in its manifest (`result_rules`).

Competition payouts use the latest globally applied result for seven days (a newer
result replaces it). Since tokenomics v3 (2026-09-27) a result pays **100 % of miner
emissions** by default: a result that does not meet the crown rule
([Result rules](#result-rules)) opens a **PODIUM** window whose pot is split
50/24/13/8/5 over the top five qualifying places; a result that meets it opens a
**CROWN** window split 90/4/3/2/1. Unfilled places go to the filled ones in proportion;
a result with no qualifying contender closes the window and the subnet burns until the
next result; a deregistered paid hotkey sends its share to the sink, and an entry whose
hotkey is not registered when the result is applied is left out of the ranking.

**Tokenomics v4 (from the frame-interpolation competition on).** Competitions run every
two weeks and a result pays for **two weeks** (336 hours), so each result covers the time
until the next one. A gap with no active result no longer burns everything: **50 % burns
and 50 % goes to the licensing treasury**, a registered hotkey that pays licensing fees to
winners whose solutions run on the VidAIO website. While a window is active the treasury
keeps **0.5 %** so it stays registered, and the window's pot and split apply to the other
99.5 % (a 90/4/3/2/1 crown pays 89.55/3.98/2.985/1.99/0.995 % of emissions). Every epoch
log names the treasury hotkey it pays (`treasury_hotkey`), and auditors check it against
their own policy; an unregistered treasury's share burns. Every competition anchors these values in its
manifest (`result_rules`), so check the manifest of the competition you enter: the pot
and the split may differ from the defaults. When a competition has a baseline, the
executable comparison floor is that archived baseline rerun on the same hidden matrix.
Every contender uses the same enrollment, scoring, audit, and payout path.
`tokenomics.competition_emissions_enabled` is retained only as an emergency off
switch; emissions-on is the normal state.

Stake never increases the size of a reward (`alpha_stake_weigh_factor` is locked at 0):
it is only the enrollment floor below. A competition pays each operator once
([One entry per operator](#one-entry-per-operator)).

## Before you start: wallet, registration, stake

- **Wallet.** Create a coldkey on a secure machine and a fresh hotkey for mining
  (`btcli wallet new_coldkey`, `btcli wallet new_hotkey`). Enrollment is signed with the
  **hotkey** only; your coldkey is never used.
- **Registration is standard btcli, operator-side**: `btcli subnet register` with your
  hotkey on netuid 85 — no code path registers for you. The hotkey must be registered
  when you enroll and still be registered when the result is applied.
- **Stake.** The signing hotkey must hold the competition's alpha-stake floor (see
  [Stake and registration](#stake-and-registration)); `list` prints it.
- **Self-signed enrollment.** The enrolling hotkey signs its own enrollment request
  (Scheme A headers, see `vidaio.services.hotkey_auth.sign_request_headers`), must be
  registered on the subnet, and must clear the configured minimum alpha stake. An
  operator cannot enroll a hotkey on its behalf; a signer may only enroll itself.
- **Client.** The enrollment client runs from this repository and needs `bittensor`:
  `pip install -e '.[chain]'`.

## Entering a competition

### Enrolling

Enrollment is self-serve and signed with your hotkey. The public
competitions address is `https://95.133.254.122/competitions`; if it ever moves, the
competition hub ([subnet.vidaio.io/competitions](https://subnet.vidaio.io/competitions)) and the
announcement in the VidAIO Discord (invite in the root [README](../README.md)) give the
current one. Run the client from the public miner repository (it needs `bittensor`)
inside your solution's clone, checked out at the commit you want evaluated:

```
cd /path/to/your-solution        # your clone, at the commit to evaluate, already pushed
python /path/to/miner-vidaio/scripts/competition_enroll.py list \
    --url https://95.133.254.122/competitions
python /path/to/miner-vidaio/scripts/competition_enroll.py enroll \
    --url https://95.133.254.122/competitions \
    --competition-id <competition id from list> \
    --repo-url https://github.com/<you>/<repo>.git \
    --wallet-name <coldkey name> --wallet-hotkey <hotkey name>
```

`--repo` defaults to the current directory, `--commit` to `HEAD` and `--tree` to
`HEAD^{tree}`; pass them to enroll something else. Before anything is signed, the client
checks that the tree belongs to the commit, that the commit is pushed, that a `Dockerfile`
is at the repository root (and `LICENSE` and `LICENSES.md` when the competition requires
them: `list` shows it), that the tree has no symlinks or submodules, and that it fits in
512 MiB. `--skip-local-checks` skips these checks; it
changes nothing about what the evaluator enforces.

The request body is only `{repo_url, commit_sha, tree_sha}` (plus `replace`) and is signed with your
**hotkey** (your coldkey is never used). The validator verifies the signature and that
the hotkey is registered on the subnet, and reads its alpha stake from the chain, never
from your request. One entry per hotkey per competition. `list` shows every manifest,
stake floor and allowed repository host; the schedule of each competition is on the hub
and in the Discord announcement. Git LFS and submodules are not supported.

### Stake and registration

The floor is the larger of the manifest's
`minimum_alpha_stake` and the validator's `hotkey_auth.min_enroll_alpha_stake`; `list`
prints it (500 alpha in every competition so far). What counts is the alpha staked on the
signing hotkey on this subnet, from any coldkey, as the chain registry reports it. It is
checked when you enroll, again on every `--replace`, and **once more when entries are
finalized** (the manifest's `finalization_time`, 24 h after the enrollment deadline in every
competition so far): an entry whose hotkey holds less than the floor at that moment is
rejected (from competition 006 on; until 2026-11-16 stake was checked only at enrollment).
Keep the stake in place until finalization. While enrollment is open, `status` and the hub
warn when your current stake is below the floor, so you can top it up in time. After
finalization, stake is not checked again. The
hotkey must still be registered when the result is applied, or the entry is left out of
the ranking. Competitions need no server, advertised IP or running miner: enrollment and
evaluation never connect to any machine of yours. If the hotkey does advertise an IP on
chain, that IP counts for [One entry per operator](#one-entry-per-operator).

### Changing or withdrawing your entry

Until the enrollment deadline:

```
# point your entry at a new commit: same entry, as often as you like, the last one counts
python /path/to/miner-vidaio/scripts/competition_enroll.py enroll --replace \
    --url https://95.133.254.122/competitions --competition-id <id> \
    --repo-url https://github.com/<you>/<repo>.git \
    --wallet-name <coldkey name> --wallet-hotkey <hotkey name>

# withdraw: final, this hotkey cannot enter this competition again
python /path/to/miner-vidaio/scripts/competition_enroll.py withdraw --yes \
    --url https://95.133.254.122/competitions --competition-id <id> \
    --wallet-name <coldkey name> --wallet-hotkey <hotkey name>

# your own entry in full (signed), or anyone's public summary
python /path/to/miner-vidaio/scripts/competition_enroll.py status \
    --url https://95.133.254.122/competitions --competition-id <id> \
    --wallet-name <coldkey name> --wallet-hotkey <hotkey name>
python /path/to/miner-vidaio/scripts/competition_enroll.py status \
    --url https://95.133.254.122/competitions --competition-id <id> --hotkey <ss58>
```

`--replace` swaps the repository, commit and tree of the same entry in place, after the
same local checks and the same stake check as `enroll`; `enroll` without it is refused
once the hotkey has an entry. A withdrawn entry shows as REJECTED with the reason
"withdrawn by the miner". After the deadline nothing can be changed: the evaluator fetches
exactly the commit and tree your entry holds at that moment, and later pushes change
nothing. Do not enroll a second hotkey to get a new version in: it is a second entry of
the same operator (below).

### Checks before the deadline

While enrollment is open the evaluator fetches every entry
version the way production does, with its own reader credentials, and checks it: the
repository is readable, the commit and tree match, the tree is safe (no symlinks, special
files or submodules) and within the size limit, a `Dockerfile` is at the root (and
`LICENSE` and `LICENSES.md` when the competition requires them), and your hotkey still
holds the stake floor. It checks within minutes of an `enroll` or `--replace`, then
every 6 hours, and every hour in the last 24 hours before the deadline. When the operator
has turned it on, it also runs a trial build of your image on the real builder (no
evaluation data, no GPU). Results show in `status` (with the full error text for your own
entry) and on the hub. These checks only **warn**; they never reject an entry. The rules
that lose an entry after the deadline ([checklist](#checklist-what-loses-you-a-clip-a-batch-or-the-whole-entry))
are unchanged, so act on a warning before the deadline: fix it, push, and `--replace`.

### Which version is enrolled

The public competition view (the hub, and
`GET https://95.133.254.122/competitions/v1/competitions/<id>`) lists each entry's
`entry_commitment`: the sha256 of `competition_id`, hotkey, `repo_url`, `commit_sha` and
`tree_sha`, joined by newlines with no trailing newline. Check yours:

```
printf '%s\n%s\n%s\n%s\n%s' <competition id> <hotkey ss58> <repo url> <commit sha> <tree sha> | sha256sum
```

The same view shows `paused` while an operator has paused the competition.

### One entry per operator

A competition pays each operator once. Entries that share a
coldkey, an advertised IP, a repository account (the `<owner>` in
`https://github.com/<owner>/<repo>`, whatever the repository), a source tree (the enrolled
`tree_sha`, so forks of one solution) or a byte-identical output matrix count as one
operator: only the lowest uid among them can hold a ranked, paid place, and the others
are excluded from the payout. An entry built from the same source as an entry with a
lower uid is taken out with that reason, and the lower uid is kept. Near-duplicates (one
solution with small changes, entered from separate keys, accounts or trees) are reviewed
by an operator and treated the same way. Teams may share ideas and work together, but a
team enters its solution once and splits the reward itself. Extra hotkeys never add
places: to change your entry, `--replace` it.

### Use a private repository

Enrollment names a repository, a commit and a tree; it does
not prove who wrote the code. Anything in a public repository can be copied, or even
enrolled as-is by someone else, before the deadline. To let the evaluator read a private
GitHub repository, add the machine user **`sn85competitionreader-collab`** under
Settings → Collaborators. It is an ordinary account, not a GitHub App, and invitations
are accepted automatically within a few minutes. GitHub only offers write access for
collaborators on personal repositories; the reader never pushes, it only fetches: during
enrollment for the checks above, and after the deadline the exact commit your entry holds
(once for the sealed archive, once for the build). If you want strict read-only access,
keep the repository in a free GitHub organization and add the reader as an outside
collaborator with the Read role. Invite it before you enroll, so the first check can read
the repository, and keep it invited until the builds are announced as done; an entry
whose repository cannot be fetched cannot be evaluated. Do not force-push over, rename or
delete the enrolled commit before then.

## Building and testing an entry

### Start from an example

`examples/competition_contenders/` ships a starting point for every track (the
[track sections](#tracks) say what each one does):

| Track | Start from |
|---|---|
| compression | `cpu_compression/` (x264, x265, VP9, SVT-AV1 fixed and a quality-searching SVT-AV1) with `materialize_cpu.py`; the GPU `template/` with `materialize.py` |
| streaming compression | `streaming_example/` |
| faithful upscaling | `upscaling_example/` |
| generative upscaling | `generative_upscaling_example/` |
| object removal | `removal_example/` |
| frame interpolation | `interpolation_example/` |

`removal_example/`, `interpolation_example/`, `streaming_example/` and
`upscaling_example/` carry a `test_local.py` self-test that needs no Docker. The header
of `cpu_compression/run.sh` states the full run contract; you can run it
locally against any folder of sha256-named clips before you enroll. Before you pin a
commit, `python scripts/competition_precheck.py <repo> --commit <sha>` checks the exact
commit against the entry-losing rules of the [checklist](#checklist-what-loses-you-a-clip-a-batch-or-the-whole-entry)
(add `--commercial` where the [licence rules](#licences) apply).

### Practice clips and past data

Every competition comes with a public practice pack
(clips in the format of its hidden items; recent packs add reference scores and
`score_practice.py`, which runs the validator's scorer), attached to a
`practice-pack-<NNN>` release on the
[releases page of the public miner repository](https://github.com/vidAio-subnet/miner-vidaio/releases).
A pack is similar in kind to its competition's content but shares no clip with any hidden
set. Scored data is public as well, with each input, output, released source and full
score packet, in the evidence bucket `https://s3.hippius.com/vidaio-sn85-evidence`
(anonymous read, no listing; on Hippius since 2026-10-05, before that on AWS). Start from
`finalized/epoch=<N>/log.json` (first epoch 24920; epoch `N` closes at block
`8994622 + 360 * (N - 24920)`), follow `audit_manifest.per_uid` to each
`audit_bundle/<aa>/<bb>/<digest>`, and the bundle names the `challenge_input`,
`miner_output`, `released/reference_original` and `score_packet` objects under the same
`<kind>/<aa>/<bb>/<digest>` layout.

## Sandbox and limits

What the code guarantees every contender of a competition, and
the field that sets each value:
- *CPU and memory:* `sandbox_resources.cpu` CPU cores and `sandbox_resources.memory_mb` MiB
  of RAM, reserved for your sandbox and capped at the same values. A `run.sh` killed for
  running out of memory is scored as a non-zero exit (below).
- *GPU:* one GPU per sandbox, of the type the manifest's `allowed_gpus` names (Modal's
  names, for example `L4`, `L40S`, `A100-80GB`). A manifest with a speed term names exactly
  one type, so every contender runs on the same one. Builds get no GPU.
- *Batches:* one `run.sh` call receives up to `evaluation_batch_size.max` clips and must
  exit within `sandbox_resources.batch_timeout_seconds` of wall-clock time. The clock starts
  when `run.sh` starts inside an already running sandbox and stops when it exits: image pull
  and sandbox start-up are not counted; model loading and everything else `run.sh` does is.
  A timeout or a non-zero exit scores the whole batch zero. An operator can only raise
  these values, for everyone, by a public execution amendment.
- *Fresh sandbox:* every batch runs in a new sandbox started from your image. Nothing you
  write survives to the next batch and nothing is warm: caches, compiled kernels and loaded
  models start cold in every batch.
- *Network and environment:* no network at run time (builds have network); no environment
  variables and no secrets; your `ENTRYPOINT`/`CMD` are ignored.
- *Files:* the inputs sit alone in their folder; treat it as read-only. Write outputs only
  into the output folder: at most 512 MiB per file and 2 GiB in total at any moment, at most
  4096 entries (validator settings `orchestrator.sandbox_output_max_bytes`,
  `sandbox_batch_output_max_bytes`, `modal_max_output_entries`), and at most 8 MiB of
  stdout plus stderr (`sandbox_log_max_bytes`). Scratch goes to `/tmp`, which is local to
  your sandbox and empty at the start of every batch.
- *Build:* the image must build within `orchestrator.build_timeout_seconds` (1800 s) and be
  at most the manifest's `container_size_limit_gb`. A build gets `build_retry_attempts` (2)
  tries; only a failure of your own Dockerfile or context marks the entry BUILD_FAILED, and
  an infrastructure failure halts the competition instead.
- *Checkout:* the files of the enrolled commit, without `.git`, take at most 512 MiB
  (`orchestrator.git_checkout_max_bytes`).

Not pinned by the code, so detect it at run time instead of assuming it: the CPU model and
generation (each batch runs on whichever host is free, so it can change from batch to
batch) and instruction sets such as AVX-512; how many CPUs the OS reports (read `nproc` or
`os.sched_getaffinity` and set thread counts explicitly); the NVIDIA driver and the highest
CUDA version it supports (check `nvidia-smi` or `torch.cuda.is_available()` at start, and
fall back loudly); the size of `/tmp` and `/dev/shm`, and whether `/tmp` counts against the
memory limit (plan as if it does); the process and thread limit. If an operator has to move
a running competition to the fallback GPU host (`remote_docker`, listed under
`amendments`), runs get the Docker runner's limits: a read-only root filesystem, `/tmp` as
a tmpfs of `orchestrator.sandbox_tmpfs_size` (256 MB by default) and at most
`orchestrator.sandbox_pids_limit` processes (256 by default), the same as the local Docker
runner. An entry that keeps its scratch small runs on both.

## Checklist: what loses you a clip, a batch or the whole entry

Every limit below is
enforced by code and the same for everyone. Nothing can be changed after the enrollment
deadline; before it, fix the entry and `--replace` it.

*The whole entry is lost when:*
- the repository cannot be fetched after the deadline (reader not invited, repository deleted
  or renamed, commit not pushed or force-pushed away), or the fetched commit/tree does not
  match what your entry holds;
- the tree contains `.gitmodules`, symlinks or special files, or its files (without `.git`)
  take more than 512 MiB. Keep large weights out of Git: download them in the Dockerfile
  (builds have network) and verify a checksum;
- the image does not build. `Dockerfile` must be at the repository root and build for
  linux/amd64 within 30 minutes; the image may be at most the manifest's
  `container_size_limit_gb` (at most 40 GB; measured after the build: the apparent size of
  every file in the image, 1 GB = 10^9 bytes). Prefer prebuilt encoder binaries to compiling
  from source;
- the Dockerfile uses something the sandbox builder does not support. The builder is not
  Docker/BuildKit: `ADD` is rejected (use `COPY`, or download inside a `RUN`), `USER` is
  ignored, and an image reference in `FROM` or `COPY --from` must carry a tag **or** a
  digest, never both (`name:tag@sha256:...` fails with "Docker references with both a tag
  and digest are currently not supported"; `name@sha256:...` works). A build that passes
  locally can still fail here, so keep the Dockerfile to plain `FROM`/`COPY`/`RUN`/`ENV`/
  `WORKDIR` and test it against these rules before pinning your commit
  (`python scripts/competition_precheck.py <repo> --commit <sha>` checks the exact commit
  you are about to enroll against all the entry-losing rules in this list);
- the image has no `/bin/sh` or no `/app/run.sh`. Your `ENTRYPOINT`/`CMD` are ignored, no
  environment variable is injected, there is no network at run time, and every call runs in a
  fresh sandbox: nothing you write survives from one batch to the next.

*A whole batch scores zero when:*
- `run.sh` runs longer than `batch_timeout_seconds`. That budget bounds ONE whole `run.sh`
  call, and one call receives `evaluation_batch_size` clips (five clips per call when the
  manifest says 5). It is wall-clock time and includes any quality measurement you run
  yourself, so keep a margin and track elapsed time inside the script;
- `run.sh` exits non-zero. Do not let one failing clip abort the script (`set -e` does exactly
  that): catch the failure and fall back to a safe encode for that clip;
- the output directory exceeds 512 MiB for one file or 2 GiB in total **at any moment of the
  run**, temporary files included, or holds more than 4096 entries. Work under `/tmp` and
  delete temporary files. The local Docker runner used for self-testing caps `/tmp` at 256 MB,
  so prefer pipes to large intermediate files;
- stdout plus stderr exceed 8 MiB. Run encoders with `-loglevel error` or redirect their logs;
- an output is a symlink or anything other than a plain regular file.

*If the platform stops your sandbox* (`run.sh` is killed by a signal and the sandbox itself
is gone, for example a preemption), the batch is rerun on a fresh sandbox, at most twice;
it scores zero only if that keeps happening. A `run.sh` killed while its sandbox keeps
running (for example out of memory) is scored as a non-zero exit. Do not trap SIGTERM to
exit 0: a batch that exits 0 is scored on the outputs it wrote, and missing clips score
zero.

*One clip scores zero when:*
- its output is missing, empty, or not named exactly like the input. Inputs are named by their
  sha256 and have **no file extension**, so tell your tools the container explicitly
  (`-f mp4`) and let them probe the input format;
- the codec is not H.264, HEVC, VP9 or AV1 (nothing else is accepted, whatever the decoder
  could read);
- width, height or the number of frames differ from the input, the duration differs by more
  than 5 %, or the timestamps are inconsistent. Do not resample the frame rate: clips come at
  24, 29.97, 30 fps and others, with frame sizes that are not always a multiple of 8 or 16.
  Pad internally if your encoder needs it, but the decoded output must have the input's size;
- mean VMAF (`vmaf_v0.6.1`, whole clip, against the input you received) is below the
  manifest's `vmaf_threshold`. A second VMAF model and chroma/tone checks run as well:
  sharpening, contrast or colour tricks that inflate VMAF zero the clip.
Outputs are compared after conversion to 8-bit 4:2:0, so 10-bit encodes are fine.

## What your code may do

Anything that fits the contract and the compute envelope.
Analysing each input and choosing the encoder, preset, filters or GPU path per clip,
encoding several candidates and keeping the best, and measuring VMAF yourself to search
for the smallest file that clears the gate are all allowed and expected; clips differ a
lot (live action, animation, AI-generated footage, film grain, screen recordings), so a
single fixed setting will do badly somewhere. Everything you need must be inside the
image: a run has no network.

**No banned settings.** There is no list of banned encoder or model settings: stock tunes
(including VMAF-oriented ones), pre-filters and custom models are all allowed. The per-clip
gates and the scorer decide, the same for everyone; a clip that fails a gate scores zero, so
keep a margin instead of sitting just under a limit. What is forbidden is fitting an entry to
the sealed evaluation set: an entry must compute its output from the input it is given, and
shipping copies of footage to look the answer up gets the entry rejected and its operator
barred (see the content rules in the [track sections](#tracks) below).

**Two things that cost real entries a clip.** (1) Check your encoder on odd frame sizes.
Distribution packages can be old: the SVT-AV1 1.7 shipped with Ubuntu 24.04 crashes on
frame widths that are not a multiple of 8, and that ffmpeg has no `libvmaf` filter, so a
quality search silently degrades to a fixed setting. The example image installs a current
static ffmpeg build instead and fails the *build* when a needed encoder or filter is
missing. (2) Leave a margin above the VMAF gate. The scorer measures the full clip with
its own pinned libvmaf build and model; your measurement can differ by a few tenths, and
a clip below the gate scores zero.

## Tracks

Every track uses the same `run.sh` contract and the limits above; the items, the
per-clip rules and the score differ by track, and generative upscaling also changes the
sandbox. Check the manifest of the competition you enter for its track and scoring mode.

### Compression (track `compression`)

Each clip is re-encoded smaller and scored with the compression formula under
[How competitions work](#how-competitions-work): fewer bytes at a VMAF above the
manifest's `vmaf_threshold`, measured against the input you received. The per-clip gates
are in the [checklist](#checklist-what-loses-you-a-clip-a-batch-or-the-whole-entry);
start from `examples/competition_contenders/cpu_compression/` or the GPU `template/`.

### Object removal (track `removal`)

The contract above is unchanged
(same `run.sh`, same limits); only the items and the score differ.
- *Input:* each item is ONE Matroska file (named by its sha256, no extension) with two
  video streams. Stream 0 is the clip with an object in it (FFV1, 4:2:0, at most
  1280×720, constant frame rate). Stream 1 is the per-frame mask (FFV1 gray, same size
  and frame count; luma above 127 marks the pixels to reconstruct). Read them with
  `-map 0:v:0` and `-map 0:v:1`. Masks are object-shaped and change from frame to frame;
  a frame whose mask is empty (the object is out of view) must come back unchanged.
- *Output:* one MP4 per item with a single H.264, HEVC, VP9 or AV1 stream of the same
  width, height, frame count and frame rate. Pixels outside the mask must stay what you
  received: the largest per-frame mean absolute RGB change outside the mask must not
  exceed 3.0, so encode losslessly (`libx264 -qp 0`) or near-losslessly (all-intra
  `-crf 8`), and paste your model's result back inside the mask only. Output size is not
  scored, but the 512 MiB / 2 GiB caps still apply.
- *Score:* only the masked region is measured, against the clean original (sealed during
  the competition, published afterwards). The validator first measures two free
  answers: its own fill (per-pixel temporal median of the unmasked frames, Telea
  inpainting where the background is never visible) and the clip returned unchanged. The
  floor is the better of the two on each metric (higher PSNR, lower LPIPS), so doing
  nothing never scores. An item scores zero when your region PSNR is not at least 1.5 dB
  above that floor, when your region flickers (temporal warp error above 5× the
  original's), when pixels outside the mask changed, or on a size/frame-count mismatch.
  Otherwise it scores `0.6 · min(1, (PSNR − floor PSNR) / 8 dB) +
  0.4 · (floor LPIPS − LPIPS) / floor LPIPS` (clamped to [0, 1]). The code is
  `vidaio/scoring/removal.py` and `vidaio/scoring_worker/removal_backend.py`.
- *Content:* the hidden items mix difficulty levels: static objects, moving objects the
  mask follows, objects that leave the frame (or vanish) and come back, and hard cases
  (large, fast, zooming or several objects). The clean background is often visible in
  other frames, so methods that propagate real pixels across time (flow-guided video
  inpainting) do far better than inpainting each frame on its own; check the licence of
  any model you ship.
- *Start from* `examples/competition_contenders/removal_example/`: a complete CPU entry
  that implements the contract (it is the free fill, so it scores about zero) with a
  local self-test.

### Frame interpolation (track `interpolation`)

The contract above is
unchanged (same `run.sh`, same limits); only the items and the score differ.
- *Input:* each item is ONE Matroska file (named by its sha256, no extension) with one
  video stream (FFV1, 4:2:0, at most 1920×1080, constant frame rate): a genuine
  high-frame-rate clip with every other frame removed, so its rate is half the original
  (for example 30 fps made from 60 fps). N input frames.
- *Output:* one MP4 per item with a single H.264, HEVC, VP9 or AV1 stream of the same
  width and height, **twice the input frame rate** and **exactly 2N − 1 frames**: frame
  2k is input frame k, frame 2k + 1 is your new frame between input frames k and k + 1.
  The input frames must come back unchanged: the largest per-frame mean absolute RGB
  change between output frame 2k and input frame k must not exceed 3.0 (this also catches
  an output shifted off the grid), so encode near-losslessly (`libx264 -crf 8`). Drive
  your model on the exact grid: wrappers that skip near-duplicate frames, repeat frames
  across cuts or round the frame rate change the frame count, and the item scores zero.
- *Score:* only the inserted frames are measured, against the real frames that were
  removed (the full-frame-rate original is sealed during the competition and published
  afterwards): PSNR over all their pixels, SSIM on luma, LPIPS-VGG on every 4th inserted
  frame. The validator measures two free answers the same way, repeating the previous
  frame (hold) and averaging the two neighbours (blend), and the floor is the better of
  the two on each metric. Each term credits only the gain over that floor, as a share of
  what is left to gain: `s_psnr = min(1, max(0, (PSNR − floor PSNR) / 8 dB))`,
  `s_ssim = clamp((SSIM − floor SSIM) / (1 − floor SSIM))`,
  `s_lpips = clamp((floor LPIPS − LPIPS) / floor LPIPS)`, and the item scores
  `0.3 · s_psnr + 0.3 · s_ssim + 0.4 · s_lpips`. Returning the hold or the blend scores
  zero; an item also scores zero on a frame-count, size or kept-frame violation. The code
  is `vidaio/scoring/interpolation.py`, `vidaio/scoring/interpolation_formula.py` and
  `vidaio/scoring_worker/interpolation_backend.py`.
- *Speed counts* when the manifest's `result_rules` carry a speed term
  ([Speed](#speed)), with `speed_inserted_frames` = the inserted frames over the hidden
  set.
- *Content:* the hidden clips are genuine high-frame-rate footage with real motion (no
  duplicated or static frames), with easy and hard cases mixed: camera pans, fast
  subjects, occlusions, thin structures and fine texture. Methods that estimate motion
  between the two neighbours do far better than blending. An entry must compute its new
  frames from the input it is given: shipping video, frames or any other copy of existing
  footage in the image to look the answer up is not a solution, and an entry found doing
  it is rejected and its operator barred from later competitions.
- *Licences:* the [licence rules](#licences) apply (commercial use only, `LICENSE` and
  `LICENSES.md`, every download pinned by content), and so does the training-data rule.
- *Start from* `examples/competition_contenders/interpolation_example/`: a complete CPU
  entry that implements the contract (it is the blend, so it scores about zero) with a
  local self-test.

### Streaming compression (track `compression`, streaming scoring)

A compression manifest that says `"compression_scoring": "streaming"` scores each clip as a
streaming rendition: the best quality inside a fixed bitrate budget, not the smallest
file over a VMAF bar. The contract above is unchanged (same `run.sh`, same limits).
- *Input:* each item is one lossless clip (FFV1 in Matroska, named by its sha256).
- *Budget:* the manifest anchors `streaming_rules`. Each clip's budget is
  `bits_per_pixel × width × height × fps / 1000` kbps, computed from the clip itself (for
  example 0.05 bpp gives 3.1 Mbps at 1080p30 and 1.4 Mbps at 720p30).
- *Output:* one file per item, a single H.264, HEVC, VP9 or AV1 stream with the input's
  size, frame count and frame rate.
- *Delivery rules,* measured on your video packets (container overhead not counted).
  Breaking one zeroes the clip:
  - the average bitrate stays within the budget, with 1 % slack;
  - no window of `peak_window_seconds` consecutive frames goes above `peak_ratio` × the budget;
  - the first frame is a keyframe, and keyframes are at most `max_keyframe_interval_seconds` apart, so the stream can be segmented.
- *Score:* `(VMAF − 60) / 40`, clamped to [0, 1]. VMAF is measured as for compression (same
  pinned model, the second-model check and the chroma and tone checks against sharpening
  or colour tricks). There is no VMAF threshold: quality is the score.
- *Speed counts* when the manifest's `result_rules` carry a speed term
  ([Speed](#speed)), with `speed_inserted_frames` = the output frames over the hidden set.
- *Licences:* the same as the other tracks, as the streaming example's README states: the
  [licence rules](#licences) apply (commercial use only, `LICENSE` and `LICENSES.md`,
  every download pinned by content), and so does the training-data rule.
- The code is `vidaio/scoring/streaming.py` and `vidaio/scoring_worker/streaming_backend.py`.
  Start from `examples/competition_contenders/streaming_example/`: a complete x264 entry
  that measures its own packets, with a local self-test.

### Faithful upscaling (track `upscaling`, faithful scoring)

The contract above is
unchanged (same `run.sh`, same limits); only the items and the score differ. A manifest
that says `"upscaling_scoring": "faithful"` is scored as below; check the manifest of the
competition you enter.
- *Input:* each item is ONE video file (named by its sha256, no extension; H.264 in MP4,
  so probe it rather than trusting a name): a sealed 1080p original, degraded and
  downscaled by 2 (to 540p) or by 4 (to 270p), about four seconds at a constant frame rate.
  The input is exactly the target divided by the factor (960×540 or 480×270; 540×960 or
  270×480 in portrait), with no crop. Next to each input, a hidden file
  `.vidaio-next-upscale-task-<sha256>` (the input's name) holds the task as JSON, for example
  `{"target_height":1080,"target_width":1920,"upscale_factor":2}`.
  Originals are landscape or **portrait**: the target is 1920×1080 or 1080×1920, so
  always read the target size from that file. The folder holds nothing else.
- *Output:* one MP4 per item with a single H.264, HEVC, VP9 or AV1 stream of exactly the
  target width and height, the input's frame count and frame rate. Your encode is
  measured too, so encode at high quality (`libx264 -crf 10` or lossless); output size
  is not scored, but the 512 MiB / 2 GiB caps still apply. Frames are compared in 8-bit
  4:2:0 BT.709 limited range: tag your stream (`-colorspace bt709 -color_primaries bt709
  -color_trc bt709 -color_range tv`); an untagged stream is read as exactly that.
- *Score:* every output frame is compared with the original it must restore (sealed
  during the competition, published afterwards): PSNR over all pixels, SSIM on luma,
  LPIPS-VGG on every 4th frame. The validator measures its free answer the same way: the
  input Lanczos-upscaled to the target size (ffmpeg `scale=W:H:flags=lanczos`). Each
  term credits only the gain over that floor, as a share of what is left to gain:
  `s_psnr = min(1, max(0, (PSNR − floor PSNR) / 6 dB))`,
  `s_ssim = clamp((SSIM − floor SSIM) / (1 − floor SSIM))`,
  `s_lpips = clamp((floor LPIPS − LPIPS) / floor LPIPS)`, and the item scores
  `0.3 · s_psnr + 0.3 · s_ssim + 0.4 · s_lpips`. Returning the Lanczos upscale scores
  zero, and so does a size or frame-count mismatch. **Faithful, not generative:** detail
  that is not in the original moves your frames away from it and costs PSNR, SSIM and
  LPIPS, so a model that restores what was lost beats one that invents texture. No VMAF,
  PieAPP or file-size gate applies. The code is `vidaio/scoring/upscaling_faithful.py`,
  `vidaio/scoring/upscaling_faithful_formula.py` and
  `vidaio/scoring_worker/upscaling_faithful_backend.py`.
- *Speed counts* when the manifest's `result_rules` carry a speed term
  ([Speed](#speed)), with `spf` = total seconds / `speed_inserted_frames`, which for
  upscaling is the number of output frames over the whole hidden set.
- *Content:* the hidden originals are private phone footage, 3D renders and AI-generated
  clips (only ones with genuine full-resolution detail), never from public stock sites, and
  every clip comes from a different video: handheld, vehicle and drone motion, faces, fur,
  foliage, water, text and signs, fine texture, night scenes; frame rates from 24 to 60 fps,
  landscape and portrait. How hard the upscale is varies on purpose, from mild ×2 to heavily
  degraded ×4. The degradation is realistic and published in kind (downscale,
  slight blur, a web-bitrate H.264 encode, sometimes low-light noise); the parameters of
  each item stay sealed. An entry must compute its output from the input it is given:
  shipping video, frames or any other copy of existing footage in the image to look the
  answer up is not a solution, and an entry found doing it is rejected and its operator
  barred from later competitions.
- *Licences:* the [licence rules](#licences) apply unchanged (commercial use only,
  `LICENSE` and `LICENSES.md`, every download pinned by content), and so does the
  training-data rule. In `practice-pack-004`, p01–p07 (VidAIO's own phone footage and
  CC0 Blender renders) may be used for training; p08–p10 were generated with LTX-2.5, so
  use them only to evaluate your entry, never to train or tune it.
- *Start from* `examples/competition_contenders/upscaling_example/`: a complete CPU entry
  that implements the contract (it is the Lanczos floor, so it scores about zero) with a
  local self-test.

### Generative upscaling (track `upscaling`, generative scoring)

A manifest
that says `"upscaling_scoring": "generative"` asks for a generative AI upscaler: inventing
plausible detail the degraded input no longer holds is the point, as long as the result
stays true to the input. The contract is the faithful one above (same `run.sh`, same task
file `.vidaio-next-upscale-task-<sha256>`, same output rules); the sandbox, the gates and
the score differ.
- *Task:* ×2 (960×540 → 1920×1080) and ×4 (480×270 → 1920×1080), portrait mirrored;
  clips of up to 10 seconds at a constant frame rate up to 60 fps. Return every frame, in
  order, at exactly the target size, the input's frame count and frame rate, as one H.264,
  HEVC, VP9 or AV1 stream tagged BT.709 limited range. Encode at high quality (x264 CRF 12
  or better) within the 512 MiB / 2 GiB caps; the validator may also score your clip after
  its own light delivery encode (x264 CRF 18), so do not rely on detail that only survives
  a lossless file.
- *Sandbox:* one **NVIDIA A100 80 GB** per run (the single entry of the manifest's
  `allowed_gpus`, Modal `A100-80GB`); the build gets no GPU and must finish within 30
  minutes; the image may be up to the manifest's `container_size_limit_gb`, **40 GB** for
  generative competitions, so bake your weights into the image (there is no network at run
  time). Every `run.sh` call starts a fresh sandbox: your model loads once per batch, and
  that time counts.
- *Gates* (a failing clip scores zero): the usual geometry and frame-count checks, plus
  **low-resolution consistency** (your frames, Lanczos-downscaled back to the input's size
  and slightly blurred, must match the equally blurred input: at least 28 dB of luma PSNR,
  or the calibrated floor in the scoring config if higher) and **colour** (the mean CIEDE2000
  against the original on downscaled frames is at most 6). Invented structure the input does
  not support, a shifted picture or a colour cast fails a gate. When the manifest's
  `result_rules` carry `max_failed_item_fraction` (0.1 means 10 %), an entry with more
  clips at zero than that share ranks with a score of 0 and holds no paid place.
- *Score:* `0.55 · fidelity + 0.15 · temporal + 0.30 · quality`, each term placed between
  two anchors published in the scoring config: the weakest of three public baseline
  upscalers (plain interpolation, a GAN upscaler, a diffusion upscaler) scores 0 on that term
  and the strongest scores 1 (separately for ×2 and ×4; the values are in the config).
  *Fidelity* is the mean of `1 − DISTS` and `1 − LPIPS-VGG` against the sealed original on
  a few sampled **full 1080p frames** (no downscale before the networks). *Temporal* is the
  original's own warp error divided by yours under the original's optical flow, capped at 1:
  flicker and boiling texture cost points, and being smoother than the original earns
  nothing extra. *Quality* is CLIP-IQA (a no-reference metric) relative to the original's
  own value, **capped at the original + 5 %**: looking sharper than reality earns nothing.
  Everything runs on CPU, deterministically, with pinned weights, so auditors recompute the
  same numbers. The code is `vidaio/scoring/upscaling_generative.py`,
  `vidaio/scoring/upscaling_generative_formula.py` and
  `vidaio/scoring_worker/upscaling_generative_backend.py`.
- *Speed counts* when the manifest carries a speed term, exactly as for faithful upscaling
  (`speed_inserted_frames` = output frames over the whole hidden set; see [Speed](#speed)).
- *Licences:* **permissive only**. Everything that makes your output, model **code and
  weights**, must be Apache-2.0, MIT or BSD licensed (for example SeedVR2, FlashVSR,
  Real-ESRGAN and BasicVSR++ qualify). Conditional or community licences (the CogVideoX
  licence, the LTX community licence), non-commercial, academic-use-only, territory-restricted
  and unlicensed code or weights are **not eligible**. GPU runtime libraries in your base
  image (CUDA, cuDNN, NCCL, TensorRT, under NVIDIA's licence) are not models and are fine;
  list them in `LICENSES.md`. `LICENSE` and `LICENSES.md` are
  required and every download is pinned by content, as in [Licences](#licences); the
  training-data rule applies unchanged, and the content rule above (no shipped copies of
  existing footage) too.
- *Start from* `examples/competition_contenders/generative_upscaling_example/`: a GPU
  skeleton (PyTorch on CUDA, raw-frame pipes, a weight-free stub where your model goes)
  that builds within the limits; it scores near the floor by design.

## Speed

*Speed counts* when the manifest's `result_rules` carry a speed term (all four
`speed_*` fields; frame interpolation, faithful and generative upscaling and streaming
compression). Every `run.sh` call is timed by the
validator's runner, from start to exit, on the single pinned GPU type and the anchored
CPU/RAM envelope (`allowed_gpus` has one entry), and the times are added over all your
batches. A batch your solution fails (non-zero exit, timeout, unsafe output) counts its
full `batch_timeout_seconds` (as raised by any execution amendment), so failing fast
never buys speed. When a batch's Sandbox is stopped and the batch is rerun, the stopped
attempt counts too, up to the time of the run that completed, so bringing your own
Sandbox down never buys a faster run. Model loading inside `run.sh` counts too. With
`spf` = total seconds / `speed_inserted_frames` (the inserted frames over the hidden set
for frame interpolation; the output frames over the whole hidden set for upscaling and
streaming compression):
`speed_score = clamp(log(speed_slow_spf / spf) / log(speed_slow_spf / speed_fast_spf), 0, 1)`
is 1 at or under `speed_fast_spf` (for example 1/30 s per inserted frame: real time for
30 to 60 fps) and 0 at or over `speed_slow_spf`, and every 2x speed-up is worth the same.
Your competition score is `mean item score x (1 - speed_weight + speed_weight x
speed_score)`; the crown and podium bars apply to that number. The per-batch times are
committed with the result (attested by the validator that ran the sandboxes) and the
auditor re-derives the score from them.

## Licences

These rules apply to frame interpolation, streaming compression and faithful upscaling;
generative upscaling applies them with its own stricter licence list (see
[its section](#generative-upscaling-track-upscaling-generative-scoring)). For any
competition, `list` shows whether `LICENSE` and `LICENSES.md` are required.

- *Licences: commercial use only.* The winning solution runs on the VidAIO website, so
  everything an entry ships (code, libraries, model weights) must be under a licence that
  allows commercial use (Apache-2.0, MIT, BSD, ISC, MPL-2.0, CC0/Unlicense, CC-BY-4.0, or
  GPL/LGPL such as ffmpeg).
  Non-commercial, research-only, no-derivatives, AGPL/SSPL or unlicensed components are
  refused, including fine-tuned, merged, adapted or distilled versions of such models.
  Weights that a model's authors publish from its repository (linked in its README or
  release) take that repository's licence. A file the image ships but your solution
  never loads at run time is not judged.
- *Training data: commercial training only.* Everything you train, fine-tune or tune your
  entry on (weights, thresholds, filter strengths or any other setting) must allow
  training a model for commercial use: your own footage, CC0 or CC-BY material, or
  synthetic data from a generator whose terms allow training other models on its outputs
  (for example Apache-2.0 Wan 2.x). Non-commercial (CC BY-NC, CC BY-NC-SA), research-only
  or no-derivatives datasets, and outputs of models whose licence restricts training on
  them, are not allowed. LTX-2.x is one: its licence (Attachment A, item 18) forbids using
  its outputs to train, improve or fine-tune another model for commercial use, and
  §3.2 makes models trained on its outputs LTX derivatives. You may still use such
  material to evaluate your entry (to measure how it scores), never to train or tune it.
  Pretrained weights you build on are judged by their own published licence, not by the
  data their authors trained them on. We cannot fully verify training data, so keeping it
  compliant is your responsibility; an entry whose weights or settings come from data
  that breaks this rule is not accepted.
- *Your own code and the component list.* Your own code must be released under
  Apache-2.0 or MIT (a `LICENSE` file at the
  repository root at the enrolled commit), and a `LICENSES.md` at the root must list
  every third-party component (name, source URL, licence, location in the repository).
- *Pinned downloads.* What is reviewed must be what is built, so every file your Dockerfile downloads must be
  pinned by content: check its sha256 in the same `RUN` step
  (`echo "<sha256>  model.pth" | sha256sum -c -`), or use a Hugging Face commit revision,
  a git commit, `pip install` with `==` versions or `--require-hashes`, or `apt-get
  install` with exact versions (`pkg=1.2.3-1`) or from a dated snapshot mirror
  (`snapshot.debian.org`, `snapshot.ubuntu.com`). An entry with an unpinned download is
  rejected. `python scripts/competition_precheck.py <repo> --commit
  <sha> --commercial` flags the common cases.
- *When it is checked.* Entries are checked in the 24 h between the enrollment deadline
  and the start of evaluation; an entry that breaks these rules is rejected then, publicly
  with the reason, and never scored. A violation found later, until the competition
  completes, takes the entry out the same way: it leaves the scoring and the result.

### Licence rulings so far

Answers the subnet team has given miners in Discord, applying the
rules above; each binds every entry the same way. They were given under the commercial-use
rules of frame interpolation and faithful upscaling: where a competition's own rules are
stricter (generative upscaling: model code and weights Apache-2.0, MIT or BSD only), those
win.
- *Weights linked by the original authors* (2026-10-06): weights a model's authors publish
  from its repository (linked in its README or release) take that repository's licence; for
  example IFRNet's checkpoints are MIT, like its code.
- *Files never loaded* (2026-10-16): a checkpoint the image ships but the solution never loads
  at run time is not judged.
- *Non-commercial training data* (2026-10-26): CC BY-NC-SA data such as Inter4K cannot be
  trained on, even when the weights are fine-tuned from a permissive base. Fine-tuning a
  permissively licensed base (for example Real-ESRGAN, BSD) on compliant data and releasing
  your weights under MIT or Apache-2.0 is fine.
- *Accepted training sources* (2026-10-26, 2026-10-27): your own Wan 2.2 clips (Apache-2.0),
  the YouTube UGC dataset (CC BY 4.0) and Netflix Open Content titles such as Meridian (CC BY
  4.0 on Netflix's own Open Content site); credit each in `LICENSES.md`. A Wan clip is fine
  even when you wrote its prompt after looking at restricted clips (2026-10-28): what is not
  allowed is training on restricted pixels.
- *Evaluation-only material* (2026-10-26, 2026-10-27): you may run restricted clips (for
  example the LTX-generated practice clips) to see how your entry scores and to compare
  builds. Training on them, or fitting a setting to them (such as picking a sharpening
  strength), is not allowed. The test: would the entry you ship be the same without the
  restricted clips?
- *CC BY-SA training data* (2026-10-28): accepted, since it allows commercial use. Credit each
  source in `LICENSES.md` plus an attribution file, and keep your own code under Apache-2.0 or
  MIT.
- *NVIDIA TensorRT runtime* (2026-10-28): accepted. NVIDIA's licence allows commercial use and
  redistribution of the runtime on NVIDIA GPUs, like the cuDNN/cuBLAS libraries PyTorch ships.
  List it in `LICENSES.md` (version, licence, source) and pin it like any other download.

## Result rules

A competition may publish its own `result_rules` in the anchored
manifest: `crown_margin` (the relative win over the rerun baseline that makes the result
a CROWN), `crown_min_score` (an absolute score the winner must also reach to crown),
and `podium_min_margin` / `podium_min_score` (what a contender must reach to hold a paid
rank at all). They are identical for every contender, fixed before enrollment, copied
into the epoch evidence and re-checked by auditors against the anchored manifest. When
the block is absent the protocol default applies (crown at an inclusive 5% margin, every
ranked contender eligible for the podium). Margins are measured against the rerun
baseline only, never against the runner-up, so entries never count against each other.
If the rerun baseline scores zero on the hidden clips, the relative margins cannot
discriminate and the absolute bars decide alone: `crown_min_score` for a CROWN and
`podium_min_score` for a paid rank. A competition may also be announced **without a
baseline** (its manifest has no `baseline` block): it must then anchor both
`crown_min_score` and `podium_min_score`, and those two numbers decide on their own —
reach `podium_min_score` to hold a paid place, and the first place crowns when it also
reaches `crown_min_score`.

## Results and what is published

The score that ranks and is paid is the final score:
the plain mean of your clip scores, multiplied by `(1 - speed_weight + speed_weight x
speed_score)` when the manifest's `result_rules` carry a speed term ([Speed](#speed)).
The crown and podium bars apply to that number. After the result the public
competition view and the hub show the results table: final score, quality (the mean
clip score), speed score, total run time, and how many clips failed with which failure
code (counts, not which clips). When
the competition completes, the evidence bucket receives every hidden input
(`challenge_input/<aa>/<bb>/<sha256>`), every scored output (`miner_output/...`) and, for
upscaling, removal and frame interpolation, the sealed originals
(`released/reference_original/...`); for compression the input you received is itself the
reference. A CROWN also publishes the winning entry's source archive, the exact tree the
evaluator archived (`released/submission_archive/<aa>/<bb>/<sha256>`). No other entry's
source is published, and a PODIUM result publishes none.
