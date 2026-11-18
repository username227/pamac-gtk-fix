# GTK Updates: synchronize repository metadata before showing actionable upgrades

## Observed regression

The tray showed pending updates, while the GUI Updates view listed only
`paru-git`. When Apply was pressed, the transaction synchronized
`extra.db` and planned a `jdk17-openjdk` download that had not appeared
in the GUI's update list. After cancelling and explicitly refreshing the
GUI, the missing repository packages appeared.

The previous default Updates path called `refresh_updates(false)`, which
joined an optional coordinator snapshot or checked packages against the
already installed pacman sync databases. `pamac-checkupdates --direct`
refreshes those databases only when `Database.need_refresh()` permits.
An ordinary full upgrade instead synchronizes repositories before
resolving the transaction. Those two snapshots can therefore disagree.
This is not evidence that libpamac skips repository candidates; its
`Database.get_updates_real()` enumerates sync database candidates before
it checks AUR/VCS.

## Change

In `src/manager_window.vala`, ordinary first-time Updates navigation
and Retry both use the same `refresh_updates(true)` path as the explicit
Refresh button. This existing path:
1. disables Apply and starts a repository synchronization;
2. aborts without publishing a verified update list if synchronization
   fails;
3. recreates the scanner database handles after the refresh;
4. discovers repository, AUR/VCS and enabled Flatpak updates;
5. publishes complete results to the coordinator and renders the list.

Tray/checkupdates periodic behavior, its refresh period, and the
coordinator's read-only `--join-only` protocol are unchanged. There
may be an additional VCS pass when opening the GUI after a tray check;
fresh upgrade visibility takes priority over reusing an older sync
snapshot.

The fix uses already available GTK/libpamac APIs. No libpamac or CLI
code change is required.

## Local verification (not yet run)

1. Build/install the GTK package from
   `fix/gtk-sync-repos-before-update-list`. Keep
   `integration/pamac-fix` untouched until approved.
2. Have repository updates available while tray/checkupdates reports
   one or more AUR/VCS updates. Start a fresh Pamac Manager and enter
   Updates using either the tab or the tray's Updates action.
3. Confirm the GUI synchronizes pacman repositories BEFORE it shows a
   selectable list, then scans AUR/VCS and enabled Flatpaks.
4. Confirm that `pacman -Qu` after the GUI's sync agrees with the
   actionable repository updates listed, accounting for configured
   IgnorePkg entries and replacements.
5. Confirm Apply does not start before the full check completes.
6. Retry an incomplete check: it must synchronize repository databases
   again. An unsuccessful sync must not enable Apply on a new snapshot.
7. With AUR checking disabled, confirm repository updates still appear.
   With Flatpak enabled, confirm it still appears in the same list.
8. Confirm that the explicit Refresh button and switching away/back
   after a complete check work without starting overlapping scans.
9. Run a planned update and inspect the transaction summary/log for
   newly discovered dependencies or upgrades.

## Remaining integrity issue

The full-upgrade transaction still synchronizes databases again before
execution. If the remote repositories change between the GUI check and
Apply, new upgrades can enter a transaction without appearing in the
previous UI snapshot. The GUI's `no_confirm_upgrade` fast path can skip
a summary for upgrade-only operations. A separate defense-in-depth
change should compare the displayed snapshot with the final resolved
upgrade set (or require explicit confirmation when they differ). This
branch fixes the immediate stale-database discrepancy; it does not
claim to solve every between-check-and-Apply race.

GitHub source-only change: not compiled or interactively tested here.

## UI progress: no phantom transaction during discovery

When the GUI enters background update discovery, a repository/AUR
synchronization can emit progress through `TransactionGtk.progress_box`
even though no package installation has begun. That same progress box
previously remained visible under the VCS status page, showing a fully
completed `Refreshing AUR...` bar and implying an ongoing transaction.

`set_background_check_running(true)` now hides the transaction progress
widget for the *entire* discovery lifecycle, including ordinary GUI
repository synchronization and its subsequent AUR/VCS scan. The existing
checking-page title, detailed VCS progress and header indicator remain
visible. When discovery completes or reports failure,
`set_background_check_running(false)` resets the no-longer-active
metadata-refresh progress and reveals the ordinary transaction controls.
`set_pending_operations()` then restores the correct download-size or
pending-operation text. Real package transactions retain their existing
progress path; no libpamac logic changes.

Additional manual regression checks:

1. From a red tray icon, open Updates directly. While repository/AUR
   synchronization and VCS scanning proceed, the bottom transaction-style
   progress bar must **never** appear.
2. Stay in Updates without switching tabs. Once scan completes, the
   bottom must not contain a stale `Refreshing AUR...` progress bar;
   normal Apply controls and any download-size summary should work.
3. Switch to Browse mid-scan, then return. It must neither recreate a
   stale transaction bar nor initiate a duplicate scan.
4. Trigger a failed/incomplete update check. The bottom transaction
   progress must not be stuck at 100%.
5. Perform an actual package upgrade after verification. Its real
   transaction progress and confirmation UI must remain visible.
6. Test non-update Browse/Search pending operations alongside background
   update discovery; their progress/selection state must recover when
   the check completes.

This is a source-only correction, not yet compiled or exercised locally.

## Completed upgrades with intentionally ignored updates

A successful upgrade that leaves explicitly excluded packages (e.g.
`muffin` while testing a new Cinnamon release) is not a failed or
incomplete transaction. Previously `run_sysupgrade()` demanded
`temporary_ignorepkgs.length == 0` before identifying an otherwise
complete upgrade. Because configured `IgnorePkg` entries are also
copied into `temporary_ignorepkgs` when the GTK updates list is built,
even permanent ignores prevented the coordinator's successful
completion notification. The result was an ambiguous **Update status
not verified** page and a tray icon stuck on its previous red state.

The completion classification no longer rejects intentional ignore
selections. It still requires a compatible default-profile check,
verified discovery (or an explicit forced refresh), an actual successful
transaction, and non-download-only/non-dry-run/non-no-refresh options.
The coordinator therefore receives the successful complete-selection
event and can clear the tray notification without repeating VCS scans.

**Scope note:** The coordinator's next *independent* discovery uses
`/etc/pamac.conf` ignore preferences. Packages excluded only for one
GTK transaction, but not configured in the persistent ignore settings,
can legitimately reappear on a later check. Keeping a transient
exclusion hidden across future checks is a different feature and is
not implemented here.

Regression tests:

1. Keep a repository package persistently ignored in Pamac, with another
   repository or AUR update available. Check and apply the nonignored
   upgrades. The GUI should not show **Update status not verified**;
   the tray should turn white without rescanning the ignored package.
2. Repeat with mixed repository and VCS updates, keeping the ignored
   package excluded and checking that the VCS snapshot is reused.
3. Confirm the ignored package is never installed by the upgrade.
4. A failed transaction, incomplete discovery, dry run, download-only
   operation, or profile mismatch must continue to be unverified.
5. A one-time GUI exclusion may be visible to the tray's *next*
   independent check; a persistent `IgnorePkg` exclusion should not.

Source change not yet compiled or locally regression-tested.
