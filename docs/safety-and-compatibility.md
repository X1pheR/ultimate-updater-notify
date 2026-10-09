# Safety and compatibility

The notifier exists to provide useful automatic update visibility without inheriting mutating behavior from Ultimate Updater's automatic-check paths.

## Automatic-check boundary

The companion deliberately separates **checking** from **installing**.

Automatic checks:

- invoke `/etc/ultimate-updater/check-updates.sh` only with `UU_JOB_SOURCE=initial-inventory`, `UU_DEFER_NOTIFICATION=true`, and bounded runtime for Proxmox hosts/guests;
- after that succeeds, reuse Ultimate Updater's `target-inventory.sh`, `external-selection.sh`, and bounded `external-apt.sh check <target>` path for selected External SSH targets;
- let Ultimate Updater own target selection, package-manager checks, security/normal classification, total counts, and reboot detection;
- consume Ultimate Updater's structured `status.json` and `STATUS_MODEL_RENDER_NOTIFICATION` output;
- never invoke the normal upstream `update -check` path;
- never install package updates;
- never start, stop, resume, suspend, or reboot Proxmox guests.

Actual package installation remains operator-triggered through Ultimate Updater. The `initial-inventory` mode is the accepted upstream read-only lifecycle boundary for Proxmox guests: stopped or paused selected guests are left unchanged and represented as `Not checked`. External checks use Ultimate Updater's native read-only SSH path and existing package metadata; they do not run `apt-get update` or install packages. The companion treats collection failures or Ultimate Updater's native `STATE=issues` result as failed scheduled checks rather than silently advancing the success heartbeat.

## Compatibility baseline

The supported safety-critical baseline is intentionally narrow at the upstream-interface level:

- Proxmox VE host running **Ultimate Updater 5.1** with its current `/etc/ultimate-updater` layout;
- `initial-inventory` behavior, the structured status-model interface, and the native External inventory/selection/read-only-check interfaces present in that release;
- target and package-manager support inherited from the accepted Ultimate Updater 5.1.3 check/status/External model rather than duplicated by the companion.

Stopped or paused selected guests are not started or resumed. Ultimate Updater represents them as `Not checked`, which the companion surfaces as a failed check. Unreachable, unsupported, errored, or otherwise not-checked selected targets likewise remain visible through Ultimate Updater's native `STATE=issues` rendering.

## Reviewed maintained source baseline (notifier v0.5.2)

This patch release explicitly accepts Ultimate Updater maintained downstream release `v5.1.3-x1pher.2` at commit `e2ce17043dd49e789e7b872966cbedf0d2c90555`. Its eight safety-critical delegated interfaces were compared against the previous accepted release `v5.1.3-x1pher.1`; changes are confined to `update.sh` (self-update pin), `check-updates.sh` (scheduled notification marker) and `status-model.sh` (optional delivery/Gatus). The native Ultimate Updater source verifier passed 29 Python scripts and 16 non-mutating shell fixtures, and the PVE release has seven-file byte parity with the published archive. The resulting PVE safety fingerprint is `c4d4c46a67429a536d1ef07517972d72e1dc0f3609ed59b10b98d3b3390e5e97`.

`accepted-updater-boundary.json` is the release-owned reviewed identity. The infrastructure deployment owner must verify its tag, source SHA, file list and fingerprint against the observed PVE bytes, archive identity and separately accepted deployment policy **before** replacing the previous accepted fingerprint. A new notifier version alone is not authorization to replace a mismatched state file. Never bypass compatibility health by deleting or editing the accepted fingerprint without this complete source-bound release gate.

## Upstream compatibility health

Before every automatic update check, the notifier validates the upstream integration boundary. A completed manual Ultimate Updater run validates the same boundary again after its normal completion notification.

The health check verifies that:

- Ultimate Updater reports exactly version `5.1`;
- `update.sh`, `check-updates.sh`, `status-model.sh`, `target-runtime.sh`, `target-inventory.sh`, `external-selection.sh`, `external-apt.sh`, and `tag-filter.sh` expose the accepted interfaces required by the delegated check paths;
- `STATUS_MODEL_RENDER_NOTIFICATION` remains callable;
- generated status.json declares exactly schema_version 1 with a list-valued targets field before any status rendering is accepted;
- Ultimate Updater's configured `LOG_FILE` still matches the manual observer path;
- no separate upstream automatic `update -check` or `check-updates.sh` cron entry exists in root's user crontab, `/etc/crontab`, or `/etc/cron.d`;
- the companion check timer and manual path watcher remain enabled and active.

The first successful compatibility check records a safety fingerprint across the eight safety-critical upstream interface files used by host/guest and External collection. Any later source drift fails closed and blocks automatic checks until a new notifier release explicitly accepts the changed upstream boundary. A failed compatibility probe sends a deduplicated ntfy warning; restoration of the accepted boundary sends one recovery notification.

## Cron ownership and restoration

During installation, matching upstream automatic-check entries are removed from:

- root's user crontab;
- `/etc/crontab`;
- files under `/etc/cron.d`.

The exact original matching lines are stored in root-only installer state. Reinstall removes reintroduced matching entries without overwriting the original backup.

Uninstall restores the saved lines to their original cron source when they are not already present. Unrelated cron lines are left alone.

This makes the companion systemd timer the only intended automatic update-check path while the companion is installed.

## Runtime bounds

The delegated Ultimate Updater `initial-inventory` run is bounded to 540 seconds by the companion. Each selected External target check is additionally bounded to 90 seconds, and the complete systemd automatic-check service remains capped at 10 minutes.

Outbound ntfy and Gatus HTTP calls use:

- 10-second connect timeout;
- 30-second total timeout.

These limits are safety bounds, not expected normal runtimes.

## Upstream relationship

The companion does not patch or redistribute Ultimate Updater source. It consumes Ultimate Updater 5.1.3's accepted read-only initial-inventory, External inventory/selection/check, and status interfaces plus the version/log/tag configuration needed for compatibility and operator-run observation.

Ultimate Updater remains responsible for the behavior and authorization of manual update installation.
