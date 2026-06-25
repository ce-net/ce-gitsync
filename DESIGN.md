# ce-gitsync — real-time git sync over the CE mesh

**Goal (Leif's words):** real-time, invisible, safe, easy to monitor. *"I will work and never think
about it — just being able to switch machines during development is awesome."* Plus: when a device
was **shut down**, on reconnect we must sync **and see what was done in the meantime** (→ git is the
substrate, because git history *is* that answer).

The substrate is **git** (durable history, offline catch-up, recover-anything, standard merge). The
**CE mesh** is the transport between the two devices' repos — no GitHub round-trip, fully P2P.

## Why git, not raw file-sync (rdev)
rdev re-walks the whole tree every reconcile (~1 file/sec over the relay) → can't be real-time at
scale, and silently overwrites. Git knows *exactly* what changed (no re-walk), transfers only deltas,
keeps full history (nothing lost), and answers "what happened while I was offline" for free.

---

## Option B — IMPLEMENTED: rdev/ce-gitsync ships commits as bundles over the blob store

Reuses the plumbing that already works (the node's content-addressed blob store + mesh messaging).
Mesh-native, no cap needed (git ops are local; peers are authenticated by node id + ce-link enrollment).

**Per git repo, on each device, a daemon:**
1. **Auto-commit** local changes on a file-watch/poll (`git add -A && git commit -m "live: <host> <ts>"`).
2. **Push delta:** bundle `<last-known-peer-head>..HEAD` (full `--all` on first contact) →
   `POST /blobs` → announce `{repo, head, bundle_cid}` to the peer via `/mesh/send`.
3. **Receive:** `GET /blobs/:cid` (mesh-fetched) → `git fetch <bundle>` → `git merge --no-edit`
   (or rebase) → record peer head in `refs/ce-gitsync/<peer>` → ack `{repo, my_head}`.
4. **Conflicts:** a true git merge conflict is preserved (both sides in history + a
   `ce-gitsync/conflict-<ts>` branch); the daemon takes the newest for the working tree and logs it.
   Never blocks — recover via git. (Rare: work is sequential when *switching* machines.)

**Monitor:** `ce-gitsync log` / `status` streams synced commits + conflicts per machine.
**Offline catch-up:** on reconnect the periodic sync pulls the peer's delta; `ce-gitsync log` shows
exactly the commits made while this device was down.

**Transport detail:** announce/ack/head-request ride `/mesh/send` + `/mesh/messages` (peer-id
authenticated, enrollment-checked like `clip`). Bundles ride `POST /blobs` + `GET /blobs/:cid`
(content address = capability; the CID is shared only over the authenticated channel).

**Limitations (acceptable for v1):** poll-based real-time (~2-3 s; fswatch upgrade later); merge
conflicts take newest-for-worktree + preserve both (no interactive resolve, by design — zero-think).

---

## Option A — FUTURE: git-native remote over the mesh

Cleaner end-state: expose each repo as a **git remote** of the other over the mesh, so plain
`git fetch mesh:debian` / `git push mesh:laptop` work, and all of git's tooling (log, diff, bisect,
mergetool) operates directly across machines.

**Build:** a `git-remote-ce` helper (git invokes `git-remote-ce mesh://<node-id>/<repo>`). It speaks
git's smart-transport (`git-upload-pack`/`git-receive-pack`) by tunnelling pkt-line over the mesh:
- rdev `serve` gains a `git/upload-pack` + `git/receive-pack` verb that runs the corresponding git
  plumbing on the host repo and streams stdin/stdout over an `ce-rs` mesh **stream** (not request/
  reply — git transport is bidirectional streaming).
- The remote helper connects the local `git fetch/push` to that stream, capability-gated by the same
  ce-cap chain rdev already uses (a `git` ability + a `repo_prefix` caveat).

**Why later:** needs a streaming (not bundle) transport + the remote-helper protocol; B delivers the
same user value (real-time + history + offline-catch-up) sooner by reusing blobs. A is the refactor
once B proves the workflow.

**Migration B→A:** the daemon's auto-commit + monitor + conflict policy stay; only the transport
swaps (bundle-over-blob → live stream). Refs (`refs/ce-gitsync/<peer>`) carry over.
