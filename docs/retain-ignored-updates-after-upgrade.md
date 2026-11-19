# Retain and reselect ignored updates after a verified upgrade

After a successful normal-profile upgrade, Pamac-Fix retains **only** packages
that were intentionally ignored and are still outdated. The ignored rows
continue to display the traditional circular-arrow icon, and already-upgraded
packages must not reappear as ignored. This does not change the coordinator's
completed status or the immediate tray result.

## Selection and fresh transaction verification

- Initially, every retained row is still ignored and Apply is disabled.
- As with ordinary Pamac, the user can click a retained row to select it.
  Selecting at least one such row enables Apply without requiring a GUI Refresh.
- The retained version is a *displayed reminder*, not authorization to install
  from that old snapshot. Applying the selection takes the normal libpamac
  full-upgrade transaction path with **force_refresh=true**, including pacman
  repository synchronization and a new AUR/VCS source check.
- GTK explicitly sets `Transaction.reuse_recent_vcs_sources=false` for this
  path, even if a previous complete scan is within the normal 120-second reuse
  window. CLI transactions already follow the independent refresh policy.
- The transaction requires its regular confirmation dialog, even where the
  normal GTK Updates Apply path would skip confirmation. The fresh scan may
  expose more upgrades than were present in the retained list; the user must
  review the final transaction plan.
- Deselecting the row again disables Apply. Other still-ignored packages remain
  excluded. Successful transaction completion retains only any still-outdated
  ignored entries.
- A manual GUI Refresh instead runs normal update discovery and resets the
  retained-row state. Normal two-minute VCS reuse remains available for other
  eligible transactions; it is deliberately disabled only for retained-row
  Apply.

No new public libpamac API or CLI code is required. This change is in GTK
`src/manager_window.vala` and uses existing transaction semantics.

## Manual regression checklist

1. Ignore one repository package (e.g. `muffin`), upgrade a different
   actionable package, and verify successful tray synchronization.
2. The upgraded package must disappear; only the still-outdated ignored
   package remains with a circular-arrow icon. Apply is disabled while every
   retained package remains ignored.
3. Click the ignored row without refreshing. It must become selected and enable
   Apply. Deselect it; Apply must become disabled.
4. Select it again and click Apply. Confirm the libpamac transaction performs
   its own repository refresh and new VCS check; with
   `PAMAC_DEBUG_VCS_REUSE=1`, there must be no accepted cached-source reuse,
   even within 120 seconds of the earlier GTK check.
5. Confirm the final transaction summary is presented before changes are made.
   Do not approve any newly discovered unrelated package upgrades without
   reviewing the selection.
6. Complete a safe upgrade. Verify the newly upgraded package does not linger
   in Ignored, while other intentionally ignored outdated packages remain.
7. With no actual upgrade left (or a failed/cancelled fresh check), the old
   retained display must not silently authorize installing stale candidates.
8. Repeat for AUR/VCS ignored rows where feasible. Check Flatpak behavior
   separately, including a case where no ALPM updates are present.
9. Verify an ordinary fully checked GTK upgrade still uses guarded source
   reuse when it qualifies, and the standalone CLI still performs a fresh
   transaction-time check.

The new post-upgrade selection path requires fresh compilation and interactive
testing before advancing the GTK integration branch.
