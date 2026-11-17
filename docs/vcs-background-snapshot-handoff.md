# GTK background VCS snapshot handoff — regression fix and test plan

## Reproduction / root cause

The user confirmed the regular Updates navigation now works, but a full
system upgrade immediately repeated the already completed VCS scan of 19
development packages. The transaction printed:

```
Cached development source check unavailable; refreshing sources.
```

The responsive GTK browsing implementation uses a separate
`update_database` for update discovery. The transaction retains the
manager's original `database`. The guarded VCS snapshot stamp, monotonic
timestamp and checked package-name set are **per Database instance** in
libpamac. Thus the transaction did not see the already verified update
scanner's record. This was a regression in the integration between
responsive browsing and guarded VCS reuse.

## Candidate fix (not yet built/tested)

Libpamac branch:
`username227/libpamac-fix:fix/gtk-background-vcs-snapshot-handoff`

GTK branch:
`username227/pamac-gtk-fix:fix/background-check-responsive-browsing`

- Add `Database.adopt_verified_vcs_snapshot (Database checked_database)`.
- Call it **only** from `apply_update_lists`, after complete and validated
  update results. The coordinator join path already compares typed
  names/versions with its verified output.
- Transfer only the original fingerprint, timestamp and exact package
  coverage, never cached package objects or ALPM handles.
- Recheck the fingerprint against the receiving transaction Database while
  holding the AUR source metadata lock; require matching enable flags,
  complete scan and nonexpired two-minute timestamp.
- Reuse remains checked again under the exclusive AUR build lock by
  `can_reuse_vcs_snapshot`. A missing, stale, changed or incomplete record
  causes the original full refresh without loosening source safety.
- The source and transaction databases remain independent for responsive
  browsing and reliable transaction semantics.

## Build order

Rebuild/reinstall **libpamac first** from its handoff feature branch,
then GTK from its responsive-browsing feature branch. CLI does not change.
Do not mix the new GTK with an older libpamac without the new VAPI symbol.

## Manual regression checklist

1. With `PAMAC_DEBUG_VCS_REUSE=1` in the freshly launched manager process,
   refresh updates via the manager and wait for all installed development
   packages to finish. Check `GTK verified VCS scan handoff: accepted`.
2. Within two minutes, initiate a normal full upgrade from the verified
   Updates view, when there is an actual upgrade to install. Confirm the
   progress reports **reused** verified development sources rather than
   refreshing all 19 packages a second time.
3. Repeat with an update check joined via `pamac-checkupdates --join-only`.
   A complete typed result should hand off if its source fingerprint agrees.
   A coordinator result older than its normal 30-second join window is not
   eligible to join.
4. After more than two minutes have elapsed since the original scan, expect
   a full rescan; handoff must not extend the original two-minute lifetime.
5. With different ignore selections, changed installed-package state, a
   changed PKGBUILD/.SRCINFO, or concurrent AUR lock contention, verify
   fail-closed fallback to a full scan (or normal lock-wait behavior).
6. Confirm background browsing responsiveness, the persistent available-
   updates notice, the normal Updates tab and the external Updates action
   all still work without redundant discovery.
7. Check no package update is auto-selected by Browse/Search Apply.
8. Validate Vala compilation and a clean package upgrade before considering
   integration; no binaries were compiled in the GitHub-only environment.

**Important:** This patch addresses redundant *VCS source refreshes*. It
does not suppress necessary repository-database synchronization performed
by the transaction.
