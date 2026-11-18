# Retain ignored updates after a verified upgrade

After successful normal-profile upgrade, GTK previously discarded all package
update objects. This correctly stopped the old actionable upgrade snapshot
from being reused, but also removed ignored packages (for example `muffin`)
until the next explicit check.

The GTK-only change preserves just the packages marked in
`temporary_ignorepkgs` when a transaction completes successfully and
the coordinator acknowledges a verified full upgrade. It retains the
traditional circular-arrow icon without changing the coordinator result,
tray status, package selection sent to libpamac, or the ignore policy.

Retained rows are informational. Their toggle is disabled, Apply remains
insensitive, and an additional guard prevents a repeat sysupgrade from the
retained list. A fresh update check resets the informational flag and
rebuilds the complete source-based update list.

Manual test:

1. Ignore one repo package, with a second actionable package available.
2. Apply. Verify the selected package upgrades and the tray turns white.
3. Verify only the ignored package remains visible with the circular-arrow
   symbol. Apply must be disabled; clicking the ignored row must not
   schedule an upgrade.
4. Switch to Browse and return. The ignored row should still be shown.
5. Explicit Refresh should perform normal update discovery and restore
   standard selection behavior without installing ignored packages.
6. Verify failed/incomplete transactions retain their existing behavior.
7. Check the AUR and Flatpak ignored-row variants if available.

Not yet built or tested locally. No changes to libpamac or the tray.
