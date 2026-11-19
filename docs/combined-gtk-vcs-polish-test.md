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

## Follow-up: actual Updates list toggle callback (2026-11-18, 18:26)

The first attempt only guarded `create_update_row()`, which creates
rows for the optional OS Updates dialog. The main Updates page instead
renders `create_packagelist_row()` via `complete_packages_list()`.
Its `is_update` callback still interpreted programmatic checkbox
deactivations from `refresh_listbox_buttons()` as explicit ignore
choices. This explains why `iptvnator-git` appeared with the ignored
symbol after its previous successful upgrade, and remained marked
ignored when a newer VCS revision was discovered.

Corrected on this same combined GTK test branch:

- Guard the real main-list update checkbox callback during transactions,
  post-upgrade retained-only display and programmatic list-button refresh.
- Prevent reentrant `refresh_listbox_buttons()` updates from modifying
  selection/ignore state through nested widget signals.
- Capture an immutable *pre-transaction* snapshot of intentional ignore
  selections. After verified completion retain only the snapshot's
  outstanding candidates, **not** the mutable set after widget signals.
- Confirm an ALPM/AUR candidate is not already installed at its displayed
  version before preserving it as ignored. Reconstruct the transient
  ignored-name set exclusively from the remaining display-only rows so
  upgraded packages do not stay falsely ignored on future scans.
- Hide `Ignore all` during the post-upgrade ignored-only display.

Regression verification: upgrade `iptvnator-git` while ignoring only
`muffin`. The main list must not mark IPTVnator ignored in the
transaction summary or after completion; it should disappear from
post-upgrade ignored reminders. If a new VCS revision arrives on the
next scan, it must be actionable by default. Muffin remains ignored
and the tray is cleared immediately.

This updated correction is not yet compiled or locally tested.


## Regression: configured ignored repository update disappears after upgrade (2026-11-18, 20:06)

Observed: `muffin` 6.8.4-1 -> 6.8.4-2 remained correctly ignored while
`python-caldav` was selected. After completing the upgrade, the GTK
Updates page incorrectly displayed "Your system is up to date". Clicking
Refresh restored the ignored `muffin` row.

Root cause: the post-upgrade retention test used
`database.get_installed_pkg(name).version`, which in libpamac's
`AlpmPackageLinked` resolves to the current *sync repository* version
when one exists. Thus comparing `installed.version != pkg.version`
compared the candidate against itself and discarded the ignored row.

Correction in GTK only: compare `installed.installed_version`, the
locally installed ALPM version, to the candidate `pkg.version`.
Both retained repository and AUR update paths were corrected.

Retest: with `muffin` 6.8.4-1 held and repo candidate 6.8.4-2,
upgrade an unrelated package. Immediately after success, `muffin`
must still appear with the circular-arrow icon (display-only) without
pressing Refresh. The tray should clear; a subsequent manual Refresh
should continue showing `muffin` as ignored. Also verify a successfully
upgraded AUR package does not remain ignored as a stale row.

This fix is code reviewed but not compiler-/runtime-verified.


## Updates tab: clickable refresh glyph and animated check

The redundant right-side header progress/status button was removed. The
existing \`view-refresh-symbolic\` glyph inside the native **Updates** tab
is now an independent click target. The **Updates** label retains ordinary
tab navigation. While a check is running, the icon changes to a native
animated \`Gtk.Spinner\`, without explanatory header text. Both the
header view switcher and responsive bottom view-switcher bar are handled.

The implementation preserves the original icon and its property binding,
and wraps it in a \`Gtk.Stack\`. A capture-phase click gesture claims
mouse presses in the icon region before the surrounding tab can navigate.
This uses libadwaita's currently private \`AdwIndicatorBin\` child property;
if that internal structure changes, the injection skips it rather than
interfering with the other tabs.

Manual regression tests (not yet built or runtime verified):

- From Browse and Installed, click only the refresh glyph next to Updates.
  A fresh repository/AUR/VCS check must start **without selecting Updates**.
- Click the Updates text separately. It should navigate to that tab without
  starting a redundant check.
- While checking, the glyph must become a visible, moving spinner even
  when another tab is selected.
- Clicking the spinner during a check must not launch a second check;
  clicking during an active package transaction must not start a check.
- On success or failure, the spinner must stop and the circular-arrow
  icon return; clicking it again should work.
- Below the 550sp breakpoint, verify the corresponding icon in the bottom
  switcher. Check that the header/footer labels retain normal proportions.
- Verify the extra header banner never reappears and the tray still
  receives completed update results.


## Superseding UX: selectable post-upgrade ignored rows (2026-11-18)

The earlier notes above described retained ignored rows as **display-only**
with disabled action toggles. That was the original safety implementation,
not the current behavior. After further user testing confirmed successful
upgrades disappear from the list, ignored updates persist, and tray sync
works, the user requested restoration of Pamac's normal selection UX.

Commit `855ffd5` allows selecting retained ignored rows **without first
clicking Refresh**. Apply only becomes active once at least one row is
selected. Instead of trusting its retained display or even a VCS scan
within the normal 120-second reuse window, Apply starts an ordinary
**libpamac transaction-time** update check with `force_refresh=true` and
`reuse_recent_vcs_sources=false`. Its normal confirmation dialog is
required because the new repository/AUR scan can discover additional updates.
The ordinary GTK/CLI upgrade paths and the normal guarded reuse optimization
remain unchanged. See `retain-ignored-updates-after-upgrade.md` for full tests.

This latest selection/revalidation adjustment is committed but **requires a
new GTK build and runtime regression**; prior successful builds and tests
covered the preceding display-only implementation.
