# WhatsApp group management — bridge endpoints + roster sync

Added 2026-08-25. Two pieces:

1. `scripts/whatsapp-bridge/bridge.js` — three new group-management endpoints
   (code is on disk; the **running bridge predates this change** and must be
   restarted before the endpoints answer — deliberately not done here).
2. `~/.hermes/scripts/roster-whatsapp-group-sync.py` — one-way sync of active
   roster staff into the UGG staff WhatsApp group via those endpoints.

## New bridge endpoints

### `GET /groups`

Lists every group the connected account participates in.

```sh
curl -s http://127.0.0.1:3000/groups | python3 -m json.tool
# [
#   { "id": "1203630xxxx@g.us", "name": "UGG staff",
#     "participantsCount": 21, "isAdmin": true }
# ]
```

`isAdmin` is true when the bridge account is `admin` or `superadmin` there.
Use this to find the group JID before writing the conf file. 503 when the
socket is down (same pattern as `/send` et al.).

### `POST /group-participants`

Add or remove members. `participants` may be raw numbers or JIDs; anything
with digits is normalised to `<digits>@s.whatsapp.net` (Australian 10-digit
`0XXXXXXXXX` becomes `61XXXXXXXXX`). Returns Baileys' per-participant array
— each entry `{ status, jid }` where `status` is a **string**; anything other
than `"200"` means WhatsApp rejected that one participant.

```sh
curl -s -X POST http://127.0.0.1:3000/group-participants \
  -H 'Content-Type: application/json' \
  -d '{"chatId":"1203630xxxx@g.us","action":"add",
       "participants":["0406117214","+61421220359"]}'
# [ { "status": "200", "jid": "61406117214@s.whatsapp.net", ... },
#   { "status": "403", "jid": "61421220359@s.whatsapp.net", ... } ]
```

400 on: missing `chatId`/`action`/`participants[]`, `action` not
`add|remove`, `chatId` not ending `@g.us`, a participant with no digits.

### `GET /group-invite?chatId=<jid>`

```sh
curl -s 'http://127.0.0.1:3000/group-invite?chatId=1203630xxxx@g.us'
# { "code": "Abc123XYZ", "url": "https://chat.whatsapp.com/Abc123XYZ" }
```

Requires admin in the group. 500 if WhatsApp returns no code (not admin, or
invite revoked).

## Sync script usage

```
~/.hermes/scripts/roster-whatsapp-group-sync.py [--dry-run]
```

- **Enabling**: put the group JID on a single line in
  `~/.hermes/scripts/roster-whatsapp-group.conf`. Missing file = silent no-op
  (exit 0), so the cron entry can stay installed while the group is stood down.
- **Staff source**: the trial DB (org = latest `roster_versions` row, else the
  single org). Active users holding the `STAFF` role; number is
  `whatsapp_number` else `phone_number`, AU-normalised.
- **Adds**: active staff numbers not in the group (via `GET /chat/<jid>`),
  applied in batches of 5. A per-participant add failure (typically 403 =
  the member's privacy settings block direct adds) triggers a one-off invite
  DM: "Hi <first name>, here is the UGG staff group link: <url>".
- **Removes**: only numbers in the group that belong to *deactivated* users
  (`is_active=0`, matched across all users regardless of role). Numbers not
  present in the users table (owners, non-staff) are **never** removed, nor is
  any number that also belongs to an active user.
- **Output contract** (watchdog pattern): completely silent on stdout when
  there were zero changes and zero failures. Otherwise one compact line, e.g.
  `WA-group sync: +Avalon(7214) / -Mitch(3667) / invite-DM sent to Jimin(4031) / failures: …`
  Names + last-4 digits only — full numbers are never printed. Exit 1 when
  anything failed, 0 otherwise.
- **Safety**: `flock` on `/tmp/roster-whatsapp-group-sync.lock` (a second
  instance exits 0 silently), 10 s HTTP timeouts, DB opened read-only, all
  crashes collapse to a single-line `WA-group sync error: …` + exit 1.
- **Env overrides**: `ROSTER_WA_SYNC_DB`, `ROSTER_WA_SYNC_BRIDGE`,
  `ROSTER_WA_SYNC_CONF`.

Suggested cron (after enabling the conf):

```
*/15 * * * * /usr/bin/python3 /home/mitchelly/.hermes/scripts/roster-whatsapp-group-sync.py
```

## Risks

- **Bridge restart needed.** The endpoints only exist after the bridge process
  restarts. Until then the sync script fails with `HTTP 404` on
  `/group-participants` (surfaced as a failures line / exit 1).
- **The bridge account must be group admin** to add/remove members and fetch
  the invite code. Check with `GET /groups` (`isAdmin`) first.
- **LID groups.** Groups in WhatsApp's LID addressing mode return members as
  `…@lid`; the script can only match `@s.whatsapp.net` members. In such a
  group every staff member looks absent → repeated re-add attempts, and
  re-adding an existing member yields a non-200 → they'd get an unnecessary
  invite DM **every run**. Verify with one manual run before scheduling. A
  future fix is exposing the bridge's LID→phone map.
- **Deactivated-number removal is irreversible from the script's side** — a
  wrongly-deactivated DB row removes a real person from the group. The
  active-number guard mitigates number reuse, not bad data.
- **WhatsApp ToS/bans.** Automated member adds via a linked-device session are
  grey-area under WhatsApp's terms; aggressive volume or frequency can get the
  number flagged. Batches are capped at 5 and the cadence should stay modest.
- **Invite DMs are user-visible.** A 403 add falls back to DMing the invite
  link to the staff member's number; if the roster's number is wrong, that DM
  goes to a stranger. (The DM text names the recipient's first name only.)
- **Conf file is unprotected.** Anyone able to write
  `roster-whatsapp-group.conf` can point the script at any `@g.us` group the
  bridge account administers. Keep `~/.hermes/scripts` tight.
