# Single-cycle GTK + libpamac test: ignored rows and redundant VCS scans

Use exactly the same branch name in both repositories:

- `username227/pamac-gtk-fix: test/combined-gtk-vcs-polish`
- `username227/libpamac-fix: test/combined-gtk-vcs-polish`
- `pamac-cli-fix`: unchanged, use existing integration branch.

The GTK branch combines:
1. Retaining **only ignored package rows** (circular-arrow icons, nonactionable)
   after a successful upgrade while keeping the coordinator and tray white.
2. Reusing a recent complete tray/coordinator VCS check after a GTK repo sync
   when freshly resolved typed update lists match; one native full scan
   remains the fallback when no matching cache exists.

The libpamac branch combines:
3. VCS-only source fingerprints so unrelated ordinary pacman-package changes
   do not trigger a repeat VCS source scan inside the 120-second window.
4. Clear `PAMAC_DEBUG_VCS_REUSE=1` logs for rejected handoffs/reuse;
   installed VCS versions and source-file changes still invalidate correctly.

## One integrated test sequence

1. Build with both repos set to `test/combined-gtk-vcs-polish`; keep CLI
   as currently integrated. Do not mix a new GTK branch with old libpamac.
2. With `muffin` persistently ignored and another update present, open
   the Updates view from the red tray immediately after a tray check.
   Repository updates must appear fully; count how many VCS checks run.
3. Apply the other updates within two minutes. Confirm successful install,
   no extra VCS source refresh when source state remains unchanged, white
   tray immediately, and no "Update status not verified" error.
4. Without refreshing, verify `muffin` remains displayed with its
   circular-arrow icon, but Apply is disabled and the retained row cannot
   be selected. Switch Browse / Updates and verify it stays visible.
5. Press Refresh. Ensure a complete new check runs; newly actionable
   package versions must never be concealed merely to save a source scan.
6. Change/install an *ordinary non-VCS* pacman package between a valid
   check and Apply; source reuse should remain eligible. Conversely, a
   changed installed VCS version or changed VCS PKGBUILD/.SRCINFO must
   invalidate the source cache and force a proper refresh.
7. After the 120-second reuse window, or if a required VCS package
   was never checked, confirm the transaction performs a safe full scan.
8. Verify failed/incomplete transactions, lock contention, mismatched
   profiles and Flatpak/AUR errors still fail safely without clearing the
   tray prematurely or silently installing unselected packages.

Both repository branches are **not yet compiled or locally tested**.
Their integration branches remain unchanged after the previous tested GTK
fast-forward. These are two physical Git branches in distinct repositories,
but one named test configuration / one build cycle.
 
## Regression found during the first combined test (2026-11-18)

The VCS sources were successfully reused (19 packages, no second source
refresh), and the transaction built and installed `iptvnator-git`
`r3449.4e7ebc7-1 -> r3451.57e6792-1`. However, immediately afterward
GTK incorrectly displayed it alongside the intentionally ignored `muffin`
under **Ignored**.

Cause: `transaction_sum_populated` calls `refresh_listbox_buttons()`,
which programmatically deactivates rows contained in the transaction
summary. The update-row `toggled` handler was interpreting that
programmatic deactivation as a new user ignore and adding the upgraded
package to `temporary_ignorepkgs`. The new ignored-row retention
correctly preserved that set, but therefore exposed the unintended
state mutation.

The update-row handler now ignores toggle changes while
`transaction_running` or while the post-upgrade ignored-only view is
being displayed. Explicit ignore choices made before Apply are not
affected. No coordinator, tray or libpamac changes were needed.

Additional tests:

- With `muffin` ignored, apply an available AUR/VCS upgrade such as
  `iptvnator-git`. After success, **only muffin** must remain under
  Ignored; the upgraded VCS package must not remain in Updates or Ignored.
- Confirm a normal pre-transaction user click on a package checkbox
  still deliberately adds/removes its ignored status.
- Recheck that the tray clears immediately and the source snapshot is
  reused. Switching tabs must not re-enable the ignored-only rows.

This correction is committed for testing but **not yet built or
runtime-verified**.
