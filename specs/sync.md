# Sync

**Status:** Draft · **Prefix:** SYNC

## Purpose

Optional: keep one person's budget files in step across their devices through a self-hostable
server, without the server being able to read the data when encryption is on.

## Requirements

| ID | Requirement | Verified by |
|---|---|---|
| SYNC-1 | Sync is opt-in per budget file; a file that is never synced needs no server or account ([FILE-4](files.md)). Offline changes queue locally and are sent on the next sync. | edit offline on 2 devices, reconnect → both changes on both |
| SYNC-2 | The server uses `@specfork/auth` in `single_owner` mode (`@specfork/auth:AUTH-18`) with username-only sign-in allowed (`@specfork/auth:AUTH-19`); more people only as admin-created users (`@specfork/auth:AUTH-22`). Each server user sees and syncs only files they uploaded; there is no sharing, team or role between users. | user B lists files → none of user A's |
| SYNC-3 | Every local write is recorded as change messages `{table, row_id, field, value, timestamp}`, one per changed field; `value` is null, a number or a string. Derived values (balances, budget figures, report results) are never synced; every device recomputes them. | change payee and notes → 2 messages |
| SYNC-4 | Deletes sync as a change to the row's `deleted` field; rows are never removed outside SYNC-9. | delete on device A → hidden on device B |
| SYNC-5 | `timestamp` is a hybrid logical clock: UTC time to the millisecond, a counter 0–65535 and a 16-hex-char device id, ordered in that sequence. A remote time more than 5 minutes ahead of the local clock → `clock_drift`; counter past 65535 → `clock_overflow`; neither message is applied. | remote stamp +6 min → `clock_drift` |
| SYNC-6 | Conflicts: per (`table`, `row_id`, `field`) the message with the greatest `timestamp` wins, on every device, regardless of arrival order. A message with an already-seen `timestamp` is ignored; an older one is stored but not applied. | A sets notes "x" at t1, B "y" at t2 > t1 → both show "y" |
| SYNC-7 | Exchange: the client sends its unsent messages and its last sync time; the server stores them (idempotent by `timestamp`) and returns all messages of that file newer than that time from other devices, plus a digest of all stored timestamps bucketed by minute. If the client's digest differs, it resends and re-requests from the first differing minute. Sync errors: `file_needs_upload` (server has no snapshot), `file_has_reset`, `file_has_new_key` (key changed elsewhere; ask for the new passphrase), `key_mismatch`, `unauthorized` (sign in again). None of them changes local data. | lost response → next sync converges, no duplicates; other device changed key → `file_has_new_key` |
| SYNC-8 | Upload stores a full snapshot of the file with its `group_id` (created on first upload). A new device downloads the latest snapshot, then syncs messages after it. The client re-uploads a snapshot at most every 7 days. Server file operations for the signed-in user: list (name, `group_id`, encrypted flag, last upload), download, rename, delete (removes snapshot and messages). | sign in on new laptop → full file appears; delete on server → local copy stays |
| SYNC-9 | Reset sync: purges deleted rows and the local message log, gives the file a new `group_id`, uploads a new snapshot and empties the server's log for the old group. Other devices then get `file_has_reset` and must download again. | reset on A → B sync → `file_has_reset` |
| SYNC-10 | End-to-end encryption (optional, per file): key from a passphrase with Argon2id (64 MiB, 3 passes) and a random 32-byte salt; snapshots and every message `value` set are encrypted with AES-256-GCM and a random 12-byte nonce. The server keeps `key_id`, salt and one encrypted check value only. Wrong passphrase → `key_mismatch`. Turning it on or changing it performs SYNC-9. | wrong passphrase on new device → `key_mismatch`, nothing decrypted |
| SYNC-11 | Delete all my data (`@specfork/privacy:PRV-6`) and account deletion (`@specfork/privacy:PRV-7`) on the server remove every snapshot and message the user uploaded; copies on devices stay ([FILE-3](files.md)). | delete all → server file list empty, local file still opens |

## Out of scope

- Sharing a budget between people, roles, admin user management.
- Peer-to-peer sync without a server.

## Decisions

- Sync kept in scope because the product is local-first with optional sync; so `@specfork/auth` is required, used only by the server, in `single_owner` mode.
- Argon2id for the key; the source used PBKDF2-SHA512 with 10,000 iterations, below current guidance.
- The source's multi-user server features (roles, per-file access) are dropped: one person per file.
