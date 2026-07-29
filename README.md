# ce-gitsync — RETIRED 2026-07-29

**It does not run. Do not re-enable it.** The executable refuses at entry and exits 2.

Leif, 2026-07-29:

> make sure ce gitsync never runs agan! it should be disabled for good because its fucking up
> so much... just let agents handled this stuff by themselves. doumetn this that its fucking
> styff up.

## What it did wrong

**It wrote `live: ...` auto-commits into real history.** Not onto a side branch — interleaved
with deliberate commits, on the branch people work on. Nobody can take them out afterwards
without rewriting the branch, so the noise is permanent.

**Its own pre-push guard then refuses to push the branches carrying them.** On 2026-07-29 that
left `PLAN` with 120 unpushed commits and `legal` with 4. Work stranded on one laptop by the
system built to distribute it — the exact failure the commit-and-push law names.

**It committed whatever was on disk, mid-edit, with a message that says nothing about why.**
A commit body is meant to carry the reasoning. `live: mac` carries none, and a commit captured
in the middle of an edit is not a state anyone chose.

**It raced the agents.** Agents already commit their own work, and each other's, deliberately.
A second writer on the same tree adds conflicts and takes the authorship away.

## What replaces it

Agents handle their own git. Commit deliberately, put the WHY in the body, push. That is
already the standing law: everything in the tree, every time, including another agent's
uncommitted work, saying whose it is.

## The pre-push guard STAYS

`.git/hooks/pre-push` is installed in 263 repos and blocks any push to GitHub containing a
commit authored by `gitsync@ce-net`. **Keep it.** Retiring gitsync stops NEW auto-commits; the
guard is what keeps the ones already in history out of GitHub.

The guard is not the bug — it is the thing holding the line. `PLAN` and `legal` are blocked
because their branches carry gitsync commits, and the fix is to squash or re-author those
commits, not to switch the guard off. Tracked as backlog `5a1612ff`.

## The code

The original implementation is kept below the refusal in `ce-gitsync`, unreachable. The
bundle-over-mesh transport it used — git bundles through the node's content-addressed blob
store, with mesh announce/ack — is worth reading before anyone builds mesh git sync again.
The transport was never the problem. Auto-committing on someone else's behalf was.
