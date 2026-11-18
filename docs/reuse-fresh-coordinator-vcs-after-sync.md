# Reuse freshly checked tray VCS sources after GUI repository synchronization

Opening GTK Updates intentionally synchronizes pacman databases before
showing an actionable list. Previously it always ran a new full VCS scan,
even when the tray had completed the identical scan moments earlier.

For a default-profile GUI, after a successful repository sync:

1. Ask the existing coordinator's read-only `--join-only` interface for a
   running/recent complete result (the coordinator currently caches for 30s).
2. Read typed repo/AUR/Flatpak updates under the shared AUR source lock,
   without rerunning Git clone / makepkg.
3. Reuse the result **only** if the complete typed package/version output
   exactly matches the coordinator's previous verified result.
4. If no cached result exists, run the original native VCS scan. If the
   freshly synchronized repo package set differs, also fall back to a full
   native scan rather than showing stale or incomplete updates.
5. Publish native fallback results to the coordinator as usual, and
   hand the accepted VCS snapshot to the manager transaction database.

This branch makes no changes to the libpamac coordinator or its cache TTL.
It does not guarantee tray-to-GTK reuse when repository versions have
changed, when the 30-second cache expires, or when GTK has different
profile/preferences. It preserves the primary safety requirement: never
sacrifice newly discovered pacman updates to avoid a VCS scan.

Manual test:

- Red tray scan finishes, immediately open GUI Updates; verify no second
  VCS source refresh when the result still matches after repo sync.
- Let 30 seconds elapse; ensure GUI falls back to normal VCS discovery.
- When a new repo update arrives between tray check and GTK sync, verify
  both the new repo package and VCS packages appear correctly after fallback.
- Test mismatched Flatpak/AUR preferences, incomplete coordinator result,
  lock contention, and failed repository synchronization.
- Upgrade a discovered VCS package and check that the transaction still
  obeys its own source-fingerprint/age checks.

Source-only: not yet compiled or locally tested. Separate from integration.
