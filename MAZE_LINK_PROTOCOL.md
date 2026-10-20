# MazeLink Wire Protocol

**Version:** 1.0  
**Status:** Reference specification  
**Project:** MazeLink — a two-peer arcade maze synchronisation prototype

## 1. Overview

MazeLink is a compact, binary, application-layer protocol for keeping two independent instances of an arcade maze game in sync. Each peer owns a maze, a player avatar, and a set of four roaming adversaries. Avatars can temporarily cross into the other peer's maze. During a session, both peers exchange snapshots, motion samples, gameplay events, and changes to shared state.

This specification defines the on-wire representation and how a receiver should interpret it. Rendering, sound, input controls, collision detection, adversary AI, and other local simulation details are outside its scope.

The terms **MUST**, **SHOULD**, and **MAY** express protocol requirements in the sense of RFC 2119.

## 2. Peer and World Model

The same four perspectives apply independently at each peer:

| Term | Meaning |
| --- | --- |
| **HOME** | An entity owned by this peer and currently playing in its own maze. |
| **VISITING** | This peer's avatar while it is playing in the other peer's maze. |
| **MIRROR** | A local representation of an entity owned by the other peer, normally active in that peer's maze. |
| **GUEST** | The other peer's avatar while it is playing in this peer's maze. |

**Home world** and **remote world** are always relative to the receiving peer. The words “sender” and “receiver” refer to the direction of a specific message; neither role implies permanent authority over the session.

## 3. Connection and Frame Format

### 3.1 Transport

MazeLink uses one TCP connection between two peers. One peer accepts the incoming connection and the other initiates it. These roles matter only for the connection and authentication exchange. Once authenticated, application messages are exchanged in both directions.

The handshake below authenticates a shared secret at the application layer; it **does not encrypt traffic**. Do not expose this protocol to untrusted networks without a secure transport such as TLS.

### 3.2 Frame layout

Every message is encoded as a length-prefixed frame:

| Offset | Size | Field | Description |
| --- | --- | --- | --- |
| 0 | 2 bytes | `size` (`u16`) | Number of bytes that follow, including the opcode |
| 2 | 1 byte | `opcode` (`u8`) | Message identifier |
| 3 | `size - 1` bytes | `body` | Message-specific fields |

All multi-byte integers use **network byte order (big-endian)**. Valid `size` values are **1 through 16,384**, inclusive. A peer **MUST** terminate the connection if it receives a size outside that range.

TCP provides a stream of bytes, not message boundaries. A decoder **MUST** read two bytes for `size`, accumulate exactly `size` additional bytes, dispatch that frame, and then continue decoding any bytes already buffered. A single TCP read may contain part of a frame or multiple frames.

### 3.3 Primitive types

| Type | Encoded size | Encoding |
| --- | --- | --- |
| `u8` | 1 byte | Unsigned 8-bit integer |
| `u16` | 2 bytes | Unsigned 16-bit integer, big-endian |
| `u32` | 4 bytes | Unsigned 32-bit integer, big-endian |
| `flag` | 1 byte | `0` = false, `1` = true |
| `octets[n]` | `n` bytes | Uninterpreted bytes |

The protocol uses unsigned fixed-point numbers for non-negative spatial values:

- `pos_x` and `pos_y` are `u32` values measured in **1/256 pixel** units.
- `velocity` is a `u16` value measured in **1/256 of the game engine's speed unit**, using the same unit convention at both peers.
- Encoding is `floor(value × 256 + 0.5)`; decoding divides the stored number by `256`.
- An encoder **MUST** ensure that the scaled value fits the field width.

For example, a horizontal position of `12.75` pixels is encoded as `3264` (`0x00000CC0`).

Tile positions, by contrast, are grid coordinates. `(0, 0)` is the upper-left tile; increasing `x` moves right and increasing `y` moves down.

## 4. Session Setup

The connecting peer **MUST** send `HELLO` as its first application message. Its payload is:

| Field | Type | Constraint |
| --- | --- | --- |
| `revision` | `u8` | `1` |
| `secret_length` | `u8` | `0..32` |
| `secret` | `octets[secret_length]` | ASCII encoded shared secret |

The accepting peer validates the revision and secret, then replies with either:

- `WELCOME`: `revision: u8` with value `1`, establishing the session; or
- `REJECT`: `cause: u8` where `0` means unspecified, `1` means incorrect secret, and `2` means unsupported revision. The accepting peer **MUST** close the connection after `REJECT`.

Following `WELCOME`, **each** peer **MUST** transmit the following initial state messages, in any order: `WORLD_IMAGE`, `POINTS_STATE`, `HEALTH_STATE`, and `ROUND_STATE`.

The two sides should not treat an initial state as complete until all four messages have been received from the other side.

## 5. Message Registry

| Opcode | Message | Purpose |
| --- | --- | --- |
| `0x01` | `HELLO` | Open an authenticated session |
| `0x02` | `WELCOME` | Confirm session establishment |
| `0x03` | `REJECT` | Report handshake failure |
| `0x10` | `WORLD_IMAGE` | Transmit an entire tile grid |
| `0x11` | `ROUND_STATE` | Report the game phase |
| `0x12` | `POINTS_STATE` | Report the current score |
| `0x13` | `HEALTH_STATE` | Report remaining lives |
| `0x20` | `AVATAR_MOTION` | Publish avatar position and movement |
| `0x21` | `ENEMY_MOTION_SET` | Publish four enemy states together |
| `0x30` | `AVATAR_ENTER` | Announce a visit to the other maze |
| `0x31` | `AVATAR_EXIT` | End the visit |
| `0x32` | `AVATAR_LOST` | Report an avatar death |
| `0x33` | `RETURN_REQUEST` | Request that a visiting avatar return home |
| `0x40` | `PICKUP_EVENT` | Reconcile an item consumed in a maze |
| `0x41` | `ENEMY_DEFEATED` | Report a defeated home-world enemy |

Numeric opcodes are deliberately fixed for this protocol version; names are symbolic labels and do not appear on the wire.

## 6. Enumerated Values

### 6.1 Heading (`u8`)

| Value | Meaning |
| --- | --- |
| `0` | `IDLE` |
| `1` | `NORTH` |
| `2` | `EAST` |
| `3` | `SOUTH` |
| `4` | `WEST` |

### 6.2 Enemy behaviour (`u8`)

| Value | Meaning |
| --- | --- |
| `0` | `PATROL` |
| `1` | `PURSUE` |
| `2` | `VULNERABLE` |
| `3` | `VULNERABLE_CONFINED` |
| `4` | `RETURNING` |

### 6.3 Round phase (`u8`)

| Value | Meaning |
| --- | --- |
| `0` | `INITIALISING` |
| `1` | `ACTIVE` |
| `2` | `BONUS_ACTIVE` |
| `3` | `LIFE_LOST` |
| `4` | `FINISHED` |
| `5` | `INTERMISSION` |
| `6` | `RESTART_READY` |

A decoder **MUST** reject unknown enumeration values for messages requiring those enums.

## 7. Message Bodies

The following tables describe the **body only**; the two-byte length and one-byte opcode precede it.

### 7.1 `WORLD_IMAGE` (`0x10`)

| Field | Type | Meaning |
| --- | --- | --- |
| `stage` | `u16` | Maze or stage identifier |
| `columns` | `u8` | Number of columns |
| `rows` | `u8` | Number of rows |
| `tiles` | `octets[columns × rows]` | Row-major tile codes |

For a tile at `(x, y)`, the array index is `y × columns + x`.

| Tile code | Meaning |
| --- | --- |
| `0` | Open passage |
| `1` | Solid barrier |
| `2` | Small collectible |
| `3` | Energiser |
| `4` | Portal entrance A |
| `5` | Portal entrance B |

`columns` and `rows` **MUST** both be greater than zero. The decoder **MUST** verify that the remaining body contains exactly `columns × rows` bytes and that every tile code is in `0..5`.

### 7.2 `ROUND_STATE` (`0x11`)

Body: `phase: u8`, containing one valid round-phase value.

### 7.3 `POINTS_STATE` (`0x12`)

Body: `points: u32`.

### 7.4 `HEALTH_STATE` (`0x13`)

Body: `lives: u8`. Receivers **MAY** disregard values outside a game-appropriate range; `0..8` is recommended.

### 7.5 `AVATAR_MOTION` (`0x20`)

The sender publishes its own avatar's most recent motion sample. A receiving peer applies that state to the appropriate local representation.

| Body offset | Size | Field | Type |
| --- | --- | --- | --- |
| 0 | 2 | `sequence` | `u16` |
| 2 | 4 | `pos_x` | `u32` |
| 6 | 4 | `pos_y` | `u32` |
| 10 | 1 | `heading` | `u8` |
| 11 | 2 | `velocity` | `u16` |

Body size: **13 bytes**. A receiver **MUST** apply the sequence freshness rule in Section 8.

### 7.6 `ENEMY_MOTION_SET` (`0x21`)

A peer sends the current states of all four enemies it owns in one atomic sample.

| Field | Type | Notes |
| --- | --- | --- |
| `sequence` | `u16` | Freshness counter for this message stream |
| `entries` | `u8` | **MUST** equal `4` |
| `enemy[0..3]` | Repeated record | Four records, as defined below |

Each record has these fields in this order:

| Field | Type | Notes |
| --- | --- | --- |
| `enemy_id` | `u8` | Must be `0..3`, once each |
| `pos_x` | `u32` | Fixed-point position |
| `pos_y` | `u32` | Fixed-point position |
| `heading` | `u8` | Heading enum |
| `velocity` | `u16` | Fixed-point speed |
| `behaviour` | `u8` | Enemy-behaviour enum |

Each enemy record is 13 bytes, making the body **55 bytes** (`2 + 1 + 4 × 13`). Receivers **MUST** validate the count, identifiers, and enum values, and **MUST** apply the freshness rule in Section 8.

### 7.7 `AVATAR_ENTER` (`0x30`)

No body. The sender declares that its avatar has left its home world and is now represented as a **GUEST** in the receiver's world.

### 7.8 `AVATAR_EXIT` (`0x31`)

No body. The sender's avatar is no longer present as a guest in the receiver's world.

### 7.9 `AVATAR_LOST` (`0x32`)

No body. Reports a death event involving the sender's avatar that needs to be reflected at the opposite peer.

### 7.10 `RETURN_REQUEST` (`0x33`)

No body. The receiver asks the sender to bring its **VISITING** avatar back to its home maze. The sender **SHOULD** restore its local state and then send `AVATAR_EXIT`.

### 7.11 `PICKUP_EVENT` (`0x40`)

| Body offset | Field | Type | Meaning |
| --- | --- | --- | --- |
| 0 | `world_selector` | `u8` | `0`: receiver's home world; `1`: receiver's mirrored remote world |
| 1 | `tile_x` | `u8` | Tile column |
| 2 | `tile_y` | `u8` | Tile row |
| 3 | `energiser` | `flag` | Whether the consumed item was an energiser |

The receiver removes the item at the indicated coordinates in the selected maze representation. When `energiser == 1` and `world_selector == 0`, it **MUST** apply its own local energiser transition, making eligible enemies vulnerable. When `world_selector == 1`, it **MUST NOT** change the behaviour of its locally owned enemies; the other peer reports its enemy behaviour through later state messages.

### 7.12 `ENEMY_DEFEATED` (`0x41`)

Body: `enemy_id: u8` in `0..3`. Indicates that the sender's visiting avatar defeated one of the receiver's home-world enemies while that enemy was vulnerable.

## 8. Sampling and Ordering

### 8.1 Transmission cadence

- `AVATAR_MOTION`: **SHOULD** be emitted approximately **20 times per second**, plus an immediate update following portal transitions.
- `ENEMY_MOTION_SET`: **SHOULD** be emitted approximately **20 times per second**.
- Points, lives, and round-phase messages: **SHOULD** be sent when their values change.
- Visit transitions and pickup/death events: **SHOULD** be sent promptly after the event.

### 8.2 Sequence comparison

Motion streams carry separate `u16` counters. The sender increments the counter for each message in its stream, wrapping from `65535` to `0`. Receivers keep a separate `last_sequence` for avatar motion and enemy-set motion.

For a newly received value, calculate:

`distance = (incoming_sequence - last_sequence) mod 65536`

The incoming sample is newer exactly when `1 <= distance <= 32767`. Only a newer sample is applied, and `last_sequence` is then updated. A duplicate or older sample is ignored. If no prior sample exists for that stream, accept the first valid sample and initialise `last_sequence`.

**Note:** TCP preserves delivery order within one connection. Sequence comparisons remain useful for detecting duplicates and for ordering application-level samples after state resets; they are not a replacement for connection/session management.

## 9. Validation and Recovery

A receiver **MUST** reject an individual message and may log the issue when any of the following occurs:

- The body has the wrong length or contains trailing bytes not defined for that message.
- A `flag` field is neither `0` nor `1`.
- An enumeration or world selector has an unknown value.
- An enemy identifier is not in `0..3`, or an enemy-set record count is not `4`.
- A tile coordinate is outside the targeted grid.
- A `WORLD_IMAGE` does not match its declared dimensions.

Unknown opcodes **MUST** be skipped using the advertised frame size so that the next valid frame can be decoded. Invalid individual message bodies are ignored without closing the connection; invalid **frame sizes** terminate the connection as described in Section 3.

For consistency, event handlers should avoid applying a consumed-item or defeated-enemy event twice when the underlying game state already reflects it. The wire format does not include unique event IDs, so higher-level replay protection is an implementation concern.

## Appendix A. Wire Examples (Informative)

Each example is shown as **length | opcode | body**, with spaces between bytes.

### A.1 `POINTS_STATE` — 420 points

- Opcode: `0x12`
- Body: `u32(420) = 00 00 01 A4`
- Length: `1 + 4 = 5 = 00 05`

```hex
00 05 12 00 00 01 A4
```

### A.2 `HELLO` — revision 1 with secret `demo-key`

ASCII bytes for `demo-key`: `64 65 6D 6F 2D 6B 65 79` (8 bytes).

- Body: revision (`01`), length (`08`), secret (8 bytes)
- Frame length field: `1 + 1 + 1 + 8 = 11 = 00 0B`

```hex
00 0B 01 01 08 64 65 6D 6F 2D 6B 65 79
```

### A.3 `AVATAR_MOTION` — sample number 55

Position `(12.75, 80.0)` pixels, heading east, speed `0.8`.

| Field | Encoded hex |
| --- | --- |
| `sequence = 55` | `00 37` |
| `pos_x = 12.75 × 256 = 3264` | `00 00 0C C0` |
| `pos_y = 80 × 256 = 20480` | `00 00 50 00` |
| `heading = EAST` | `02` |
| `velocity = round(0.8 × 256) = 205` | `00 CD` |

- Body: 13 bytes
- Length: `1 + 13 = 14 = 00 0E`

```hex
00 0E 20 00 37 00 00 0C C0 00 00 50 00 02 00 CD
```

### A.4 `PICKUP_EVENT` — energiser at tile `(14, 17)` in receiver's home world

- Body: `world_selector=0`, `tile_x=14`, `tile_y=17`, `energiser=1`
- Length: `1 + 4 = 5 = 00 05`

```hex
00 05 40 00 0E 11 01
```

## Appendix B. Implementation Notes

- Build a buffered state-machine decoder; do not assume that a single `recv` returns a complete message.
- Use independent inbound and outbound queues so state bursts do not block rendering.
- Keep maze ownership explicit: a world selector is interpreted **relative to the receiver**, never as a global maze ID.
- Check fixed-point overflow, integer widths, byte order, and payload lengths before serialising or applying a state update.
- Test handshakes, fragmented frames, back-to-back frames, unknown opcodes, malformed arrays, sequence wraparound, and both directions of avatar travel.

---

*This is a general-purpose reference specification for a personal networking prototype. It is not an official standard or a claim of production-level security.*
