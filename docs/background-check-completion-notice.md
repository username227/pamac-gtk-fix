# Background update-check completion notice

Research/development checklist for `fix/background-check-responsive-browsing`.
This feature is **not merged into the GTK integration branch** until its new
completion-notice behavior has been tested with a built application.

## Intended behavior

- During a background update scan the existing header progress indicator
  continues to be visible and Apply remains insensitive.
- After a **verified, complete** background check, the spinner changes to a
  clickable, pluralized `1 update available` / `N updates available` notice
  when actionable repository/AUR/Flatpak updates exist.
- Ignore-list entries are not counted as actionable updates. Zero actionable
  updates hides the indicator. Incomplete or deferred checks never display a
  newly verified update count.
- The notice stays in the header while browsing until the user clicks it,
  navigates into Updates by another route, or successfully starts a transaction.
- Clicking the notice navigates to the existing Updates view; it does not
  install anything or start a second completed check.
- Clicking Apply from Browse/Search with other packages staged clears the
  notice, but invokes the normal `run_transaction()` path. It **does not**
  call `run_sysupgrade()` or append the discovered update list to staging.
- A later visit to Updates after a non-upgrade transaction follows the
  manager's existing refresh/verification behavior; the discovered update is
  not silently treated as installed by the notice dismissal.
- Refreshing updates again replaces the old notice with progress and, if
  successful, a fresh count.

## Manual regression steps

1. Queue one unrelated package install from Browse; begin a background update
   check while a real update is available. Confirm browsing still responds,
   the header shows progress and Apply stays disabled during the check.
2. After the check completes, confirm Apply becomes enabled for the staged
   package, the header reads `1 update available` and **stays visible**.
   Navigate the catalog without changing tabs; it must stay visible.
3. Click the header notice. Confirm it disappears, the Updates view opens and
   shows the discovered update, and no installation begins. Separately,
   navigate from Browse to the **ordinary Updates tab** after the same
   completed check: it must display the cached verified list, not start
   a second check. Returning to Browse and Updates again must not recheck.
4. Repeat steps 1–2, but click Apply in Browse instead. Confirm only the
   explicitly staged package is requested by Pamac; an unrelated update is
   not silently queued. After the transaction, enter Updates and refresh if
   necessary: the unrelated update should still be available.
5. Repeat with two or more updates. Confirm the pluralized count.
6. Repeat with zero actionable updates and with ignored updates; confirm no
   misleading count is shown.
7. Cause an update-check failure or lock deferral; confirm the manager retains
   its incomplete/waiting behavior and does not show a success count.
8. Start another update check while a notice is visible, and switch tabs
   while the scan is running. Confirm the count never outlives the previous
   snapshot and the header correctly switches back to progress.
9. Verify a narrow window's header, keyboard focus/clickability and relevant
   translations; no background scan should enable Apply prematurely.
10. After an unrelated transaction invalidates the completed snapshot,
    open the ordinary Updates tab. It must show `Update status not verified`
    and request an explicit Refresh, rather than silently scanning or
    presenting a stale list. Use the Refresh button and confirm a new
    verified list replaces the old state.
11. In a fresh manager window with no prior update check, the first visit
    to Updates must still initiate the initial discovery.

## Scope

All code changes are contained in `src/manager_window.vala`; the existing
libpamac upgrade engine, transaction confirmation and AUR lock rules are
unchanged. This branch has not been compiled or interactively tested in
the remote research environment.
