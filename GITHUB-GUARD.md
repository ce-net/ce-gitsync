# ce-gitsync GitHub guard

**Policy: keep GitHub in sync with local — but cleanly.** Push whenever you have new
work; don't let GitHub fall behind. The one thing GitHub must never carry is the
ce-gitsync `live: ...` WIP snapshot commits — so before each push, scrub those out
(keep every real commit). Then sync the normal way: **`git pull` (merge converging
work) then `git push`.** GitHub is a shared remote — expect healthy convergence and
merge it. **Do NOT routinely force-push** (it clobbers others' work); reserve
`--force` for exceptional recovery. The guard below makes "cleanly" automatic.

ce-gitsync (`rdev gitsync`) syncs every git repo across your machines **over the CE
mesh + locally only** — it `git add -A && git commit -m "live: <host> ..."` on the
working branch and ships a delta **git bundle** over the reliable mesh message path
to every linked device, which fetch+merge it. It never pushes to GitHub. ce-hub's
git mirror is configured **inbound only** (`direction: in`), so it never pushes out
either.

GitHub commits appear only when a human or agent runs `git push` — which carries the
accumulated `live: ...` WIP commits up, polluting history and triggering CI on
half-finished states.

## The guard

A `pre-push` hook refuses to push any range containing a `ce-gitsync`-authored
commit (`gitsync@ce-net`) to a **GitHub** remote. Other remotes (the mesh, a relay,
a local bare repo) are unaffected. Result: the mesh is the sync path; GitHub only
receives deliberate, properly-authored commits.

For a **new ref** (a tag, or a brand-new branch) the hook inspects only commits not
already on the remote (`<sha> --not --remotes=<remote>`), so tagging a release at an
already-pushed HEAD never re-flags its `live:` ancestors — no `--no-verify` needed.

Installed:
- into every existing repo under `~/ce-net` (`.git/hooks/pre-push`), and
- via `git config --global init.templateDir ~/.config/git/template` so every future
  `git init` / `git clone` gets it automatically.

## Keeping GitHub in sync (cleanly)

Scrub the `gitsync@ce-net` WIP commits first so the pushed range has none, then push.
Two cleaning patterns:

**A — WIP only at the tip** (squash everything since the last real commit into one):
```
git reset --soft <last-real-commit>     # or origin/main
git -c user.name="Leif Rydenfalk" -c user.email="ledamecrydenfalk@gmail.com" \
    commit --author="Leif Rydenfalk <ledamecrydenfalk@gmail.com>" -m "..."
git push origin <branch>
```

**B — WIP interleaved with real commits** (drop only the gitsync commits, keep every
real one): rebuild the branch by re-snapshotting each kept (non-gitsync) commit's tree
onto a fresh linear history — each kept commit's tree already subsumes the WIP snapshots
before it, so nothing is lost and there are no conflicts. Verify
`git diff <old-HEAD> <new-HEAD>` is empty before moving the branch.

Then sync normally — pull/merge first, then push:
```
git pull --rebase origin <branch>       # converge with what's already on GitHub
git push origin <branch>
```
Either way the pushed range has zero `gitsync@ce-net` commits, so the guard passes with
no `--no-verify`. (Bypass once, not recommended: `git push --no-verify`.)

**Force-push is NOT routine.** GitHub is shared — other devices/agents push too, so
expect healthy convergence and resolve it with a normal merge. `git push --force` is
dangerous (it clobbers their work); use it ONLY for exceptional recovery when a bad
history reached the remote and the local tree is the agreed source of truth.
