# Cutover runbook — live Paperclip: Hostinger VPS → tyr-dash-ops (DRAFT)

Status: **DRAFT — not authorized to execute.** Every phase ends in a Marc gate.
Date drafted: 2026-10-01.
Scope: move the live Paperclip control plane from the Hostinger VPS
(`paperclip.tyr-x.com`, compose project at `/opt/tyrx-paperclip`) to
**tyr-dash-ops** (the tyr-ops / dash-ops target host).

This runbook does **not** move the macOS execution host. The Mac Studio keeps
running agent workloads; only the control plane that SSHes into it changes.

## 0. How to read this

- Phases are strictly ordered. Do not reorder; later phases assume earlier
  verifications passed.
- Every phase has **Goal → Preconditions → Steps → Verify → Rollback → Gate**.
- `GATE Gn` means stop and get Marc's explicit go-ahead before continuing.
- Anything written as `<angle-brackets>` is **unverified** and must be filled in
  during Phase A discovery. This runbook deliberately does not invent facts
  about tyr-dash-ops.

### 0.1 HARD STOP list — never do these during this cutover

1. **No Vulcan unpause.** Vulcan stays paused for the entire window.
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

### 0.3 Variables — fill before execution

```sh
# Old host (Hostinger VPS)
export VPS_SSH="root@<vps-tailnet-ip>"     # tailnet (100.64.0.0/10) address; take from the
                                            # cutover request or the vault. Do not commit it.
export VPS_COMPOSE_DIR=/opt/tyrx-paperclip

# New host (tyr-dash-ops)
export OPS_SSH="<user>@<tyr-dash-ops-address>"
export OPS_COMPOSE_DIR="<path-to-compose-project-on-tyr-dash-ops>"

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
| A1 | tyr-dash-ops reachable address and SSH user | `ssh $OPS_SSH 'hostname; uname -a'` | |
| A2 | OS, arch, CPU, RAM, free disk | `ssh $OPS_SSH 'uname -m; free -h; df -h'` | |
| A3 | Docker + Compose versions present | `ssh $OPS_SSH 'docker version; docker compose version'` | |
| A4 | Node and pnpm available (24.11+ / 9.15.4) | `ssh $OPS_SSH 'node -v; pnpm -v'` | |
| A5 | Is tyr-dash-ops on the same tailnet as the Mac and the VPS? | `ssh $OPS_SSH 'tailscale status'` — see `docs/deploy/tailscale-private-access.md` | |
| A6 | Can tyr-dash-ops reach the Mac execution host at all? | `ssh $OPS_SSH 'nc -z <mac-address> 22; echo $?'` | |
| A7 | Who terminates TLS for `paperclip.tyr-x.com` on the new host (Traefik? same compose?) and how are certs issued | inspect `$OPS_COMPOSE_DIR` / DNS provider | |
| A8 | DNS provider, current record type and TTL for `paperclip.tyr-x.com` | DNS console | |
| A9 | **DB mode on the VPS**: embedded or external `DATABASE_URL` | Phase 3 step 3.1 | |
| A10 | Backup destination with room for a full logical dump | `ssh $OPS_SSH 'df -h <backup-path>'` | |
| A11 | Vault of record for this cutover — Marc said 1Password; the Jev secret lives in **Proton Pass** today. Pick one and record it | Marc | |

**Verify.** No blank cells. A6 is a hard blocker: if tyr-dash-ops cannot reach
the Mac, the cutover cannot complete.

**Gate G0.** Marc confirms the discovery table and the choice of vault (A11).

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
pull it down on tyr-dash-ops, or transfer it host-to-host and decrypt in place.
Shred the plaintext on both ends when done.

**Never land in the repo, a PR, a Paperclip comment, or a chat message:**
`master.key`, any `.env`, `DATABASE_URL`, `PAPERCLIP_SECRETS_MASTER_KEY`,
`TYPESAFE_API_KEY`, board/agent API tokens, SSH private keys, the age/gpg
recipient's private key, or any dump file.

### 3.5 Rehearse the restore — this path is not productized

There is **no `db:restore` command** in this tree. `paperclipai db:backup`
produces a logical dump; restoring it is a standard PostgreSQL logical restore
into the target database, and in embedded mode you must first establish how to
reach the embedded instance on tyr-dash-ops. Treat this as the highest-risk step
in the cutover: rehearse it on a scratch instance on tyr-dash-ops (fresh
`PAPERCLIP_HOME`, different instance id) and confirm the restored instance
starts, reports healthy, and can decrypt a known secret with the restored
master key.

**Verify.** Backup file exists with a plausible size; encrypted bundle decrypts
on tyr-dash-ops; rehearsal instance boots healthy and decrypts a known secret.

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
# 1. On tyr-dash-ops: generate a control-plane keypair (do not reuse the VPS key).
ssh $OPS_SSH 'ssh-keygen -t ed25519 -C "paperclip-control-plane@tyr-dash-ops" -f ~/.ssh/paperclip_exec -N ""'
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
| PR #6 Node/pnpm docs | Draft, docs only | Merge or treat as host prerequisite | **Treat as prerequisite** regardless: build tyr-dash-ops on Node 24.11+ and pnpm 9.15.4, and expect `pnpm install --frozen-lockfile` to fail on the acpx patch-hash mismatch until a plain `pnpm install` rewrites it. Do not commit a rewritten lockfile |
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

1. **Credential revocation (preferred).** Issue a new DB role for tyr-dash-ops;
   after Phase 7 drain, revoke or rotate the VPS role so the old host cannot
   write even if it restarts.
2. **Network isolation.** Firewall the database to accept connections only from
   tyr-dash-ops once the drain is verified quiet.

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

## Phase 8 — Bring up tyr-dash-ops (drained) and verify

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
curl -sS http://<tyr-dash-ops-address>:<port>/api/health | jq '{status, version, databaseBackup}'
#    A 503 {"error":"database_unavailable"} or status "unhealthy" with
#    "database_unreachable" means the restore or DATABASE_URL is wrong -- STOP.

# 2. The drain is ACTIVE from first boot (Phase 2 flag).
curl -sS http://<tyr-dash-ops-address>:<port>/api/instance/task-drain \
  -H "Authorization: Bearer $PAPERCLIP_BOARD_TOKEN"
#    expect draining: true, source startup_env. If it is not draining, STOP:
#    the host was built without the flag and may already be dispatching.

# 3. Restart bookkeeping is sane.
ssh $OPS_SSH "cat $PAPERCLIP_HOME/instances/$INSTANCE/hot-restart-report.json" | \
  jq '{previousServerVersion, newServerVersion, drainReason, adoptedRunIds, lostRunIds}'

# 4. Data spot-checks: companies, projects, and agents all present.
curl -sS http://<tyr-dash-ops-address>:<port>/api/companies \
  -H "Authorization: Bearer $PAPERCLIP_BOARD_TOKEN" | jq 'length'

# 5. Secrets decrypt (proves master.key travelled correctly) -- check that a
#    known secret-bound agent or routine resolves, WITHOUT printing any value.
```

**Concurrent-run safety.** Only after checks 1–5 pass, and under Marc's gate,
release the drain and watch exactly one run:

```sh
curl -sS -X DELETE "http://<tyr-dash-ops-address>:<port>/api/instance/task-drain" \
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

**Goal.** Move `paperclip.tyr-x.com` to tyr-dash-ops only after the new host has
proven itself.

**Preconditions.** G8 passed.

**Steps.**

1. **In advance** (can be done during Phase A): lower the TTL on
   `paperclip.tyr-x.com` to 60 s so rollback propagates quickly.
2. Confirm the new host's TLS path (A7): Traefik is routing
   `paperclip.tyr-x.com` and a certificate is issued or issuable.
3. Flip the DNS record to tyr-dash-ops.
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

**Rollback.** Flip DNS back to the VPS, re-arm the drain on tyr-dash-ops,
reverse Phase 6, release the drain on the VPS. **Point of no return:** once the
new host has admitted and completed work *after* the DNS flip, the VPS database
is stale; rolling back then means dumping from tyr-dash-ops and restoring onto
the VPS, or accepting the loss of everything written since the Phase 7 dump.

**Gate G9.** Marc confirms the flip and the post-flip board checks.

## Phase 10 — Watch window and decommission

**Goal.** Keep the rollback path alive long enough to trust the move.

**Steps and verification.**

1. Keep the VPS powered, drained, and unable to write to the database for the
   full watch window (recommend one full business day plus one scheduled-routine
   cycle, so at least one routine and one scheduled retry fire on the new host).
2. Monitor on tyr-dash-ops: `/api/health` (`status`, `databaseBackup`), task
   drain status, run success rate, and copy-back duration per run.
3. Confirm a backup has run on the new host and is not stale.
4. Confirm the Mac's `authorized_keys` still contains both keys.

**Only after Marc's sign-off:** remove the VPS key from the Mac's
`authorized_keys`, archive the final VPS dump to the vault/cold storage, and
decommission `/opt/tyrx-paperclip`. Keep the `deploy/*` branches forever.

**Gate G10.** Marc authorizes decommission.

## Appendix A — Marc's gate checklist

| Gate | Confirms |
| --- | --- |
| G0 | Discovery table complete; vault of record chosen |
| G1 | Cutover branch off `deploy/tyr746-26df56cc`; 11/196 divergence; four fix SHAs are ancestors |
| G2 | `PAPERCLIP_TASK_DRAIN_ON_START` present on the branch and set on the new host |
| G3 | DB mode known; backup taken; **restore rehearsed successfully** |
| G4 | New host → Mac SSH probe passes under strict host-key checking; VPS key retained |
| G5 | Explicit merge-or-carry decision for #5, #6, #2, and the TTL branch |
| G6 | Single-writer mechanism and its rollback lever approved |
| G7 | Old host drained, queue quiet, final dump good |
| G8 | New host healthy, drained at boot, one clean end-to-end run |
| G9 | DNS flipped; board checks pass on the new host |
| G10 | Watch window clean; decommission authorized |

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
3. Whether tyr-dash-ops can reach the Mac execution host (A6) is unconfirmed and
   is a hard blocker.
4. The bsdtar behaviour that PR #5's benefit depends on is unverified on the Mac;
   Phase 5 includes the one-command check.
5. Vault of record is ambiguous (1Password per instruction vs Proton Pass in
   current practice) and must be settled at G0.
6. This document must land in `tyrxtech/paperclip` on a branch cut from
   `deploy/tyr746-26df56cc`, not on `master`.
