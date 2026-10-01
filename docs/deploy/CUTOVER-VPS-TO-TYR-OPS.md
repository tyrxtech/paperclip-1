# Cutover runbook — live Paperclip: Hostinger VPS → tyr-ops (DRAFT)

Status: **DRAFT — not authorized to execute.** Every phase ends in a Marc gate.
Date drafted: 2026-10-01.
Scope: move the live Paperclip control plane from the Hostinger VPS
(`paperclip.tyr-x.com`, compose project at `/opt/tyrx-paperclip`) to
**tyr-ops** (the tyr-ops / dash-ops target host).

This runbook does **not** move the Mac Studio's agent workloads. The Studio
keeps executing agent runs, and the control plane that SSHes into it changes
hosts. It **does** move the security layer: the Wazuh manager that runs on the
Studio today relocates to tyr-ops under the locked co-location rule in
section 0.3.

## 0. How to read this

- Phases are strictly ordered. Do not reorder; later phases assume earlier
  verifications passed.
- Every phase has **Goal → Preconditions → Steps → Verify → Rollback → Gate**.
- `GATE Gn` means stop and get Marc's explicit go-ahead before continuing.
- **Gate numbering is local to this runbook.** `G0` to `G10`, plus the two
  lettered security gates `GS` (Wazuh relocation) and `GG` (Greenbone first
  deploy), are cutover checkpoints only. The security gates are lettered
  deliberately so they cannot be mistaken for governance gate numbers. They are
  not Marc's governance gates and they do not
  renumber, replace, or reinterpret any TYR-677 gate or the locked G12
  Vulcan-active decision. When you read a gate aloud, say "runbook gate G*n*".
- Anything written as `<angle-brackets>` is **unverified** and must be filled in
  during Phase A discovery. This runbook deliberately does not invent facts
  about tyr-ops.

### 0.1 HARD STOP list — never do these during this cutover

1. **No Vulcan state change as part of this cutover.** Vulcan is **active** by
   Marc's locked G12 decision, and that decision stands — this runbook does not
   ban it and must not be read as a pause order. What is forbidden here is
   *changing* Vulcan's state, seat, or config to work around a cutover problem.
   During the cutover window Vulcan is governed by the same rules as every other
   agent: it is held by the task drain in Phases 7 and 8, and it is not
   hand-released, re-seated, or re-pointed outside a Marc gate. Vulcan's seat and
   its config/instructions move with the Paperclip instance (section 0.3).
2. **No Hermes routing change.** Do not touch Hermes routing, adapters, or
   gateway config as part of the move.
3. **No Paperclip recycle, restart, or redeploy on the live VPS without a Marc
   gate.** The task drain is per-process and in-memory: a restart clears an
   armed drain even while the database still holds running rows. A careless
   restart after Phase 7 silently re-admits work.
4. **No secret value in chat, PR, issue, commit, Paperclip comment, shell
   argument, or terminal transcript.** Secrets move vault → target file only.
   Edit `.env` in place in the container/host with an editor; never pass a value
   on a `docker exec` command line.
5. **No feature-flag changes.** `PAPERCLIP_JEV_SHADOW_ENABLED` stays `false`;
   the cutover authorizes no new live third-party calls.
6. **No force-push, rebase, or deletion of any `deploy/*` branch.** They are the
   record of what ran in production.
7. **No merge of the cutover branch into `master`** during the window, and no
   bootstrapping of the new host from `master`.
8. **No destruction of the VPS, its volumes, its database, or the old DNS record**
   until the watch window in Phase 10 closes with Marc's sign-off.
9. **No DNS flip** until the new host passes health checks *and* completes one
   verified end-to-end agent run (Phase 8).
10. **No split security layer after program acceptance.** All four components —
    Paperclip board, Vulcan seat and config, Wazuh manager, **and Greenbone** —
    run on tyr-ops. A temporary split during the window is allowed only
    under an explicit Marc gate, and only with a written end state that is fully
    co-located on tyr-ops. Acceptance and decommission (runbook gate G10)
    cannot pass while any component in section 0.3 still runs on another host.
11. **No security component on Mac2, ever.** Mac2 is a build and workstation
    machine only. Do not place the Wazuh manager, Greenbone, the Paperclip
    control plane, or Vulcan's primary seat on it — not permanently, and not as
    a "temporary" overflow when tyr-ops looks tight. If capacity fails,
    **STOP** and escalate to Marc; do not relieve pressure by splitting onto
    Mac2 or back onto the Studio.
12. **Greenbone is in scope for this stack, and tyr-ops is its only
    permitted host.** Greenbone CE is **required**, not optional and not a
    someday-elsewhere item. It is undeployed today; the code is in
    `tyrxtech/tyrx-ai-security`. The deployment must be **planned for
    tyr-ops**, and it must **not be deployed now without Marc**. Do not
    stand it up on the Studio, on Mac2, or on the VPS, not even as an interim.
13. **Do not accept the cutover as done while either of these is true:**
    Greenbone is both undeployed *and* unscheduled on tyr-ops, or the Wazuh
    manager remains permanently on the Mac Studio. Either condition means the
    single-location security stack does not exist yet, so the program is not
    complete regardless of how healthy the board is on the new host.

### 0.2 Locked inventory (do not contradict without evidence)

| Fact | Consequence for this cutover |
| --- | --- |
| Live prod code is branch `deploy/tyr746-26df56cc`, **not** `master` | Bootstrapping from `master` drops the TYR-668 reopen rule, TYR-722 scheduler liveness, and both TYR-746 wake-queue fixes, and pulls in ~196 upstream commits that have never run here |
| `PAPERCLIP_TASK_DRAIN_ON_START` exists on the deploy lineage, unmerged (PR #2) | The first-boot drain is only available if the new host is built from the cutover branch |
| TTL native-restart hole may sit on `cursor/ttl-native-restart-recovery-f723`, no PR | Explicit carry-or-skip decision required in Phase 5 |
| Instance secrets and DB mode are hand-carried (`PAPERCLIP_HOME=/paperclip`, instance `default`, `.env` at `0600`), not in git | Phase 3 is manual and vault-mediated; nothing here is recoverable from the repo |
| Host change ⇒ update Mac `authorized_keys` **and** `knownHosts` in the execution spec | Phase 4; agent runs fail closed until both sides match |
| PR #5 (`node_modules` SSH sync) still open, based on the superseded `deploy/tyr722-0c6bc08e` | Without it, the ~2 h per-run copy-back follows the move to the new host |
| Draft PR #6 documents Node 24.11+ / pnpm 9.15.4 / frozen-lockfile mismatch | Host build prerequisite regardless of whether #6 merges |
| Dual-host DB heartbeat risk | Both hosts must never have a working path to the same database while either admits work (Phase 6) |
| Live Paperclip remains on the VPS today | The VPS is the source of truth until Phase 9 completes |
| The entire security layer must be co-located on tyr-ops (section 0.3) | The cutover is not complete when only the board moves; the Wazuh manager must relocate off the Studio, and capacity must be measured first |
| Greenbone CE is a **required** member of the stack, undeployed today | Its first deployment targets tyr-ops and must be scheduled; an undeployed *and* unscheduled Greenbone blocks acceptance (HARD STOP 13). The sizing bar for the host therefore includes Greenbone from the start, not later |

### 0.3 LOCKED — the entire security layer is co-located on tyr-ops

Marc locked this rule. Security must not be split across hosts. tyr-ops is
the single security host, and it must house all four of the following. All four
are **required members** of the stack — none of them is optional, and none of
them is deferred to another host:

| # | Component | State today | Requirement for this cutover |
| --- | --- | --- | --- |
| 1 | **Paperclip board** (control plane) | Hostinger VPS | Moves to tyr-ops — Phases 1 to 9 of this runbook |
| 2 | **Vulcan agent seat + config/instructions** | On the live Paperclip instance | Moves with the Paperclip instance. The seat is not re-pointed to another host, and the config/instructions travel with the instance data in Phase 3 |
| 3 | **Wazuh manager** | Mac Studio | **Relocates to tyr-ops.** Agents are re-pointed from the Studio manager to the tyr-ops manager. A permanent split is not acceptable, and leaving it on the Studio blocks acceptance (HARD STOP 13) |
| 4 | **Greenbone CE** | **Required**, undeployed. Code lives in `tyrxtech/tyrx-ai-security` | **In scope for this stack.** Its first deployment targets tyr-ops and no other host. Plan and schedule it as part of this program; do **not** deploy it now without Marc. An undeployed *and* unscheduled Greenbone blocks acceptance (HARD STOP 13). (That repository is not readable from this agent's credentials, so this runbook makes no claim about its contents, and its footprint must be filled in at A15) |

**Mac2 is explicitly out of scope as a security host.** Mac2 is a build and
workstation machine only: no Wazuh manager, no Greenbone, no Paperclip control
plane, no Vulcan primary seat. See HARD STOP 11.

**The Mac Studio keeps executing agent runs.** Relocating the Wazuh manager off
the Studio does not change the Studio's role as the execution host, and it does
not change the SSH execution path in Phase 4.

#### Sequencing

The Wazuh relocation and the Greenbone first deployment are distinct
workstreams, each with its own Marc gate. Neither may be interleaved with the
database and DNS phases, because a failure in either one would then be hard to
attribute. Record the chosen order at A16; whatever the order, the end state is
all four components co-located on tyr-ops.

1. Capacity is measured first, for the **full** stack including Greenbone —
   Phase A rows A12 to A16. Capacity is **unverified until measured**.
2. The board cutover completes through runbook gate G8 (new host healthy, one
   clean end-to-end run).
3. Relocate the Wazuh manager and re-point agents, under a Marc gate (gate GS).
4. Deploy Greenbone CE on tyr-ops, under a separate Marc gate (gate GG).
   It must be scheduled before acceptance even if the deployment itself lands
   after the board watch window; an undeployed *and* unscheduled Greenbone
   blocks acceptance.
5. Acceptance and decommission (runbook gate G10) require the co-located end
   state in full: board, Vulcan seat and config, Wazuh manager, and Greenbone
   deployed or scheduled on tyr-ops, with nothing on Mac2.

#### Temporary dual-run

A temporary period where a security component runs in two places, or still runs
on the Studio after the board has moved, is permitted **only** when all three
hold:

- Marc gives an explicit gate for the temporary state.
- The written end state is recorded in this runbook and is fully co-located on
  tyr-ops.
- A named owner and a review date are recorded with it.

An undocumented split, or a split that outlives acceptance, is a HARD STOP 10
violation.

#### Capacity is the gating risk — prior Studio experience

Recorded by Marc from the Mac Studio deployment, and carried here as a warning,
not as a measurement of tyr-ops:

- Wazuh plus other heavy stacks **contested the Studio's ~16 GiB Docker VM**.
- Greenbone was **deferred on the Studio for RAM reasons** when it was to sit
  beside Wazuh.

That second point **raises the sizing bar for tyr-ops**: Greenbone is no
longer a future maybe that can be dropped if the host turns out tight. It is a
required member of the stack, so tyr-ops must be sized for Paperclip board
+ Vulcan seat + Wazuh manager + Greenbone **together**, from the start.

Capacity remains **unverified until measured** — the two points above describe
the Studio, not tyr-ops. **If headroom fails, STOP and escalate to Marc.**
Do not resolve a capacity failure by splitting the stack: not onto Mac2 (HARD
STOP 11), not by leaving the Wazuh manager on the Studio (HARD STOP 13), and not
by dropping Greenbone from scope (HARD STOP 12). The permitted responses are to
add resources to tyr-ops, or to re-plan with Marc.

### 0.4 Variables — fill before execution

```sh
# Old host (Hostinger VPS)
export VPS_SSH="root@<vps-tailnet-ip>"     # tailnet (100.64.0.0/10) address; take from the
                                            # cutover request or the vault. Do not commit it.
export VPS_COMPOSE_DIR=/opt/tyrx-paperclip

# New host (tyr-ops)
export OPS_SSH="<user>@<tyr-ops-address>"
export OPS_COMPOSE_DIR="<path-to-compose-project-on-tyr-ops>"

# Instance identity (same on both hosts)
export PAPERCLIP_HOME=/paperclip
export INSTANCE=default

# Code
export DEPLOY_BASE=deploy/tyr746-26df56cc
export CUTOVER_BRANCH=deploy/tyr-ops-cutover

# API base: the OLD host until Phase 9, then the new one
export API=https://paperclip.tyr-x.com
```

Board API calls need an operator token. Read it into the shell without echoing
it, and never place it in a command that gets committed or pasted:

```sh
read -rs PAPERCLIP_BOARD_TOKEN && export PAPERCLIP_BOARD_TOKEN
```

## Phase A — Discovery (no changes)

**Goal.** Replace every `<angle-bracket>` above with a verified value. Nothing in
this runbook may be executed until this table is complete.

| # | Question | Command / source | Value |
| --- | --- | --- | --- |
| A1 | tyr-ops reachable address and SSH user | `ssh $OPS_SSH 'hostname; uname -a'` | |
| A2 | OS, arch, CPU, RAM, free disk | `ssh $OPS_SSH 'uname -m; free -h; df -h'` | |
| A3 | Docker + Compose versions present | `ssh $OPS_SSH 'docker version; docker compose version'` | |
| A4 | Node and pnpm available (24.11+ / 9.15.4) | `ssh $OPS_SSH 'node -v; pnpm -v'` | |
| A5 | Is tyr-ops on the same tailnet as the Mac and the VPS? | `ssh $OPS_SSH 'tailscale status'` — see `docs/deploy/tailscale-private-access.md` | |
| A6 | Can tyr-ops reach the Mac execution host at all? | `ssh $OPS_SSH 'nc -z <mac-address> 22; echo $?'` | |
| A7 | Who terminates TLS for `paperclip.tyr-x.com` on the new host (Traefik? same compose?) and how are certs issued | inspect `$OPS_COMPOSE_DIR` / DNS provider | |
| A8 | DNS provider, current record type and TTL for `paperclip.tyr-x.com` | DNS console | |
| A9 | **DB mode on the VPS**: embedded or external `DATABASE_URL` | Phase 3 step 3.1 | |
| A10 | Backup destination with room for a full logical dump | `ssh $OPS_SSH 'df -h <backup-path>'` | |
| A11 | Vault of record for this cutover — Marc said 1Password; the Jev secret lives in **Proton Pass** today. Pick one and record it | Marc | |
| A12 | **Headroom on tyr-ops for the whole stack — Paperclip board + Vulcan seat + Wazuh manager + Greenbone, together:** total and currently free RAM, disk, and CPU, plus the container runtime's own memory ceiling if it runs in a VM. Greenbone is required scope, so it counts against this budget from the start | `ssh $OPS_SSH 'nproc; free -g; df -h; docker info --format "{{.MemTotal}} {{.NCPU}}"'` | |
| A13 | **Does the Wazuh manager fit beside the board?** Measure the board's steady-state and peak footprint on the new host after Phase 8, then compare the remainder against the Studio manager's measured footprint (indexer, manager, dashboard, and its data volume growth rate) | `docker stats --no-stream` on both hosts; `du -sh` on the Wazuh data volume on the Studio | |
| A14 | **Agent re-point path from the Studio manager:** how many agents are enrolled, how they are enrolled (enrollment password, keys, or manifest), how `ossec.conf` server address is managed per agent, and whether re-enrollment or a server-address change is required | Studio Wazuh manager: `/var/ossec/bin/agent_control -l` (or the containerised equivalent) and the agent config management path | |
| A15 | **Greenbone CE sizing — required scope, not optional:** RAM, disk, and feed-sync footprint it will add on tyr-ops, from `tyrxtech/tyrx-ai-security` and the Greenbone CE docs. This row cannot be closed as "defer, decide later"; it is part of the budget the host must satisfy | That repository plus upstream sizing guidance — not readable from this agent | |
| A16 | **Sequencing decision:** the order of the three workstreams — board cutover, Wazuh manager relocation off the Mac Studio, and the Greenbone CE first deployment. Record the chosen order, the owner of each, and the scheduled date of the Greenbone deployment. Any order is acceptable provided none of them is interleaved with the database and DNS phases, each carries its own Marc gate (G8, GS, GG), and the end state is **all four components co-located on tyr-ops** | Marc, with the measured numbers from A12 to A15 | |

**Capacity is unverified until measured.** A12 to A16 must contain real numbers
from the commands above, not estimates. Treat the prior Studio experience in
section 0.3 as a warning, not a measurement: Wazuh plus heavy stacks contested
the Studio's ~16 GiB Docker VM, and Greenbone was RAM-deferred beside Wazuh on
that host. That contention is exactly why the sizing bar for tyr-ops is
higher than it would be for the board alone — the host must carry all four
components at once, Greenbone included, because Greenbone is required scope and
cannot be dropped to make the numbers work.

**If headroom fails, STOP.** Escalate to Marc. Do not split the stack to make it
fit: not onto Mac2 (HARD STOP 11), not by leaving the Wazuh manager permanently
on the Studio (HARD STOP 13), and not by dropping Greenbone from scope (HARD
STOP 12). The permitted responses are to add resources to tyr-ops, or to
re-plan with Marc.

**Verify.** No blank cells. Three hard blockers: A6 — if tyr-ops cannot
reach the Mac, the cutover cannot complete; A12/A13/A15 — if the co-located
footprint of all four components does not fit, the program cannot be scheduled
as drawn; and A16 — without a recorded sequence and a Greenbone date, acceptance
has no path to a co-located end state.

**Gate G0.** Marc confirms the discovery table, the choice of vault (A11), the
measured capacity verdict for the full four-component stack (A12 to A15), and
the sequencing decision (A16).

## Phase 1 — Cut the code branch off the deploy branch

**Goal.** A cutover branch whose ancestry is the exact code running in
production — never `master` alone.

**Preconditions.** G0 passed.

**Steps.**

```sh
git fetch origin --prune
git switch -c "$CUTOVER_BRANCH" "origin/$DEPLOY_BASE"
```

**Verify.** All four checks must pass before you build anything:

```sh
# 1. Tip is the known production commit.
git log --oneline -1
#    expect: 26df56cc fix(heartbeat): include statusVersion in deferred-wake admission select

# 2. Divergence matches the locked inventory (11 ahead / 196 behind master).
git rev-list --count origin/master..HEAD   # expect 11
git rev-list --count HEAD..origin/master   # expect 196

# 3. The three fix families are ancestors of this branch.
for sha in 1de12604 0c6bc08e 432a5635 26df56cc; do
  git merge-base --is-ancestor $sha HEAD && echo "$sha OK" || echo "$sha MISSING -- STOP"
done
#    1de12604 TYR-668 block implicit reopen of blocked issues
#    0c6bc08e TYR-722 scheduler failure and liveness invariants
#    432a5635 TYR-746 suppress stale terminal comment wakes
#    26df56cc TYR-746 statusVersion in deferred-wake admission

# 4. You are NOT on master's lineage by accident.
git merge-base --is-ancestor origin/master HEAD && echo "STOP: branched off master" || echo "ok"
```

**Rollback.** Delete the local branch; nothing has been deployed.

**Gate G1.** Marc confirms the branch name and the four verification outputs.

## Phase 2 — First-boot drain flag

**Goal.** Guarantee the new host boots holding admission, so it cannot dispatch
queued runs or due retries while you are still verifying it.

**Preconditions.** G1 passed.

**Steps.**

```sh
# The flag must already exist on this lineage. It is NOT on master.
git grep -n "PAPERCLIP_TASK_DRAIN_ON_START" -- server/src | head
```

If that returns nothing, **STOP**: the branch is wrong, or the drain-on-start
commits (`d3d57d5d`, `5751f8a6`) are missing. Do not "fix" this by merging
`master`.

Set the flag in the new host's environment file — in place, as the file, not on
a command line:

```sh
ssh $OPS_SSH
# then, in an editor, in $PAPERCLIP_HOME/instances/$INSTANCE/.env :
#   PAPERCLIP_TASK_DRAIN_ON_START=true
chmod 600 $PAPERCLIP_HOME/instances/$INSTANCE/.env
```

**Verify.** After the new host first boots (Phase 8) the drain must report
active. Do not release it here.

**Rollback.** Remove the line; the default is off.

**Gate G2.** Marc confirms the flag is present on the cutover branch and set in
the new host's `.env`.

## Phase 3 — DB mode, secrets, and backup

**Goal.** Know which database mode is live, get a restorable backup, and move
secrets through a vault without ever writing them into git or chat.

**Preconditions.** G2 passed. Vault chosen (A11).

### 3.1 Confirm the DB mode (names only, never values)

Paperclip starts an embedded PostgreSQL when `DATABASE_URL` is unset
(`docs/deploy/database.md`). Determine which mode the VPS is in by listing
variable **names** only:

```sh
ssh $VPS_SSH "cd $VPS_COMPOSE_DIR && docker compose ps"
ssh $VPS_SSH "cd $VPS_COMPOSE_DIR && docker compose exec -T <server-service> printenv | cut -d= -f1 | sort | grep -E 'DATABASE|PAPERCLIP_HOME|PAPERCLIP_INSTANCE|DB_BACKUP'"
```

- `DATABASE_URL` present ⇒ **external** Postgres. Record where it points (host
  only, from the vault copy — do not print the URL).
- `DATABASE_URL` absent ⇒ **embedded** Postgres living on the VPS disk under
  `PAPERCLIP_HOME`. The data must be dumped and restored; it cannot be shared.

Record the answer in A9. The remaining phases branch on it.

### 3.2 Take a logical backup

```sh
ssh $VPS_SSH "cd $VPS_COMPOSE_DIR && docker compose exec -T <server-service> \
  sh -lc 'PAPERCLIP_DB_BACKUP_DIR=/paperclip/backups/cutover pnpm paperclipai db:backup'"
ssh $VPS_SSH "ls -lh $PAPERCLIP_HOME/backups/cutover"
```

Logical dumps cover non-system schemas, the Drizzle migration journal, and
plugin-owned schemas.

### 3.3 Collect what the dump does **not** contain

A database backup is not sufficient on its own. All of the following must travel
with it:

1. **The local encrypted secrets master key** —
   `$PAPERCLIP_HOME/instances/$INSTANCE/secrets/master.key`, mode `0600`.
   Restore requires *both* the database and this key; either artifact alone is
   useless, and user-scoped secret values stay undecryptable without it. If
   `PAPERCLIP_SECRETS_MASTER_KEY` or `PAPERCLIP_SECRETS_MASTER_KEY_FILE` is set
   instead, carry that value/path.
2. **`$PAPERCLIP_HOME/instances/$INSTANCE/.env`** (`0600`) — the hand-carried
   instance configuration and secrets.
3. **Local-disk uploads and workspace files** under `PAPERCLIP_HOME`, which
   logical dumps exclude.
4. **`config.json`** for the instance (`secrets.localEncrypted.keyFilePath`).

### 3.4 Move them through the vault

```sh
# On the VPS: encrypt before anything leaves the host.
ssh $VPS_SSH "cd $PAPERCLIP_HOME && tar -czf - instances/$INSTANCE/.env \
  instances/$INSTANCE/secrets/master.key instances/$INSTANCE/config.json \
  | age -r <age-recipient-public-key> > /root/cutover-secrets.tar.gz.age"
```

Then either store `cutover-secrets.tar.gz.age` as a vault attachment (A11) and
pull it down on tyr-ops, or transfer it host-to-host and decrypt in place.
Shred the plaintext on both ends when done.

**Never land in the repo, a PR, a Paperclip comment, or a chat message:**
`master.key`, any `.env`, `DATABASE_URL`, `PAPERCLIP_SECRETS_MASTER_KEY`,
`TYPESAFE_API_KEY`, board/agent API tokens, SSH private keys, the age/gpg
recipient's private key, or any dump file.

### 3.5 Rehearse the restore — this path is not productized

There is **no `db:restore` command** in this tree. `paperclipai db:backup`
produces a logical dump; restoring it is a standard PostgreSQL logical restore
into the target database, and in embedded mode you must first establish how to
reach the embedded instance on tyr-ops. Treat this as the highest-risk step
in the cutover: rehearse it on a scratch instance on tyr-ops (fresh
`PAPERCLIP_HOME`, different instance id) and confirm the restored instance
starts, reports healthy, and can decrypt a known secret with the restored
master key.

**Verify.** Backup file exists with a plausible size; encrypted bundle decrypts
on tyr-ops; rehearsal instance boots healthy and decrypts a known secret.

**Rollback.** Nothing on the live host has changed. Delete the rehearsal
instance and shred transferred artifacts.

**Gate G3.** Marc confirms DB mode, backup integrity, and a successful restore
rehearsal. **This gate is mandatory — do not proceed on an unrehearsed restore.**

## Phase 4 — SSH identity for the new control plane

**Goal.** The new host can open an SSH session to the Mac execution host under
strict host-key checking, and the Mac accepts it.

**Preconditions.** G3 passed. A6 confirmed reachability.

**Steps.**

```sh
# 1. On tyr-ops: generate a control-plane keypair (do not reuse the VPS key).
ssh $OPS_SSH 'ssh-keygen -t ed25519 -C "paperclip-control-plane@tyr-ops" -f ~/.ssh/paperclip_exec -N ""'
ssh $OPS_SSH 'cat ~/.ssh/paperclip_exec.pub'

# 2. On the Mac: append that public key to the execution user's authorized_keys.
#    Do NOT remove the VPS key yet — it is the rollback path.
#    (append the pubkey line, then:)
#    chmod 600 ~/.ssh/authorized_keys

# 3. Capture the Mac host key for the execution spec's knownHosts.
ssh $OPS_SSH 'ssh-keyscan -t ed25519 <mac-address>'
```

Update the Paperclip execution spec on the new host with the new private key and
the captured `knownHosts` entry, keeping `strictHostKeyChecking` enabled.

**Verify.**

```sh
# Non-mutating probe from the new host, strict checking on:
ssh $OPS_SSH 'ssh -i ~/.ssh/paperclip_exec -o StrictHostKeyChecking=yes \
  -o UserKnownHostsFile=~/.ssh/known_hosts <mac-user>@<mac-address> "sw_vers; uname -m; whoami"'
```

A host-key prompt or `REMOTE HOST IDENTIFICATION` error means `knownHosts` is
wrong — fix the spec, do not disable strict checking.

**Rollback.** Remove the new public key from the Mac's `authorized_keys`. The
VPS key is still in place, so the old host keeps working.

**Gate G4.** Marc confirms the probe output and that the VPS key is still
present for rollback.

## Phase 5 — Open PR decisions (merge or carry)

**Goal.** An explicit, recorded decision for every open change, relative to the
deploy branch — not to `master`.

**Preconditions.** G4 passed.

| Item | State | Decision required | Recommended |
| --- | --- | --- | --- |
| PR #5 `fix/ssh-sync-skip-node-modules` | Open, based on the **superseded** `deploy/tyr722-0c6bc08e` | Rebase onto `$DEPLOY_BASE` and cherry-pick into `$CUTOVER_BRANCH`, **or** defer | **Carry.** Without it the ~2 h copy-back per Codex run follows the move. Before carrying, run the bsdtar exclude check on the Mac (below) — the saving depends on it. Independent review found no blocking gaps |
| PR #6 Node/pnpm docs | Draft, docs only | Merge or treat as host prerequisite | **Treat as prerequisite** regardless: build tyr-ops on Node 24.11+ and pnpm 9.15.4, and expect `pnpm install --frozen-lockfile` to fail on the acpx patch-hash mismatch until a plain `pnpm install` rewrites it. Do not commit a rewritten lockfile |
| PR #2 drain-on-start | Open against `master`, already live on the deploy branch | Keep carrying on the deploy lineage | **Carry.** Do not merge to `master` during the window |
| `cursor/ttl-native-restart-recovery-f723` | No PR; closes the TTL native-restart hole | Carry into `$CUTOVER_BRANCH` or accept the gap | **Decide explicitly.** If you will arm a TTL-bounded drain anywhere in this cutover, carry it. If every drain is indefinite, the gap is not on the cutover path — record that choice |

bsdtar check for the PR #5 decision, run on the Mac execution host (read-only):

```sh
cd <a-representative-workspace> && \
  tar --exclude node_modules --exclude '*/node_modules' -cf - . | wc -c && du -sk .
```

If the archive is not dramatically smaller than `du`, the macOS `tar` is not
honouring the patterns at depth and PR #5 needs the four-form exclude set
before it buys anything.

**Verify.** Each row has a decision and, where "carry" was chosen, the commit is
an ancestor of `$CUTOVER_BRANCH` (`git merge-base --is-ancestor <sha> HEAD`).

**Rollback.** Reset `$CUTOVER_BRANCH` to `origin/$DEPLOY_BASE` and rebuild.

**Gate G5.** Marc signs off each row, including explicit "defer" decisions.

## Phase 6 — Single-writer plan for the database

**Goal.** Make it structurally impossible for both hosts to admit work against
the same database.

**Preconditions.** G5 passed. A9 recorded.

**Why the drain is not enough.** The task-drain status is **per process**:
`GET /api/instance/task-drain` reports in-process work only, and a process
restart clears the drain even while the database still holds running rows. A
drain on the old host therefore protects nothing against a *second* server
process pointed at the same database.

**If embedded (A9 = embedded).** The database is local to each host and cannot
be shared, so there is no dual-writer path — but the trade is a frozen window:
the dump taken in Phase 7 is the last state the new host will have. Nothing
written on the VPS after that dump survives the move.

**If external (A9 = `DATABASE_URL`).** Choose one, before the new host admits
work:

1. **Credential revocation (preferred).** Issue a new DB role for tyr-ops;
   after Phase 7 drain, revoke or rotate the VPS role so the old host cannot
   write even if it restarts.
2. **Network isolation.** Firewall the database to accept connections only from
   tyr-ops once the drain is verified quiet.

Record which mechanism, who executes it, and how it is verified.

**Verify.** Demonstrate that the old host *cannot* write: after the chosen
mechanism is applied, the VPS server's `/api/health` must report a database
error (`database_unavailable` / `database_unreachable`) rather than healthy.

**Rollback.** Re-grant the VPS role or reopen the firewall rule; this is the
primary rollback lever for Phases 8–9.

**Gate G6.** Marc approves the single-writer mechanism and the rollback lever.

## Phase 7 — Drain the old host and verify the queue is quiet

**Goal.** No new work admitted and no runs in flight on the VPS before the
final dump.

**Preconditions.** G6 passed.

**Steps.**

```sh
# Arm an INDEFINITE drain. Omit ttlMs on purpose: a TTL can expire mid-cutover.
curl -sS -X POST "$API/api/instance/task-drain" \
  -H "Authorization: Bearer $PAPERCLIP_BOARD_TOKEN" \
  -H 'Content-Type: application/json' -d '{}'

# Poll until quiet.
watch -n 15 "curl -sS '$API/api/instance/task-drain' \
  -H 'Authorization: Bearer \$PAPERCLIP_BOARD_TOKEN'"
```

**Do not restart or recycle the VPS server after this point** (HARD STOP #3) —
a restart clears the in-memory drain.

**Verify.**

- `GET /api/instance/task-drain` reports `draining: true` and quiescent counts
  at zero.
- `GET /api/health` on the VPS still reports `status` healthy with the expected
  `version`, and `databaseBackup` has no stale/missing warnings.
- No running heartbeat rows remain in the database (confirm by query; the drain
  status is in-process only and will not tell you this).
- Take the **final** dump now, repeating Phase 3.2, and re-transfer the bundle.
  This dump — not the rehearsal one — is what gets restored.

**Rollback.** `DELETE /api/instance/task-drain` releases the hold and resumes
withheld native restart recovery and queued dispatch on the VPS. This is a clean
rollback as long as the single-writer mechanism from Phase 6 has not yet been
applied and the new host has not admitted work.

**Gate G7.** Marc confirms quiet queue and a good final dump.

## Phase 8 — Bring up tyr-ops (drained) and verify

**Goal.** The new host is fully healthy, still holding admission, and proven on
one real run before any traffic moves.

**Preconditions.** G7 passed. Final dump restored per Phase 3.5. Phase 6
mechanism applied so the VPS can no longer write.

**Steps.**

```sh
ssh $OPS_SSH "cd $OPS_COMPOSE_DIR && docker compose up -d"
ssh $OPS_SSH "cd $OPS_COMPOSE_DIR && docker compose ps"
```

**Verify — in this order.**

```sh
# 1. Health, version, and backup health. Call the host directly, not via DNS.
curl -sS http://<tyr-ops-address>:<port>/api/health | jq '{status, version, databaseBackup}'
#    A 503 {"error":"database_unavailable"} or status "unhealthy" with
#    "database_unreachable" means the restore or DATABASE_URL is wrong -- STOP.

# 2. The drain is ACTIVE from first boot (Phase 2 flag).
curl -sS http://<tyr-ops-address>:<port>/api/instance/task-drain \
  -H "Authorization: Bearer $PAPERCLIP_BOARD_TOKEN"
#    expect draining: true, source startup_env. If it is not draining, STOP:
#    the host was built without the flag and may already be dispatching.

# 3. Restart bookkeeping is sane.
ssh $OPS_SSH "cat $PAPERCLIP_HOME/instances/$INSTANCE/hot-restart-report.json" | \
  jq '{previousServerVersion, newServerVersion, drainReason, adoptedRunIds, lostRunIds}'

# 4. Data spot-checks: companies, projects, and agents all present.
curl -sS http://<tyr-ops-address>:<port>/api/companies \
  -H "Authorization: Bearer $PAPERCLIP_BOARD_TOKEN" | jq 'length'

# 5. Secrets decrypt (proves master.key travelled correctly) -- check that a
#    known secret-bound agent or routine resolves, WITHOUT printing any value.
```

**Concurrent-run safety.** Only after checks 1–5 pass, and under Marc's gate,
release the drain and watch exactly one run:

```sh
curl -sS -X DELETE "http://<tyr-ops-address>:<port>/api/instance/task-drain" \
  -H "Authorization: Bearer $PAPERCLIP_BOARD_TOKEN"
```

Then dispatch **one** agent task that uses the SSH path to the Mac and confirm:
the run starts, the workspace syncs to the Mac, the agent executes, the restore
comes back, and the task reaches a terminal state. Watch the copy-back duration
— it is the direct measure of the Phase 5 PR #5 decision. Keep concurrency at
one until that run completes cleanly.

**Rollback.** Re-arm the drain on the new host, reverse the Phase 6 mechanism
(re-grant the VPS role / reopen the firewall), release the drain on the VPS. DNS
has not moved, so the VPS is still serving.

**Gate G8.** Marc confirms all health checks and the single clean end-to-end run.

## Phase 9 — Flip DNS last

**Goal.** Move `paperclip.tyr-x.com` to tyr-ops only after the new host has
proven itself.

**Preconditions.** G8 passed.

**Steps.**

1. **In advance** (can be done during Phase A): lower the TTL on
   `paperclip.tyr-x.com` to 60 s so rollback propagates quickly.
2. Confirm the new host's TLS path (A7): Traefik is routing
   `paperclip.tyr-x.com` and a certificate is issued or issuable.
3. Flip the DNS record to tyr-ops.
4. Watch propagation; do not stop the VPS container.

**Verify.**

```sh
dig +short paperclip.tyr-x.com
curl -sS https://paperclip.tyr-x.com/api/health | jq '{status, version}'
#    version must match the cutover-branch build, not the old VPS build.
curl -sSI https://paperclip.tyr-x.com/ | head -5   # TLS + Traefik routing
```

Then confirm from the board UI: sign in, open a company, open a task thread,
post a comment, and confirm an agent wake is queued.

**Rollback.** Flip DNS back to the VPS, re-arm the drain on tyr-ops,
reverse Phase 6, release the drain on the VPS. **Point of no return:** once the
new host has admitted and completed work *after* the DNS flip, the VPS database
is stale; rolling back then means dumping from tyr-ops and restoring onto
the VPS, or accepting the loss of everything written since the Phase 7 dump.

**Gate G9.** Marc confirms the flip and the post-flip board checks.

## Phase 10 — Watch window and decommission

**Goal.** Keep the rollback path alive long enough to trust the move.

**Steps and verification.**

1. Keep the VPS powered, drained, and unable to write to the database for the
   full watch window (recommend one full business day plus one scheduled-routine
   cycle, so at least one routine and one scheduled retry fire on the new host).
2. Monitor on tyr-ops: `/api/health` (`status`, `databaseBackup`), task
   drain status, run success rate, and copy-back duration per run.
3. Confirm a backup has run on the new host and is not stale.
4. Confirm the Mac's `authorized_keys` still contains both keys.
5. **Confirm the security layer is co-located** per section 0.3, for all four
   required components: the Paperclip board, Vulcan's seat and
   config/instructions, and the Wazuh manager all run on tyr-ops; every
   Wazuh agent reports to the tyr-ops manager; no security component runs
   on Mac2; and Greenbone CE is either already deployed on tyr-ops or has a
   scheduled deployment on tyr-ops with a named owner and a date (A16).
   Greenbone being both undeployed and unscheduled fails this step — see HARD
   STOP 13. Record the measured footprint against the A12, A13, and A15 numbers.

**Only after Marc's sign-off:** remove the VPS key from the Mac's
`authorized_keys`, archive the final VPS dump to the vault/cold storage, and
decommission `/opt/tyrx-paperclip`. Keep the `deploy/*` branches forever.

**Gate G10.** Marc authorizes decommission. **This gate cannot pass on a split
security layer.** If any component in section 0.3 still runs on another host,
either the relocation finishes first, or Marc records an explicit temporary-split
gate with a co-located end state, a named owner, and a review date. It also
cannot pass while Greenbone is both undeployed and unscheduled on tyr-ops,
or while the Wazuh manager remains permanently on the Mac Studio (HARD STOP 13).

## Appendix A — Marc's gate checklist

| Gate | Confirms |
| --- | --- |
| G0 | Discovery table complete; vault of record chosen; **measured** capacity verdict for the full four-component stack (board + Vulcan + Wazuh + Greenbone) on tyr-ops; sequencing recorded at A16 |
| G1 | Cutover branch off `deploy/tyr746-26df56cc`; 11/196 divergence; four fix SHAs are ancestors |
| G2 | `PAPERCLIP_TASK_DRAIN_ON_START` present on the branch and set on the new host |
| G3 | DB mode known; backup taken; **restore rehearsed successfully** |
| G4 | New host → Mac SSH probe passes under strict host-key checking; VPS key retained |
| G5 | Explicit merge-or-carry decision for #5, #6, #2, and the TTL branch |
| G6 | Single-writer mechanism and its rollback lever approved |
| G7 | Old host drained, queue quiet, final dump good |
| G8 | New host healthy, drained at boot, one clean end-to-end run |
| G9 | DNS flipped; board checks pass on the new host |
| GS | **Wazuh relocation** (per the A16 order): Wazuh manager moved to tyr-ops, every agent re-pointed and reporting, Studio manager retired, nothing on Mac2 |
| GG | **Greenbone first deploy** (per the A16 order): Greenbone CE stood up on tyr-ops and on no other host, within the measured A15 footprint. Marc authorizes the deployment; it must not be deployed without this gate |
| G10 | Watch window clean; **security layer verified co-located** (section 0.3) — board, Vulcan, Wazuh, and Greenbone deployed-or-scheduled on tyr-ops; decommission authorized |

Runbook gates are local to this document. They do not renumber or replace any
TYR-677 gate, and they do not reopen the locked G12 Vulcan-active decision.

## Appendix B — Quick command reference

```sh
# Drain status / arm / release
curl -sS      "$API/api/instance/task-drain" -H "Authorization: Bearer $PAPERCLIP_BOARD_TOKEN"
curl -sS -X POST   "$API/api/instance/task-drain" -H "Authorization: Bearer $PAPERCLIP_BOARD_TOKEN" \
     -H 'Content-Type: application/json' -d '{}'
curl -sS -X DELETE "$API/api/instance/task-drain" -H "Authorization: Bearer $PAPERCLIP_BOARD_TOKEN"

# Health
curl -sS "$API/api/health" | jq '{status, version, databaseBackup}'

# Manual backup
pnpm paperclipai db:backup     # honours PAPERCLIP_DB_BACKUP_DIR
```

Related docs: `docs/deploy/database.md`, `docs/deploy/secrets.md`,
`docs/deploy/docker.md`, `docs/deploy/tailscale-private-access.md`,
`docs/deploy/dev-plane-restart-hygiene.md`, `doc/DATABASE.md` (backups and the
local encrypted secrets provider), `doc/DEVELOPING.md` (backup configuration and
hot-restart reporting).

## Appendix C — Known gaps in this draft

1. Every `<angle-bracket>` value is unverified; Phase A exists to close them.
2. The restore path is not productized (no `db:restore`), so Phase 3.5 is a
   rehearsal, not a scripted step.
3. Whether tyr-ops can reach the Mac execution host (A6) is unconfirmed and
   is a hard blocker.
4. The bsdtar behaviour that PR #5's benefit depends on is unverified on the Mac;
   Phase 5 includes the one-command check.
5. Vault of record is ambiguous (1Password per instruction vs Proton Pass in
   current practice) and must be settled at G0.
6. This document must land in `tyrxtech/paperclip` on a branch cut from
   `deploy/tyr746-26df56cc`, not on `master`.
7. **Capacity for the co-located security layer is unverified until measured.**
   Rows A12 to A16 are open. The only data points on record are from the Mac
   Studio — Wazuh contested a ~16 GiB Docker VM and Greenbone was RAM-deferred
   beside it — and they do not describe tyr-ops. Because Greenbone is
   required scope rather than a future maybe, that Studio contention raises the
   sizing bar here: the host is budgeted for all four components at once, and a
   headroom failure is a STOP, not a reason to split.
8. `tyrxtech/tyrx-ai-security` is not readable from this agent's credentials, so
   the Greenbone CE footprint in A15 has to be filled in by someone with access.
   That makes A15 the weakest number in the capacity verdict even though
   Greenbone is required scope. This runbook asserts only the target host, which
   Marc locked.
9. The Wazuh relocation and agent re-point procedure is named and gated here
   (gate GS) but not yet written as steps. It needs its own section once A13 and
   A14 are answered, including the rollback: keep the Studio manager reachable
   until every agent reports to tyr-ops.
10. The Greenbone CE first deployment is likewise named and gated (gate GG) but
    not written as steps. It needs its own section once A15 and A16 are
    answered. Nothing in this draft authorizes deploying it.
