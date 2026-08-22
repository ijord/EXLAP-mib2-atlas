# EXLAP on VW Group MIB2 HIGH — Protocol & Credential Guide

> ### ▶ [Open the interactive EXLAP atlas](https://ijord.github.io/EXLAP-mib2-atlas/)
> [GitHub repository](https://github.com/ijord/EXLAP-mib2-atlas) · Select one or more authorization profiles,
> receive the matching `OPEN-*` login, and inspect the resulting URL union.

**EXLAP** (Extended Link Access Protocol, also called SAI-Server) is a TCP + XML telemetry and control channel
on VW Group MIB2 HIGH infotainment systems. Companion applications use it to read live vehicle data such as
engine, dynamics, tyre and navigation values, and to invoke supported media, sound and vehicle controls.

The interactive atlas contains **260 interface constructors: 229 addressable URLs and 31 nested schema
types**. Availability varies by firmware, installed equipment, active FSIDs and authorization profile.

---

## Credentials

EXLAP auth is a per-user SHA-256 digest challenge/response. These names describe what the profiles are useful
for; the internal group name is retained alongside each one.

| Profile name | Internal group | Username | Password | Observed `Dir` coverage |
|---|---|---|---|---|
| Vehicle Telemetry | `vehicledata` | `OPEN-300000` | `O6z0lZ3/+IbaUmDxnd0tJ95p8K5pHvV/AoaOsJn6A/c=` | **75 measured** |
| Navigation Data | `navigation` | `OPEN-030000` | `f9uydlV5NXZbkQQ3la0MgfEVRebbnYftED9wT0Ie48g=` | **22 measured** |
| Eco & Driver Context | `ecohmi` | `OPEN-00C000` | `g8OFe0DOqeA3hAwikobi6dath7YDXHj7IHrLmVo9xxI=` | **26 measured** |
| Interior & Climate | `interior` | `OPEN-003000` | `B++ENIgKchRpzhwglOW3yvfvfRZJTjOFoeuf9HJMHhY=` | **31 measured** |
| Infotainment & Mixed Data | `misc` | `OPEN-000C00` | `gJZlPWrttYGuex7UlyI2GhyQ/3FTeMRknlYeaXH70TQ=` | **60 measured** |
| High-Performance Extensions | `vehicledata_performance` | `OPEN-000300` | `N+ae73tgz7vtuzGDubmMtWmlBrDAtn9gEy8y8/q8EuQ=` | **23 measured** |
| Rear Seat Entertainment Core | `general` | `OPEN-C00000` | `SGt5VscsjouAyYf2/diCOQ+V8kjojDBAdd7i48mg3Y4=` | **unknown** — requires Rear Seat Entertainment FSID |
| Rear Seat Entertainment Media | `media` | `OPEN-0C0000` | `e90OlX8IVYnMtySEeBMkdkZgrCTFwW6+autLlc9NE/Y=` | **unknown** — requires Rear Seat Entertainment FSID |
| Rear Seat Entertainment Premium Comfort | `interior_comfort_premium` | `OPEN-0000C0` | `dOAQWvPak/OnG1MEKJJOrmFz+mwvWKeNbIuXp8889fY=` | **unknown** — requires Rear Seat Entertainment FSID |
| Maximum Useful Union | all nine useful fields | `OPEN-FFFFC0` | `0XFdYtgsi0S3Cd7TPJTEI6tMupH3WXuET2OXP5ClV6o=` | **168 expected** on the reference unit; 229 registration candidates with Rear Seat Entertainment |

“Measured” is a count of unique URLs returned by that credential's isolated live `Dir` response. Profiles
overlap, so the counts are **not additive**. “Unknown” is not zero: the Rear Seat Entertainment-only credentials are rejected when
the Rear Seat Entertainment FSID is absent and must be measured on a suitable unit. `defaultGroup` is omitted
because its only URL is the VIN, which all useful profiles already expose. `group09` and `experimental` are
omitted because they have no assigned service path in this atlas.

### Interactive union builder

The web atlas allows any non-empty combination of the nine useful fields: **511 possible unions**. It encodes
the chosen fields at authorization value 3, displays the distinct URL union, and supplies its username and
password. The browser assets contain only the 511 precomputed results; they do **not** contain the master secret
or password-derivation logic.

The six profiles measured on the reference unit have exact object lists. Credentials combining Rear Seat
Entertainment fields are also exact, but their displayed Rear Seat Entertainment additions are a registration
superset until the Core/Media/Premium authorization split is captured on an equipped unit.

---

## How authentication works

```
1. TCP connect 127.0.0.1:25010 (on-car) — or the head unit's IP:25010 over its Wi-Fi/USB link
2. read greeting:  <Status><Init/></Status>
3. send:  <Req><Protocol version="1" returnCapabilities="true"/></Req>
4. send:  <Req><Authenticate phase="challenge"/></Req>
   recv:  <Rsp><Challenge nonce="<b64>"/></Rsp>
5. cnonce = base64(16 random bytes)
   digest = base64( SHA256( "user:password:nonce:cnonce" ) )     <-- each field TRUNCATED to 44 chars
   send:  <Req><Authenticate phase="response" user="<user>" cnonce="<cnonce>" digest="<digest>"/></Req>
   recv:  <Rsp/>            (success)
```

**Gotcha:** every field in the digest string is truncated to **44 chars** (`%.44s`). A client that passes the
full-length nonce computes the wrong digest and fails auth even with the correct password.

### Credential-generation boundary

The public atlas intentionally publishes neither the password-derivation secret nor derivation code. Its
browser asset is a lookup table containing only the 511 non-empty unions of the nine useful profiles. This
lets users obtain a credential for any supported union without turning the site into a general-purpose
credential generator. Keep usernames at 44 ASCII characters or fewer to avoid challenge-digest truncation.

### Username suffix: six hex digits, twelve two-bit fields

The final `-H0H1H2H3H4H5` must contain six uppercase hex digits. Each nibble `b3 b2 b1 b0` becomes two
values: `2*b3+b2` and `2*b1+b0`. Value 0 disables the field; values 1, 2, and 3 are increasing authorization
levels within that field. They are not equivalent. Value 2 adds control/action URLs in several groups, while
value 3 can add token-management URLs. The atlas publishes value 3 for every useful isolated field so each
profile represents that field's maximum observed directory.

| Field | Suffix bits | Authentication group | Bit weights | Required FSID/service | Atlas profile |
|---:|---|---|---|---|---|
| 0 | `H0[3:2]` | `general` | `H0.3=2`, `H0.2=1` | Rear Seat Entertainment | Rear Seat Entertainment Core (`OPEN-C00000`) |
| 1 | `H0[1:0]` | `vehicledata` | `H0.1=2`, `H0.0=1` | Vehicle Data | Vehicle Telemetry (`OPEN-300000`) |
| 2 | `H1[3:2]` | `media` | `H1.3=2`, `H1.2=1` | Rear Seat Entertainment | Rear Seat Entertainment Media (`OPEN-0C0000`) |
| 3 | `H1[1:0]` | `navigation` | `H1.1=2`, `H1.0=1` | Navigation | Navigation Data (`OPEN-030000`) |
| 4 | `H2[3:2]` | `ecohmi` | `H2.3=2`, `H2.2=1` | Vehicle Data | Eco & Driver Context (`OPEN-00C000`) |
| 5 | `H2[1:0]` | `interior` | `H2.1=2`, `H2.0=1` | Vehicle Data or Rear Seat Entertainment | Interior & Climate (`OPEN-003000`) |
| 6 | `H3[3:2]` | `misc` | `H3.3=2`, `H3.2=1` | Vehicle Data or Rear Seat Entertainment | Infotainment & Mixed Data (`OPEN-000C00`) |
| 7 | `H3[1:0]` | `vehicledata_performance` | `H3.1=2`, `H3.0=1` | High Performance | High-Performance Extensions (`OPEN-000300`) |
| 8 | `H4[3:2]` | `interior_comfort_premium` | `H4.3=2`, `H4.2=1` | Rear Seat Entertainment | Rear Seat Entertainment Premium Comfort (`OPEN-0000C0`) |
| 9 | `H4[1:0]` | `group09` | `H4.1=2`, `H4.0=1` | none assigned in this build | not published |
| 10 | `H5[3:2]` | `defaultGroup` | `H5.3=2`, `H5.2=1` | any active service | not published — VIN only, redundant |
| 11 | `H5[1:0]` | `experimental` | `H5.1=2`, `H5.0=1` | none assigned in this build | not published |

Live value-level comparison on the reference unit:

| Internal group | Value 1 | Value 2 | Value 3 | Higher-level additions |
|---|---:|---:|---:|---|
| `vehicledata` | 75 | 75 | 75 | none in `Dir` |
| `navigation` | 14 | 19 | 22 | route actions at 2; external-access token actions at 3 |
| `ecohmi` | 26 | 26 | 26 | none in `Dir` |
| `interior` | 29 | 31 | 31 | ambient-light and temperature control at 2 |
| `misc` | 41 | 57 | 60 | media/radio/sound actions at 2; token actions at 3 |
| `vehicledata_performance` | 22 | 23 | 23 | `stopWatch_control` at 2 |

`OPEN-000300` is deliberately isolated and returns only the 23 High-Performance Extension URLs. It is not the
complete Porsche Track Precision profile. `PHP-D22200` combines `vehicledata` value 1,
`navigation` value 2, `interior` value 2 and `vehicledata_performance` value 2. It also requests `general`
value 3, which contributes nothing on the non-Rear-Seat-Entertainment reference unit. Those active fields
produce a **138-URL union**—not the sum of their overlapping counts. Consequently, PTP receives ordinary
dynamics such as `yawRate` from Vehicle Data and the supplemental track objects from High-Performance
Extensions. Performance values 2 and 3 expose the same 23 directory URLs in this build, although the PTP
credential still requests value 2.
`OPEN-FFFFC0` requests all nine useful fields at value 3. In this atlas it has the same useful service
coverage as legacy `OPEN-FFFFFF`: fields 9 and 11 have no assigned path, while field 10 contributes only the
VIN already exposed by useful profiles. The live `OPEN-FFFFFF` reference capture returned 168 URLs; an
equipped Rear Seat Entertainment unit can register up to the 229 addressable URLs in this build, and other
firmware/equipment combinations can differ.

## Request verbs

`Subscribe`, `Get`, `Unsubscribe`, `Dir`, `Capabilities`, `ServerInfo`, `Interface`, `ObjectInfo`, **`Call`**,
`Heartbeat`, `Authenticate`, `Protocol`. There is **no `Set`** verb.

```
Dir:        <Req><Dir urlPattern="*" fromEntry="0" numOfEntries="300"/></Req>   (what YOUR tier can see)
Interface:  <Req><Interface url="coolantTemperature"/></Req>                    (descriptor: units/range/params)
Read once:  <Req><Get url="engineSpeed"/></Req>
Subscribe:  <Req><Subscribe url="engineSpeed" ival="100" content="true"/></Req> (ival ms; push on change)
Control:    <Req><Call url="Media_Pause"/></Req>                                (invoke an action object)
```

**EXLAP is read AND control.** Media/sound *actions* (`Media_Play`/`Pause`, `Sound_Mute`/`Unmute`/`Increase`/
`DecreaseVolume`, `Media_Next`/`PreviousTrack`) are invoked with `Call` and succeed (no write to arbitrary
values — there's no `Set` verb; you drive state through the action objects). Objects outside your tier, and
non-existent URLs, both return `<Rsp status="noMatchingUrl"/>` (they're indistinguishable from the client).

The atlas classifies **177 `<Object>` URLs as Read** and **52 `<Function>` URLs as Call/control**. The 31
nested schema types are not directly requestable. EXLAP has no generic `Set` verb.

---

## Notes

- **Values read `nodata`/`EC_01 "not existing"`** until the relevant subsystem is active (ignition on, media
  playing, a control unit fitted). Not every catalog object is wired on every model/trim.
- **Update rate:** vehicle dynamics top out at **10 Hz** (`ival="100"`); the server is change-based and
  ival-capped. Temps/pressures update ~1 Hz.
- **GPS:** `Nav_GeoPosition` (lat/lon) is exposed; on this car the head unit's own GNSS also publishes a fix
  outside EXLAP. No satellite-count object.
- **Descriptions:** each object carries a German `@description` in its `Interface` response (units, semantics).
