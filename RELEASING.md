# Releasing HSBG

The update check in `HSBG Script Final.ahk` compares its own build stamp against
the newest release tag on GitHub. **Four things have to agree**, and if they
drift the failure is silent — users are either told to install a version they
already have, on every launch, or never told about a new one at all.

| Where | What it must say |
|---|---|
| `HSBG Script Final.ahk` — the `global HSBG_BUILD :=` line near the top | `global HSBG_BUILD := "v5.1.0"` |
| `HSBG Script Final.ahk` — the `HSBG_REPO_OWNER` / `HSBG_REPO_NAME` / `HSBG_REPO_BRANCH` lines beside it | `Adriatik-B` / `HSBG` / `FINAL` |
| The GitHub release tag | `v5.1.0` |
| `version.json`, `"version"` | `"v5.1.0"` |

The `Version` line in the file's header comment is repeated prose — bump it with
`HSBG_BUILD` so a person reading the header and a script reading the stamp agree.

Versions are compared as numbers, so `v5.1.0`, `5.1.0` and `5.1` are all equal
and all fine. A tag like `release-5` or `final2` is not a version and is
ignored — the check logs it and says nothing to the user.

> **The repository constants are on that list for a reason.** v5.1.0 was built
> pointing at `HSBG-Script-Final`, which exists, answers, and is not where
> releases are published. The check worked perfectly and compared against the
> wrong history:
>
> ```
> UPDATE this build (v5.1.0) is current; newest published is v5.0.0
> ```
>
> Nothing about running the script could reveal that. Hence the self-test below.

---

## Before you publish: run the self-test

`HSBG Update Self-Test.ahk` sits beside the script. **Run it before every
release and again straight after publishing.** It reads the repository, branch
and build number out of the real script and its settings file — so it tests
what will actually ship, not a copy of an idea of it — then asks GitHub the same
questions the real check asks and tells you what a user would experience.

It answers, in order:

- Does the Releases API answer at all? (404 here means the repository is wrong,
  private, or has a tag but no *published Release* — a tag alone is not enough.)
- Does the release carry an `.ahk` asset?
- Does `version.json` answer on your branch?
- **Do those two sources agree?** They are independent answers to the same
  question and the script takes whichever responds first, so a disagreement
  means users see one or the other at random.
- Would somebody on `v0.0.1` be offered this release? Would somebody on
  `v999.0.0` correctly be left alone?
- Optionally: does the asset actually download?

Every failing row says what it would mean for a user. Results are also written
to `HSBG Update Self-Test.log`.

Before publishing, expect one benign result: *"This build is NEWER than anything
published"* — that is what an unreleased version looks like. After publishing,
re-run it and that row should read *"is exactly current"*.

---

## Publishing a release

**1. Bump the numbers.** `HSBG_BUILD` in the script (and the header's `Version`
line with it), and `"version"` in `version.json`. Put one plain sentence in
`"notes"` — it goes into the user's log, so say what changed, not "misc fixes".

**2. Check the tracked files** are the current ones:

```
HSBG Script Final.ahk       ← the script
HSBG Update Self-Test.ahk   ← the update-path self-test
HSBG Config.ini             ← 16 keys
README.md
RELEASING.md
version.json
```

**3. `HSBG.log` stays untracked — `.gitignore` is preventative.** The script
writes a fresh log beside itself on every machine that runs it, so a committed
copy would only ever be one stale session belonging to whoever generated it.
Every line in it carries an absolute path, so that copy would also carry a
Windows username. The `.gitignore` exists so a `git add -A` run from a folder
you have actually been running the script in — which is exactly what step 4
is — cannot break that.

**4. Commit and tag:**

```bash
git add -A
git commit -m "v5.1.0"
git tag v5.1.0
git push origin FINAL --tags
```

**5. Publish the release.** On GitHub: *Releases* → *Draft a new release* →
choose the `v5.1.0` tag → **attach `HSBG Script Final.ahk` as a binary asset**
→ publish.

The attached file is what the tray item downloads. A release without it still
announces itself, but the tray item can only open the releases page — which is
the state `v5.0.0` was left in.

**6. Re-run the self-test.** Every row should pass. Someone running the old
version sees the notice within about ten seconds of their next start-up, or
immediately via the tray menu's **Check for updates now**.

---

### Worth knowing

- **`raw.githubusercontent.com` caches for about five minutes.** If you push
  `version.json` and test immediately, you may still get the old one. The
  Releases API is not cached this way, so a properly published release shows up
  at once. The self-test will show you both answers separately, which makes this
  obvious rather than confusing.
- **Users can turn it off.** `UpdateCheck=0` in `HSBG Config.ini` means no
  network request is ever made. Anyone who sets it will not hear about releases.
- **Forks don't need to edit the script.** `UpdateOwner`, `UpdateRepo` and
  `UpdateBranch` in `HSBG Config.ini` override the three constants. They are
  also how the self-test can be pointed at a scratch repository to exercise the
  "an update is available" path without publishing anything real.
- **Don't delete an old release to force an upgrade.** The check only ever
  compares against the newest tag; removing older ones changes nothing and
  breaks the download links of anyone mid-install.
- **A downgrade is never announced.** If you publish `v5.1.0` and then pull it,
  the newest tag becomes `v5.0.0` again and nobody is told to move backwards.
- **Rate limits are per-IP and clear within the hour.** 60 unauthenticated API
  calls. If the self-test reports 403, that is what happened; the real script
  falls back to `version.json` and is unaffected.
