# miner-vidaio

Mining on the **VidAIO subnet** (Bittensor **SN85**) means entering **competitions**.
You build a video-processing solution as a containerised entry in a Git repository,
enroll it with a request signed by your registered hotkey before the deadline, and the
validator builds it and evaluates it on a hidden set of clips in a sealed GPU sandbox.
Results are published, independently audited and paid from the subnet's miner
emissions. There is no live-serving mining any more: you do not run a miner server,
advertise an endpoint or answer validator requests.

This repository is what you need to take part: the enrollment client
(`scripts/competition_enroll.py`), the entry pre-check (`scripts/competition_precheck.py`),
a starting entry for every track (`examples/competition_contenders/`), and the public
scoring, competition and payout code that validators and auditors run, so you can check
exactly how you are scored and paid.

> **Release mirror.** Each VidAIO release lands here as one snapshot commit. The full
> rulebook is [docs/MINING.md](docs/MINING.md); this README is the quickstart.

---

## 1. How a competition works

- Each competition has one track: **compression**, **streaming compression**,
  **faithful upscaling**, **generative upscaling**, **object removal** or **frame
  interpolation**.
- Its manifest is public and anchored on chain before enrollment opens: the lifecycle
  times, the stake floor, the sandbox resources and GPU type, the result rules, the
  baseline (if any) and commitments to the hidden clips. Every contender gets the same.
- You enroll a pinned `repo_url + commit_sha + tree_sha`. The evaluator builds the
  `Dockerfile` at your repository root and runs `/bin/sh /app/run.sh <input_dir>
  <output_dir>`: one output per input under the same name, exit 0, no network.
- After the enrollment deadline your entry cannot change. Entries are finalized (the
  manifest's `finalization_time`, 24 h after the deadline so far), built, run on the
  hidden clips and scored. Your score is the plain mean of your per-clip scores (times a
  speed factor when the manifest has a speed term).

Details: [How competitions work](docs/MINING.md#how-competitions-work).

## 2. How payouts work

- Miner emissions go **100 % to competition results**. Stake never increases a reward.
- A result that meets the crown rule opens a **CROWN** window split **90/4/3/2/1** over
  the top five qualifying places; any other result opens a **PODIUM** window split
  **50/24/13/8/5**. Unfilled places go to the filled ones in proportion.
- A result pays for **two weeks (336 hours)**, until the next one. While a window is
  active the licensing treasury keeps 0.5 % and the split applies to the other 99.5 %.
  A result with no qualifying contender closes the window; a gap with no active result
  burns 50 % and sends 50 % to the licensing treasury.
- Each competition anchors its own pot, split and bars in its manifest
  (`result_rules`): check the manifest of the competition you enter.
- **[One operator, one paid place](docs/MINING.md#one-entry-per-operator).**
  Entries that share a coldkey, an advertised IP, a repository account, a source tree or
  identical outputs count as one operator.
- The hotkey must still be registered when the result is applied; a deregistered paid
  hotkey's share goes to the sink.

Details: [How you earn](docs/MINING.md#how-you-earn), [Result rules](docs/MINING.md#result-rules).

## 3. What you need

| Piece | Requirement |
|---|---|
| Chain | a Bittensor wallet and a hotkey registered on netuid **85** |
| Stake | alpha staked on that hotkey at or above the competition's floor (`list` prints it; 500 alpha in every competition so far), kept in place until entries are finalized |
| Code | a Git repository on an allowed host (`list` prints them), private recommended |
| Tools | Python 3.11+, `git`, `btcli`; Docker if you want to build your image locally; a GPU only if your approach needs one to test (the evaluator provides one per sandbox) |

## 4. Step by step

**1. Wallet and registration.**

```sh
pip install bittensor-cli
btcli wallet new_coldkey --wallet.name vidaio            # on a SECURE machine
btcli wallet new_hotkey  --wallet.name vidaio --wallet.hotkey miner1
btcli subnet register --netuid 85 --wallet.name vidaio --wallet.hotkey miner1
```

Enrollment is signed with the hotkey only; the coldkey is never used. Back up both
mnemonics offline: a lost hotkey is a lost registration.

**2. Stake.** Stake alpha on the hotkey on netuid 85 (from any coldkey) up to the floor.
It is checked when you enroll, on every `--replace`, and once more when entries are
finalized: an entry whose hotkey is below the floor then is rejected. `status` and the
hub warn while enrollment is open. See
[Stake and registration](docs/MINING.md#stake-and-registration).

**3. Install this repository.**

```sh
git clone https://github.com/vidAio-subnet/miner-vidaio.git
cd miner-vidaio && pip install -e '.[chain]'      # the enrollment client needs bittensor
```

**4. Find the current competition.**

```sh
python scripts/competition_enroll.py list --url https://95.133.254.122/competitions
```

`list` prints every competition: id, track, status, enrollment window, stake floor,
sandbox resources and GPU, result rules, and whether `LICENSE` and `LICENSES.md` are
required. The competition hub ([subnet.vidaio.io/competitions](https://subnet.vidaio.io/competitions))
and the Discord announcement give the schedule, and the current address if it ever moves.

**5. Build your entry** from the example for the track, in a fresh repository of its own:

| Track | Start from `examples/competition_contenders/` |
|---|---|
| compression | `cpu_compression/` (with `materialize_cpu.py`), or the GPU `template/` (with `materialize.py`) |
| streaming compression | `streaming_example/` |
| faithful upscaling | `upscaling_example/` |
| generative upscaling | `generative_upscaling_example/` |
| object removal | `removal_example/` |
| frame interpolation | `interpolation_example/` |

The input, output, score and content rules of each track are in
[Tracks](docs/MINING.md#tracks).

**6. Test locally.** Run the example's `test_local.py` self-test where it has one (no
Docker needed). Download the competition's practice pack from the `practice-pack-<NNN>`
release on this repository's
[releases page](https://github.com/vidAio-subnet/miner-vidaio/releases); recent packs
include reference scores and `score_practice.py`, which runs the validator's scorer. Then
check the exact commit you will enroll:

```sh
python scripts/competition_precheck.py /path/to/your-solution --commit <sha>   # add --commercial where the licence rules apply
```

**7. Push to a private repository and invite the reader.** Add the machine user
**`sn85competitionreader-collab`** as a collaborator before you enroll, and keep it until
the builds are announced as done. Do not force-push over, rename or delete the enrolled
commit before then. See [Use a private repository](docs/MINING.md#use-a-private-repository).

**8. Enroll** from your solution's clone, checked out at the pushed commit you want
evaluated (the client pins `HEAD` and its tree by default):

```sh
cd /path/to/your-solution
ENROLL=/path/to/miner-vidaio/scripts/competition_enroll.py
python $ENROLL enroll --url https://95.133.254.122/competitions --competition-id <id from list> \
    --repo-url https://github.com/<you>/<repo>.git --wallet-name vidaio --wallet-hotkey miner1
```

The client checks the entry locally before it signs anything. One entry per hotkey.

**9. Change, check or withdraw** until the enrollment deadline, from the same clone and
with the same `--url`, `--competition-id` and wallet flags:

```sh
python $ENROLL enroll --replace --repo-url ... ...   # point the entry at a new pushed commit
python $ENROLL status ...             # your entry, the pre-deadline checks, the entry commitment
python $ENROLL status --url ... --competition-id <id> --hotkey <ss58>   # anyone's public summary
python $ENROLL withdraw --yes ...     # final for this hotkey in this competition
```

The evaluator checks every entry version before the deadline and reports problems in
`status` and on the hub; it only warns, so fix the problem, push and `--replace` in time.

**10. Deadline and finalization.** After the deadline the evaluator fetches exactly the
commit and tree your entry holds; later pushes change nothing. Licence and rule checks
run in the 24 h between the deadline and the start of evaluation; an entry that breaks
them is rejected publicly with the reason.

**11. Results.** The hub and the public competition view show the results table (final
score, quality, speed score, total run time, failed-clip counts by failure code). When
the competition completes, the evidence bucket receives the hidden inputs, every scored
output and, for upscaling, removal and frame interpolation, the sealed originals; a CROWN
also publishes the winning entry's source. See
[Results and what is published](docs/MINING.md#results-and-what-is-published).

## 5. Sandbox and licences at a glance

- **Sandbox:** CPUs, memory, GPU type and the per-batch time limit come from the
  manifest. Every `run.sh` call gets a fresh sandbox (model loading counts against the
  time limit), with no network, no environment variables and no secrets. Outputs: at most
  512 MiB per file, 2 GiB in total, 4096 entries; at most 8 MiB of logs. The image must
  build within 30 minutes and fit the manifest's `container_size_limit_gb`; the builder
  rejects `ADD`, ignores `USER`, and refuses image references with both a tag and a
  digest. Full list: [Sandbox and limits](docs/MINING.md#sandbox-and-limits) and the
  [checklist](docs/MINING.md#checklist-what-loses-you-a-clip-a-batch-or-the-whole-entry).
- **Licences** (frame interpolation, streaming compression and both upscaling tracks;
  `list` shows whether a competition requires `LICENSE` and `LICENSES.md`): everything
  you ship must allow commercial use; your own code is
  Apache-2.0 or MIT with `LICENSE` and `LICENSES.md` at the root; every download is
  pinned by content; training data must allow commercial training. Generative upscaling
  accepts only Apache-2.0, MIT or BSD model code and weights. Full rules and the rulings
  so far: [Licences](docs/MINING.md#licences).

## 6. Troubleshooting

| Symptom | Likely cause → fix |
|---|---|
| `enroll` refused: hotkey not registered | Register the hotkey on netuid 85 (`btcli subnet register`) and enroll again |
| `enroll` refused: stake below the floor | Stake alpha on the signing hotkey up to the floor `list` prints |
| `this hotkey already has an entry` | Use `enroll --replace` to point the entry at a new commit |
| Local checks failed, nothing was signed | `Dockerfile` at the root, no symlinks or submodules, tree within 512 MiB, `LICENSE` and `LICENSES.md` when required; fix, commit, push, retry |
| `status` warns the repository cannot be fetched | Invite `sn85competitionreader-collab`, push the commit, and do not force-push it away |
| Entry BUILD_FAILED | The sandbox builder is not Docker: no `ADD`, tag **or** digest (never both), build within 30 minutes; run `competition_precheck.py` |
| A whole batch scored zero | `run.sh` timed out or exited non-zero (avoid `set -e` aborting on one clip), outputs or temp files over the caps, or more than 8 MiB of logs |
| A clip scored zero | Output missing or misnamed, wrong codec, size or frame count changed, or below the track's quality gate |
| Entry rejected at finalization | Stake dropped below the floor before `finalization_time` |
| Entry left out of the ranking | The hotkey was not registered when the result was applied |

## 7. Getting help

- **Discord**: the VidAIO server (invite via [vidaio.io](https://vidaio.io)); ask in the
  miners channel. Licence questions are answered there and recorded in
  [the rulings](docs/MINING.md#licence-rulings-so-far).
- **GitHub issues on this repository**: bugs, documentation gaps and reproducible
  scoring questions (include the competition id, your hotkey and the enrolled commit).

## License

MIT — see [LICENSE](LICENSE).

## Whitepaper

[Vidaio Whitepaper v1.0 (PDF)](docs/vidaio-whitepaper.pdf) — the mechanism design: measurement, continuous scoring, competitions, anchored epoch logs and independent audit, payout mechanics.
