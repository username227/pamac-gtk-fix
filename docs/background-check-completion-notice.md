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
   shows the discovered update, and no installation begins. Repeat the check
   with a normal Updates tab click instead; that must show the same verified
   list without starting a new discovery.
   Also test the app/tray Updates action against an already-open manager:
   both entry points now use the same show_updates_view() method; neither
   should navigate through win.back or force a redundant refresh.
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

## Scope

All code changes are contained in `src/manager_window.vala`; the existing
libpamac upgrade engine, transaction confirmation and AUR lock rules are
unchanged. This branch has not been compiled or interactively tested in
the remote research environment.

## Investigating an unexpected second scan

In a **newly launched** Pamac Manager process with debugging enabled,
the automatic discovery branch logs its state when someone opens Updates
without a verified snapshot:

```sh
PAMAC_DEBUG_UPDATE_NAVIGATION=1 pamac-manager
```

Close any already-running manager first; otherwise the application can
reuse the existing process without inheriting that environment variable.
If the ordinary Updates tab runs a fresh check while the completed
notice was visible, capture the log line beginning
`Updates navigation started discovery`. The valid completed path
(`updates_checked == true`) should show the current list instead,
without calling `refresh_updates(false)`.

A newly opened independent application instance cannot reuse another
window's in-memory `updates_checked` flag, so do not equate a new
process's first discovery with a redundant scan in an already-open
window. No silent refresh suppression or stale-cache fallback has
been added: if state is truly unverified, the original first-visit
behavior remains in place pending further evidence.
