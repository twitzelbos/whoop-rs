# Using poop next to the WHOOP app

Findings from testing poop against a real strap while the official WHOOP app
was in use, so they don't have to be rediscovered. The question was: **can
poop and the WHOOP phone app both talk to the same strap, and how?**

- **Date:** 2026-11-24
- **Strap:** WHOOP 5.0 / MG (advertised name `WHOOP 5AM…`)
- **Host:** macOS (CoreBluetooth), poop 0.0.2 plus the branches named below
- **Phone:** iPhone with the WHOOP app, paired with the strap

## Summary

| Way | Works while the app is connected? | What you get | Status |
|---|---|---|---|
| `poop hr` (standard Heart Rate service) | **Yes** | Live bpm, about 1 per second. No R-R intervals | Tested |
| WHOOP custom service, phone still connected | **No**, needs a bonded link | Nothing | Tested |
| Taking turns (phone Bluetooth off) with `history --no-ack` | No, one at a time | Preview of history, battery, clock, settings | **Untested on hardware**; may also need bonding |
| Pairing the Mac with the strap | Unknown | Everything poop reads | **Not tried**: risks the phone's pairing |
| WHOOP's cloud API or Apple Health | Yes | Processed data (recovery, sleep, strain, HRV) | Not tried; no Bluetooth involved |

## Findings

### 1. While the app is connected, the strap hides its WHOOP service

With the phone connected, `poop scan` found nothing (10 s and 30 s windows).
An unfiltered BLE scan from the same Mac saw about 50 devices, including the
phone, so Bluetooth and its permission were fine. No device advertised either
WHOOP custom service:

- `61080001-8d6d-82b8-614a-1c8cb0f8dcc6` (4.0)
- `fd4b0001-cce1-4033-93ce-002d5875f58a` (5.0/MG)

`poop scan` filters on exactly those UUIDs, so it can't see a strap that the
phone holds.

### 2. "Broadcast Heart Rate" opens a second, unbonded connection

With **Broadcast Heart Rate** on in the WHOOP app (Device Settings), and the
phone's Bluetooth still on:

- The strap advertises the standard Heart Rate service (`180D`) under its name
  `WHOOP <serial>`. A scan filtered on `180D` finds it; `poop scan` does not.
- A second client (the Mac) can connect **without bonding** and subscribe to
  Heart Rate Measurement (`2A37`). Values arrived about once a second, and a
  20 s test ran and disconnected cleanly.
- In broadcast mode the strap sends **bpm only**: the R-R flag was never set,
  so no HRV can be computed from this stream.
- Still to confirm: that the WHOOP app's own connection stays up the whole time
  (check the app while `poop hr` runs).

This is what `poop hr` (branch `hr-broadcast`) does.

### 3. The WHOOP service needs a bonded link

Through the same `180D` sighting, with the phone still connected, the Mac
connected and asked for the full WHOOP service set (branch `via-hr`,
`poop info --via-hr`):

- Service discovery worked: the 5.0/MG custom service `fd4b0001-…` was found
  and the family resolved to WHOOP 5.0 / MG.
- Every notify characteristic refused to subscribe:
  - `fd4b0003-…`: *Encryption is insufficient* (CBError 15)
  - `fd4b0004-…`, `fd4b0005-…`, `fd4b0007-…`: *Authentication is insufficient* (CBError 5)
- With no notify channel, poop stops before sending any command, so nothing
  was written to the strap.
- macOS did **not** start pairing, and the strap did not appear among the
  Mac's Bluetooth devices afterwards.

So the WHOOP data channels are open only to a **bonded** (paired) client, and
standard Heart Rate is the exception. `history --no-ack --via-hr` was not run,
because it connects the same way and would fail at the same point.

**Open question:** does the strap also require bonding when the phone is
*disconnected*? poop's README doesn't say how it bonds on 5.0/MG. If it does,
taking turns without pairing won't work either. The first run with the
phone's Bluetooth off will tell.

### 4. Acknowledging history deletes it from the WHOOP app's view

From poop's code and README (`src/history.rs`, "Offloading history safely"):

- The strap keeps unsynced records in a ring buffer and sends them in chunks.
  Each chunk ends with an 8-byte token that the client echoes back to
  acknowledge it.
- **An acknowledged chunk is one the strap may discard.** A plain
  `poop history` acknowledges every chunk after storing it, so those records
  may never reach the WHOOP app, leaving gaps in its sleep, strain and
  recovery data.
- The offload only moves on after an acknowledgement. Without one, the strap
  keeps resending end markers: empty, or the same chunk again.

`poop history --no-ack` (branch `history-no-ack`) stores chunks without ever
acknowledging them. It stops at the first empty or repeated chunk, before
writing it, and ends the offload. The result is a **preview** (about the first
50 records), and the strap keeps everything for the WHOOP app. This was tested
against the simulator only: one chunk of 50 records, then a clean stop.

### 5. Things ruled out

- **Two full connections at once:** BLE allows it, but this strap's firmware
  doesn't offer its WHOOP service to a second, unbonded client (finding 3).
- **A relay (Mac between strap and phone):** the phone is paired with the
  strap's identity and keys, and macOS can't impersonate another device's
  address.
- **Automated turn-taking:** iOS can't reliably switch Bluetooth fully off from
  a script, and the strap goes to whichever side reconnects first.

## Not tried, and why

- **Pairing the Mac with the strap.** The strap may hold only one pairing, so
  pairing the Mac could remove the iPhone's and force a re-pair in the WHOOP
  app. Try it only with a way back agreed first, and after reading how poop
  handles bonding on 5.0/MG.
- **The phone-off run.** Not done yet; see the open question in finding 3.

## Branches in this fork

| Branch | Contents |
|---|---|
| `hr-broadcast` | `poop hr`: live heart rate over the standard service only (finding 2) |
| `history-no-ack` | `poop history --no-ack`: offload without acknowledging (finding 4) |
| `via-hr` | `--via-hr`: find the strap through its `180D` broadcast and try the WHOOP service (finding 3). Based on `history-no-ack` |

## Reproducing

```sh
# Broadcast Heart Rate on in the WHOOP app; phone Bluetooth on.
poop scan                   # finds nothing while the phone holds the strap
poop hr --duration 30       # live bpm (hr-broadcast)
poop info --via-hr          # connects, then "Encryption/Authentication is insufficient" (via-hr)

# Phone Bluetooth off (Settings → Bluetooth), wait ~30 s:
poop scan
poop history --no-ack --out peek.ndjson
```
