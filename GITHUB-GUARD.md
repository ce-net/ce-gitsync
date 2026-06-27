# ce-gitsync GitHub guard

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

## Pushing real work to GitHub

Re-author/squash the WIP first so the pushed range has no `gitsync@ce-net` commits,
e.g.:

```
git reset --soft <last-real-commit>     # or origin/main
git -c user.name="Leif Rydenfalk" -c user.email="ledamecrydenfalk@gmail.com" \
    commit --author="Leif Rydenfalk <ledamecrydenfalk@gmail.com>" -m "..."
git push origin main
```

To bypass once (not recommended): `git push --no-verify`.
