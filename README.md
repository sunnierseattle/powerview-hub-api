# PowerView Gen 1 / Gen 2 Hub — Local HTTP API

> **Unofficial.** Independent documentation of the PowerView hub's local API, built by testing real
> hubs. Not affiliated with or endorsed by Hunter Douglas. PowerView® is a trademark of Hunter Douglas.
> Writing to a hub (moves, scene edits, settings) changes real hardware — test with care.

Reference for the **local** HTTP API of the Hunter Douglas PowerView Hub (`http://<hub>/api/...`),
covering Gen 1 and Gen 2 hubs. Gen 3 ("Gateway") speaks a different API (`/home/...`, 0–1 float
positions) and is out of scope.

## Legend

Every call and behaviour in this document carries one of two marks:

| Mark | Meaning |
|---|---|
| 📘 **Documented** | Described in the vendor API document (see below). |
| 🧪 **Undocumented** | Not in that document. Observed on a real hub; no vendor guarantee it survives a firmware update. |
| ⚠️ **Contradicts** | The document says one thing; the hub does another. Listed together below. |

A 🧪 call is additionally either **exercised** (called and seen working) or **advertised only** (the
hub's `OPTIONS` reply lists the method, but it was never called — usually because it could unpair,
deregister or overwrite the hub).

**About the vendor document.** *Hunter Douglas PowerView® Hub API for Home Automation Integration*,
REST API version 2.0.0, generated 2019-05-17. It is written as Hunter Douglas's own developer
specification and matches the hub's behaviour wherever tested, but it carries no author or copyright
notice and **Hunter Douglas has never published it** — its support site links no API document. It
appears to have been given to integration partners. Public copies: an
[openHAB issue attachment](https://github.com/openhab/openhab-addons/files/7583705/PowerView-Hub-REST-API-v2.pdf)
and [Debian's python-aiopvapi source package](https://sources.debian.org/src/python-aiopvapi/3.1.1-2/PowerView_Hub_REST_API.pdf).
It documents **no POST or DELETE calls at all** and only three PUTs.

Reference hubs:
- **Gen 1** — `mainProcessor.name "PowerView Hub"`, firmware `1.1.827`, no `radio` block.
- **Gen 2** — `mainProcessor.name "PV Hub2.0"`, firmware `2.0.1056`, `radio 2.0.2610`.

Last checked against a Gen 2 hub on build 1056 on **2026-11-26**. Behaviour can drift across firmware.

## ⚠️ Where the hub contradicts the document

| Topic | The document says | The hub does |
|---|---|---|
| `stop` on Gen 1 (§3.4) | `stop` is a motion for every hub (only home automation is marked Gen 2 only) | Gen 1 answers 200 and **lets the move finish**. Gen 2 halts the shade. |
| Saved position after a stopped move (§3.3) | "Since the Hub is always involved in these motion events, it tracks the final shade position and saves it" (overview; it does not discuss a move interrupted by `stop`) | With position verification off, a move stopped mid-travel kept reporting its **commanded target** for at least 70 s, until a refresh. Treat this as a qualification of the overview rather than a flat contradiction. |
| Scene activation reply on Gen 1 (§4) | `GET /api/scenes?sceneId={id}` → `{"shadeIds":[…]}` | Gen 1 (build 827) nests it: `{"scene":{"shadeIds":[…]}}`. Gen 2 (build 1056) matches the document. |
| Single-shade read on Gen 1 (§2) | `GET /api/shades/{id}` returns that shade | Gen 1 also returns every other shade in a `shadeData` tail. |

---

## 1. Call index

### 📘 Documented calls

| Call | Purpose |
|---|---|
| `GET /api/fwversion` | Hub and radio firmware versions |
| `GET /api/homeautomation` | Position-verification settings and state (Gen 2 only) |
| `PUT /api/homeautomation` | Set `enabled`, `timeoutSec` (15–300), `configNum` (Gen 2 only) |
| `GET /api/homeautomation/report` | Positions from the last verified move (Gen 2 only) |
| `GET /api/rooms`, `GET /api/rooms/{id}` | Rooms |
| `GET /api/scenecollections`, `GET /api/scenecollections/{id}` | Multi-room scenes |
| `GET /api/scenecollections?sceneCollectionId={id}` | Activate a multi-room scene |
| `GET /api/scenes`, `GET /api/scenes/{id}` | Scenes |
| `GET /api/scenes?sceneId={id}` | Activate a scene |
| `GET /api/shades` (`?groupId=`, `?roomId=`) | List shades, optionally filtered |
| `PUT /api/shades?groupId={id}` | Move, or run a motion on, every shade in a group |
| `GET /api/shades/{id}` (`?refresh=`, `?updateBatteryLevel=`, `?requestFirmwareRev=`) | One shade; optional RF requests |
| `PUT /api/shades/{id}` | Move a shade, or run a motion (`down`, `up`, `stop`, `jog`, `heart`, `leftTilt`, `rightTilt`) |
| `GET /api/userdata` (`?includeCounts=`) | Hub identity, network, time, LED colour, flags |

### 🧪 Undocumented calls

| Call | Purpose | Seen |
|---|---|---|
| `PUT /api/homeautomation` with `postBackUrl` | **Real-time push feed** of shade motion (§5.2) | exercised |
| `PUT /api/userdata` | Change hub settings, including the **status LED** colour (§6.2) | exercised |
| `GET /api/shades/{id}?survey=true` | Radio diagnostic: signal strength of neighbouring devices (§3.1) | exercised |
| `GET /api/scenemembers?sceneId={id}` | A scene's shades and their target positions (§4) | exercised |
| `GET /api/backup` | Full hub configuration dump, no authentication (§6.3) | exercised |
| `GET /api/groups` | Radio groups | exercised |
| `GET /api/repeaters` | Signal repeaters | exercised (empty) |
| `GET /api/smartpowersupplies` | Smart power supplies | exercised (empty) |
| `GET /api/scheduledevents` | Hub-side schedules (only the `enableScheduledEvents` flag is documented) | exercised (empty) |
| `GET /api/scenecollectionmembers` | Members of multi-room scenes | exercised (empty) |
| `GET /api/homekit/ispaired` | Whether the hub is paired with HomeKit | exercised |
| `GET /api/integrations` | Third-party integrations (`{nest:null}`) | exercised |
| `GET /api/register` | Cloud account registration state | exercised |
| `PUT /api/times` | Set hub time (there is no GET) | exercised: every body tried got `400 {}` |
| `POST /api/rooms`, `PUT /api/rooms/{id}`, `DELETE /api/rooms/{id}` | Create, rename, delete a room | exercised |
| `POST /api/scenes`, `PUT /api/scenes/{id}`, `DELETE /api/scenes/{id}` | Create, rename, delete a scene | exercised |
| `POST /api/scenemembers`, `DELETE /api/scenemembers?sceneId=&shadeId=` | Add and remove a scene member | exercised |
| `POST /api/scheduledevents`, `PUT /api/scheduledevents/{id}`, `DELETE /api/scheduledevents/{id}` | Create, edit, delete a schedule | exercised |
| Writes to `homekit`, `register`, `backup`, `integrations`, shade POST/DELETE, scene collections, repeaters, smart power supplies | Pair/unpair, cloud registration, restore, and objects this hub does not have | advertised only (deliberately not exercised) |

Full method list per route, from the hub's `OPTIONS` replies (Gen 2):

| Route | collection | `/:id` |
|---|---|---|
| `/api/shades` | POST, GET, HEAD, PUT, DELETE | POST, GET, HEAD, PUT, DELETE |
| `/api/scenes` | POST, GET, HEAD | POST, GET, HEAD, PUT, DELETE |
| `/api/scenecollections` | GET, HEAD, POST | GET, HEAD, POST, PUT, DELETE |
| `/api/scenemembers` | POST, GET, HEAD, DELETE | — |
| `/api/scenecollectionmembers` | GET, HEAD, POST, DELETE | — |
| `/api/rooms` | GET, HEAD, POST | POST, GET, HEAD, PUT, DELETE |
| `/api/groups` | GET, HEAD | POST, GET, HEAD |
| `/api/scheduledevents` | GET, HEAD, POST, DELETE | GET, HEAD, POST, PUT, DELETE |
| `/api/repeaters` | POST, GET, HEAD | GET, HEAD, PUT, DELETE |
| `/api/smartpowersupplies` | GET, HEAD, DELETE, PUT | — |
| `/api/userdata` | GET, HEAD, PUT | — |
| `/api/fwversion` | GET, HEAD | — |
| `/api/times` | PUT | — |
| `/api/backup` | GET, HEAD, POST, PUT | — |
| `/api/register` | GET, HEAD, POST, DELETE | — |
| `/api/homekit/:param` | GET, HEAD, PUT | — |
| `/api/homeautomation` | GET, HEAD, PUT | — |
| `/api/homeautomation/report` | GET, HEAD | — |
| `/api/integrations` | GET, HEAD, PUT | — |

Not routed (`404 Not Found`): `/api`, `/api/<id>/activate`, `/api/shades/<id>/motion`,
`/api/shades/<id>/calibrate`, `/api/network`, `/api/discovery`, `/api/firmware`, `/api/reset`,
`/api/reboot`, `/api/restore`, `/api/unregister`, `/api/v2/*`, `/home/*`, and anything outside `/api/`
(no static files, no Swagger). Motion has no sub-route; it is a body on `PUT /api/shades/{id}`.

---

## 2. 🧪 Transport and server behaviour

None of this section is in the vendor document.

| | Gen 1 (build 827) | Gen 2 (build 1056) |
|---|---|---|
| Server | hand-rolled C (`Server: HunterDouglas PDBU Hub v1.01.827`) | Node.js / Express (`X-Powered-By: Express`) |
| Open ports | 80 | 80 only (no 443/22/3000/3002/MQTT) |
| JSON | hand-rolled (duplicate keys, odd whitespace) | standard serializer |
| Unknown `/api/*` path | TCP reset | `404` JSON |
| Keep-alive | **reuses the previous request's id** (`/api/scenes` after `/api/shades/{id}` → 404) | correct; reuse is safe |
| Single-shade read | ⚠️ also returns every other shade in a `shadeData` tail | only the requested shade |
| Content type | `application/json` | `application/json; charset=utf-8` |

**For clients:** on Gen 1, send `Connection: close` on every request and serialize calls (the hub also
drops TCP while its radio is busy). On Gen 2 neither is required. Both hubs handle one RF operation at
a time, so serializing writes is wise on both.

### HTTP parsing (Gen 2)

Lenient: HTTP/1.0, missing or duplicate `Host`, a bogus version (`HTTP/9.9`) and multi-KB headers all
succeed. `Expect: 100-continue` is honoured. Routing is case-insensitive (`/API/SHADES` ==
`/api/shades`) and ignores a trailing slash. Path traversal is not collapsed: `//`, `/./`, `/../` and
`%2f` all `404`. An unknown or mis-cased method token (e.g. lowercase `get`) gets a TCP reset.

### Error model (Gen 2)

The vendor document lists 400 / 423 / 500 per call; the finer distinctions below are observed.

| Outcome | Meaning |
|---|---|
| `400 {"message":"Failed to decode param '%'","error":{}}` | malformed percent-encoding in a path segment |
| `404 {}` (2-byte body) | a real route; the id parsed but no such resource (or the id is falsy, e.g. `0`) |
| `404 {"message":"Not Found","error":{}}` | no route matched |
| `423` | hub busy with maintenance (📘 documented) |
| TCP reset | unknown or mis-cased HTTP method token |

`{}` versus `{"message":"Not Found"}` tells a real route from an unrouted path. `error` is always `{}`;
no stack traces leak.

### Ids and query parameters (Gen 2)

Path ids are coerced like JavaScript `parseInt`: `1342abc` → 1342, `01342` → 1342, `0x53E` → 1342,
`1342.0` → 1342, `1e3` → 1. Non-numeric ids → `404 {}`.

Unknown query parameters are silently ignored on both generations (byte-identical response). Known
parameters are value-checked (`refresh=false` is a no-op).

---

## 3. Shades

### 3.1 Reading

- 📘 `GET /api/shades` — list. Documented shade fields: `id`, `type`, `batteryStatus`,
  `batteryStrength` (required); `name`, `secondaryName`, `roomId`, `groupId`, `order`, `firmware`,
  `positions` (optional).
  🧪 Gen 2 entries also carry `signalStrength`, `motor`, `capabilities`, `batteryKind` and
  `smartPowerSupply`; `motor` is missing on some shades. On Gen 1 the list has no positions until a
  refresh.
- 📘 `GET /api/shades/{id}` — one shade, `{"shade":{…}}`.
- 📘 `GET /api/shades/{id}?refresh=true` — asks the shade for its position over RF.
  🧪 This wakes the motor (audible), takes ~3–6 s, and returns `timedOut:true` if the shade does not
  answer; otherwise the shade's **measured** position.
- 📘 `GET /api/shades/{id}?updateBatteryLevel=true` — asks the shade for its battery level over RF.
  🧪 Also wakes the motor.
- 🧪 `GET /api/shades/{id}?survey=true` — RF round-trip (~6 s, no movement) returning the signal
  strength the shade hears from its neighbours:
  `{"shade_id":…,"survey":[{"neighbor_id":0,"rssi":-64}]}`. `neighbor_id 0` is the hub. RSSI is in
  dBm; nearer 0 is stronger. `{}` if the shade does not answer. Gen 1 ignores the parameter.

### 3.2 Position values

📘 `positionN` is an integer from 0 to 65535; `posKindN` 1 = primary (bottom) rail, 2 = secondary
(top) rail, 3 = vanes. For a bottom-up shade 0 = closed and 65535 = open. A top-down shade uses the same
coordinates reversed (65535 is its closed value). A top-down/bottom-up shade uses two coordinate spaces,
one per rail, each with 0 = closed and 65535 = fully open.

📘 `posKindN` 0 = none, 4 = error.

🧪 Fully open does not always read 65535. A shade sent `motion: up` measured **65535**; shades sent a
position of ~65400–65460 measured **65321–65474**. So 65535 is a real reading for a shade at its top
limit, consistent with the document. Scenes use it the same way: a scene whose members are all
65535 opened every shade, and afterwards each shade read 65535. (Earlier notes treated 65535 as a
"position unknown" marker; the evidence for that was a shade reading 65535, which a fully open shade
also does.)

Not verified: the reference hubs have no tilting shades, so the vane range is untested. The document
gives 0–65535 for every position value; client libraries treat vanes (`posKind 3`) as 0–32767.

### 3.3 🧪 Which position a read returns

⚠️ The vendor document says the hub saves the final position of every move it makes. In practice what
a plain read returns depends on position verification (§5.1):

| Source | Reports |
|---|---|
| plain read, verification **off** | the **last commanded** position, updated immediately; after a stopped move it stayed at the target until a refresh |
| plain read, verification **on** | the commanded position, then the **measured** position once the hub verifies (after `timeoutSec`) |
| `GET …?refresh=true` (once `timedOut:false`) | the shade's **measured** position |
| push feed (§5.2) | the hub's report of where the shade stopped, as 0–100 % — close to measured but not exact (see accuracy in §5.2) |

With verification **off** (checked 2026-11-26): commanded 30000 (46 %) and stopped after 3 s, the plain
read returned `30000` at 10 s, 30 s and 70 s. A refresh then returned `57875` (88 %), and the plain
read showed `57875` from then on. A plain read is wrong after a stopped move, or a move made from a
remote or the app, until a refresh.

With verification on (`timeoutSec:15`), the same test corrected itself: commanded 30000 (46 %) and
stopped after 3 s, the plain read showed `30000` at t = 6.4 s and `57795` (88 %) at t = 16.4 s (polled
about once a second; the change happened between those samples), matching a refresh (`57800`).
Moves made from a remote or the app are still not seen, because the hub only
verifies moves it commanded.

### 3.4 Moving

- 📘 Move: `PUT /api/shades/{id}` `{"shade":{"id":…,"positions":{"posKind1":1,"position1":30000}}}`.
  🧪 On Gen 2 the reply arrives after the RF command is sent (~1.8 s) and echoes the **target**.
- 📘 Motion: `PUT /api/shades/{id}` `{"shade":{"motion":"stop"}}` (also `up`, `down`, `jog`, `heart`,
  `leftTilt`, `rightTilt`). Only one of `positions` or `motion` per request.
  ⚠️ **Gen 2 halts the shade on `stop`; Gen 1 answers 200 and lets the move finish.** The reply echoes
  the move's target either way, not where the shade stopped.
- 🧪 Motions observed (Gen 2), one run each: `down` → measured 0 about 50 s later. `up` from fully
  closed → measured 82 % at the first read (~25 s), and 65535 (fully open) when read at 40 s in a
  second run. `heart` → measured 58 % about 10 s later. The document gives `heart` no meaning; the
  PowerView remote's "favourite" button is a hypothesis, not verified.
- 🧪 After `down`, `active` read false 1.7 s later and `/report` was empty, so motions may not be
  position-verified (the document lists only position sets and scenes); not confirmed for other motions.
- Not verified: client libraries send motion `calibrate` (relearn travel limits). It is not in the
  document's motion list and was not exercised.
- 📘 `?requestFirmwareRev=true` 🧪 is an RF round-trip (~3 s); the returned firmware was unchanged.
- 📘 Group: `PUT /api/shades?groupId={id}` `{"shade":{"positions":{…}}}` or `{"shade":{"motion":…}}`.
  🧪 Returns `{"shadeIds":[…]}`. A group position sent five shades to 50000; all measured 49941–49993.
  A group `stop` halts every shade in the group with one RF command.

---

## 4. Scenes

- 📘 `GET /api/scenes` — list: `{id, name (base64), roomId, order, networkNumber, colorId, iconId}`.
  🧪 Entries also carry `hkAssist`.
- 📘 Activate: `GET /api/scenes?sceneId={id}` → `{"shadeIds":[…]}`. 🧪 Gen 2 (build 1056) matches,
  replying in ~3.3 s. ⚠️ Gen 1 (build 827) replies `{"scene":{"shadeIds":[…]}}`, so a client should
  accept both. The hub sends one RF burst so the shades move together; the reply names the shades but
  not their targets.
- 🧪 `GET /api/scenemembers?sceneId={id}` → each member `{positions, id, sceneId, shadeId, type}`,
  carrying the target positions, so a client can update its state after a scene without reading each
  shade. A member at `position1:65535` means fully open: activating a scene whose members were all
  65535 opened every shade (§3.2).
- 📘 Multi-room scenes: `GET /api/scenecollections`, activate with
  `GET /api/scenecollections?sceneCollectionId={id}`.

---

## 5. Home automation

### 5.1 📘 Position verification (documented, Gen 2 only)

`GET /api/homeautomation` → `{"homeautomation":{"enabled":true,"timeoutSec":30,"configNum":10,"active":false}}`.

Per the vendor document: when `enabled`, after any position command the hub waits `timeoutSec`
(15–300 s, restarted by each new command), then asks the affected shades for their position, retrying
if needed. `configNum` increments (and persists) whenever a shade fails verification — a final position
well off target, or no answer. `active` is true while the timer runs or the hub is polling.
`PUT /api/homeautomation` sets any of `enabled`, `timeoutSec`, `configNum`. The document says to set
`timeoutSec` longer than the slowest full travel of any shade in the installation, with extra time for
a nearly flat battery.

`GET /api/homeautomation/report` → for each shade in the last move, `lastPos`, `targetPos`,
`reportedPos`, `state` (reached target) and `timedOut`. Valid once `active` is false.

🧪 Checked 2026-11-26; consistent with the document. With `timeoutSec:15` (a short test value, not a
recommendation), a position command then a stop after 3 s, polled about once a second: `active` was
true at the first sample (6.4 s) and at 16.4 s, and false at 19.7 s; `configNum` went from 87 to 88 by
19.7 s, alongside a failed verification; `/report` showed `lastPos 65385`, `targetPos 30000`,
`reportedPos 57795`, `state:false`, `timedOut:false`, and a refresh agreed (57800). Setting
`configNum` with PUT works (75 → 80).

### 5.2 🧪 Real-time push feed (undocumented)

The vendor document never mentions `postBackUrl` or any outbound request. On Gen 2, setting it turns
the hub into a push source; not observed on Gen 1.

**Enable:**
```
PUT /api/homeautomation
{"homeautomation":{"enabled":true,"postBackUrl":"<host:port>","timeoutSec":60}}
```
`postBackUrl` is a bare `host:port` (no scheme, no path). The factory value is `localhost:7041`.
There is one `postBackUrl`, so one consumer at a time.

**The hub then POSTs to `http://<postBackUrl>/`:**
- `User-Agent: Wget`, `Content-Type: application/x-www-form-urlencoded`, `Connection: close`.
- **Motion events:** body is **base64 of a JSON array** of `{service:"primary", shadeId, type, …}`:

  | `type` | extra fields |
  |---|---|
  | `targetLevelChanged` | `durationMs`, `initialPosition`, `targetPosition` |
  | `beginsMoving` | `currentPosition`, `remainingDurationMs` |
  | `levelChanged` | — |
  | `startsOpening` / `startsClosing` | — |
  | `stops` | `stoppedPosition` |
  | `hasOpened` / `hasClosed` | — |

  Positions are **0–100 %** (e.g. `targetPosition:46` == 30000/65535). Example:
  `[{"type":"stops","shadeId":12345,"stoppedPosition":55,"service":"primary"}]`.
- **Config events:** plain form body `configNum=<n>` (not base64).
- The hub expects a prompt 2xx; an empty `{}` works.

**Arming:**
- Delivery fires on a **real level change** (a move or stop to a new position). A `jog` and plain
  config writes (e.g. setting the LED) send no motion events.
- No handshake: after enabling, the hub stays silent until the next level change, then POSTs at once
  (the first motion POST is preceded by a `configNum=N`).
- No keep-alive needed: a move 110 s after enabling, with no traffic in between, still delivered.
- `active` tracks position verification (§5.1), not the feed, so it says nothing about whether
  events are being delivered.
- `enabled` resets across a hub reboot, so a consumer must re-enable periodically.

**Accuracy (60 moves on 3 shades, compared with a settled `refresh=true`):**
- No `stops` event for 1 move in 60 (1.7 %).
- 86 % of `stoppedPosition` values within 1 % of the refresh reading.
- After a mid-travel stop, `stoppedPosition` ran 3–4 % past where the shade came to rest (12 % on one
  shade with a weak stop), apparently reported before the motor finished coasting.

**Why it matters:** the feed reports where shades stopped — including after a stop, and moves made
from a remote or the app — without the motor-waking refresh reads, within the miss and error rates
above.

---

## 6. Hub

### 6.1 Identity and settings

- 📘 `GET /api/userdata` — `serialNumber`, `macAddress`, IP configuration, `firmware`, `rfID`, `color`,
  `times {timezone, localSunriseTimeInMinutes, localSunsetTimeInMinutes, currentOffset}` (the
  document's schema uses these names; its example shows `localSunriseInMinutes` /
  `localSunsetInMinutes`, which the hub does not use), and flags (`remoteConnectEnabled`,
  `setupCompleted`, `autoBackup`, `enableScheduledEvents`, …). With `?includeCounts=true` (the document
warns this is expensive and should not normally be used) it adds
  `roomCount`, `shadeCount`, `groupCount`, `sceneCount`, `sceneMemberCount`, `sceneControllerCount`,
  `sceneControllerMemberCount`, `multiSceneCount`, `multiSceneMemberCount`, `scheduledEventCount`,
  `repeaterCount` (verified 2026-11-26).
  🧪 On a hub that never reaches the cloud, `localSunriseTimeInMinutes` and `localSunsetTimeInMinutes`
  stay `0`, so
  sun-relative hub schedules would misfire.
- 📘 `GET /api/fwversion` — `{"firmware":{"mainProcessor":{name,revision,subRevision,build},"radio":{…}}}`.
  🧪 Generation test: `mainProcessor.revision == 1` → Gen 1 (no `radio` block); `== 2` → Gen 2.
- 🧪 `PUT /api/times` exists but returned `400 {}` for every body tried (the `userdata.times` object,
  a timezone alone, and timezone + offset + local time); the body format is unknown. There is no
  `GET /api/times` on Gen 2 (404). Time lives in `userdata.times`.

### 6.2 Status LED

📘 Reading: `GET /api/userdata` → `userData.color` `{red, green, blue (0–255), brightness (0–100)}`,
"the color of the LEDs". The document lists only GET for `/api/userdata`.

🧪 Setting: `PUT /api/userdata`
`{"userData":{"color":{"red":0-255,"green":0-255,"blue":0-255,"brightness":0-100}}}` changes the LED
and persists. The PUT **merges** (other `userData` fields are untouched) and echoes the full
`userData`. `{0,0,0,0}` turns it off; `brightness:0` with a non-zero colour is stored but leaves the
LED lit.

### 6.3 🧪 Other undocumented endpoints

- `GET /api/backup` — **an unauthenticated, plain-HTTP dump of the whole configuration**
  (`{"backUp":{"base64Encoded":…}}` → serial, rfID, MAC, every room, shade, group and scene). Anyone on
  the LAN can read the hub's identity. No credentials are included. POST/PUT (restore) not exercised.
- `GET /api/groups` — `{id, networkGroupNumber, roomId, shadeCount, type}`.
- `GET /api/homekit/ispaired` → `{"homeKit":{"paired":bool}}`.
- `GET /api/integrations` → `{"integrations":{"nest":null}}`.
- `GET /api/register` — whether the hub is registered to a cloud account (POST registers with
  `{"pvKey":"…"}`, DELETE deregisters; neither exercised).
- `GET /api/repeaters`, `/api/smartpowersupplies`, `/api/scheduledevents`, `/api/scenecollectionmembers`
  — present; empty on a simple setup.
- 🧪 Room detail: `GET /api/rooms` (📘) entries are `{id, name (base64), type, order, colorId, iconId}`;
  `type:2` is a hidden system "Default Room".

### 6.4 Battery

📘 `batteryStatus`: 0 = no status available, 1 = low, 2 = medium, 3 = high, 4 = plugged in.
📘 `batteryStrength`: "the current strength of the battery", unit not given.
🧪 `batteryStrength` is **pack voltage in tenths of a volt** (e.g. 177 = 17.7 V, against an ~18 V
pack), not a percentage. `batteryStrength:0` means "not yet polled", not empty.
Battery rides along on every plain read; only `?updateBatteryLevel=true` forces a fresh measurement.

---

## 7. 🧪 Firmware updates

The vendor document does not cover firmware updates. Observed: the hub fetches firmware from Hunter
Douglas itself; there is **no local upload path and no API to start an update**. On boot it calls
`GET https://homeauto.hunterdouglas.com/api/v2/firmware?revision=<packed>&hardware=<id>` (HTTP Basic,
username = serial, empty password — the server ignores the auth). `revision` is packed `0xRRSSBBBB`
(`2.0.1056` = `0x02000420`). The server replies with `fwUrl`/`rfUrl` and MD5s, but as of 2026-11-25
**every advertised firmware file returns 404**, for Gen 1 and Gen 2 hardware alike. A hub stays on
whatever build it last installed, and the image cannot be downloaded for analysis.
