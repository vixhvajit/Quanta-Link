# Quanta-Link

[![License](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)

A lightweight, quantum-safe link protocol for drones. Quanta-Link encrypts and
authenticates the MAVLink command and telemetry link between a drone and its
ground station, using post-quantum key exchange. It sits between the radio and
the equipment at each end, so neither the flight controller firmware nor the
ground control software has to change.

## Status

Design stage. This README is the plan: the protocol is not yet specified in
full and nothing is implemented. See the [Roadmap](#roadmap).

## The problem

MAVLink is the protocol that ArduPilot, PX4 and most ground stations use to
command a drone and read its telemetry. By default it has no security at all:

- **Anyone can listen.** With a matching radio and open-source tools, a
  bystander reads the drone's position, home location, mission and battery
  state. A SiK radio's NETID only selects a channel; it is not a password.
- **Anyone can command.** The same bystander can send a disarm, a mode change
  or a new mission, and the flight controller accepts it.

MAVLink 2 message signing fixes the second problem but not the first. It
appends a 48-bit truncated SHA-256 signature, keyed by a 32-byte pre-shared
secret, so forged commands are rejected. It adds no encryption, has no key
exchange, and uses one static key that is never rotated. It is also off by
default.

Links that do encrypt, such as OpenHD, wfb-ng or a WireGuard tunnel, use
classical public-key cryptography. Traffic recorded today can be decrypted once
a large enough quantum computer exists. For a defence drone, a recorded flight
log, route or video feed can stay sensitive for years, so this "harvest now,
decrypt later" risk is real.

## What "quantum-safe" means here

There are two ways to defend a link against quantum computers, and Quanta-Link
uses the practical one.

- **Post-quantum cryptography (PQC)** is ordinary mathematics that quantum
  computers are not known to break. NIST standardised it in August 2024:
  ML-KEM (FIPS 203) for key exchange and ML-DSA (FIPS 204) for signatures. It
  runs on a microcontroller. **Quanta-Link uses PQC.**
- **Quantum key distribution (QKD)** sends key material as single photons. It
  needs optical quantum hardware and a line of sight, and on drones it is still
  research-grade. It is not lightweight, so Quanta-Link does not depend on it.
  The key schedule takes keys from a pluggable source, so a QKD-delivered key
  can be mixed in later without changing the protocol.

## Design goals

- **Lightweight.** About 21 bytes of overhead per packet. The handshake fits a
  slow telemetry radio, and the air node runs on an ESP32-S3.
- **Quantum-safe.** Session keys come from a hybrid of X25519 and ML-KEM-768,
  so an attacker has to break both.
- **Transparent.** No changes to ArduPilot, PX4, QGroundControl or Mission
  Planner. Both ends still see plain MAVLink.
- **Fails closed.** A packet that does not authenticate is dropped. There is
  no fallback to plaintext.
- **Built from vetted parts.** No home-made cryptographic primitives; only
  established, constant-time libraries.

## How it works

Quanta-Link is a "bump in the wire". An air node sits between the flight
controller and the radio, and a ground node sits between the radio and the
ground station software. Each end sees ordinary MAVLink; everything on the air
is ciphertext.

```
FC --UART--> [air node] --radio--> [ground node] --UDP 14550--> QGC / Mission Planner
```

### Transports

The protocol core is independent of the radio. Two transports are planned:

**A. Over OpenHD (UDP).** The air node runs on the same Raspberry Pi as
OpenHD, and the encrypted frames travel over OpenHD's wifibroadcast link.
OpenHD routes MAVLink and drops anything else, so each encrypted frame is
carried inside a standard MAVLink `TUNNEL` message (id 385, 128-byte payload),
which OpenHD forwards without being able to read.

```
FC --UART--> Pi: Quanta-Link air --> OpenHD air ==wfb 5.8 GHz==> OpenHD ground --UDP--> Quanta-Link ground --> GCS
```

**B. Separate control link (serial).** The air node drives its own SiK
telemetry radio on 433 MHz, and OpenHD carries video only. If the 5.8 GHz video
link is jammed, the control link on a different band survives.

```
FC --UART--> Pi or ESP32-S3: Quanta-Link air --UART--> SiK 433 MHz ~~~> SiK --USB--> Quanta-Link ground --> GCS
```

Either way, **Quanta-Link must be the only path to the flight controller.**
If OpenHD keeps its own MAVLink connection to the flight controller, an
attacker can go around Quanta-Link through it, so OpenHD's flight-controller
telemetry is switched off.

### Pairing

The air and ground nodes are paired once, on the bench, over USB. Pairing
stores a shared secret on both sides. No pairing traffic is ever sent over the
radio.

### Handshake

Each session starts with a handshake modelled on the Noise Protocol
Framework's pre-shared-key patterns, with a hybrid KEM in place of
Diffie-Hellman alone:

1. The ground node sends a fresh X25519 public key (32 bytes) and a fresh
   ML-KEM-768 encapsulation key (1,184 bytes).
2. The air node replies with its own X25519 public key and an ML-KEM-768
   ciphertext (1,088 bytes).
3. Both sides derive the session keys from the X25519 shared secret, the
   ML-KEM shared secret and the paired secret.

The keys are fresh for every session, which gives forward secrecy: a key stolen
later does not decrypt sessions recorded earlier. Mixing in the paired secret
authenticates both ends without sending signatures over the air. An ML-DSA
signature is 2.4 KB, which is too large for a slow radio.

The whole handshake is about 2.3 KB, split into fragments that fit a SiK
frame. That is roughly 0.3 s of raw airtime at SiK's default 64 kbps air
rate, and somewhat more in practice. It happens once per session.

### Data packets

Each MAVLink frame is encrypted with ChaCha20-Poly1305 under a 256-bit session
key:

| Field | Size |
|---|---|
| Header (packet type, key epoch) | 1 byte |
| Sequence number (the low bits of a 64-bit counter) | 4 bytes |
| Encrypted MAVLink frame | same as the plaintext |
| Authentication tag | 16 bytes |

The nonce is built from the direction and the full counter, so it is never
reused within a session. The receiver keeps a sliding window of sequence
numbers and drops replays. Keys are rotated after a set number of packets or
minutes, and every boot starts a new session, so a reboot cannot reuse a nonce.

### Software

Planned:

- **Core:** C11, so the same code runs on Linux and on an ESP32.
- **ML-KEM:** [mlkem-native](https://github.com/pq-code-package/mlkem-native).
- **X25519, ChaCha20-Poly1305, BLAKE2b:** [Monocypher](https://monocypher.org).
- **Tests and attack scripts:** Python with pymavlink, against ArduPilot SITL.

## Threat model

| Attack | MAVLink 2, unsigned (default) | MAVLink 2, signed | Quanta-Link |
|---|---|---|---|
| Read position, mission, telemetry | Trivial | Trivial: signing adds no encryption | No: only packet sizes and timing leak |
| Inject commands (disarm, mode, mission) | Trivial | No, not without the key | No, not without the key |
| Replay captured commands | Works | Blocked by timestamps | Blocked by sequence numbers |
| Decrypt recordings later with a quantum computer | Nothing to decrypt: never encrypted | Same | No: ML-KEM |
| Use a stolen key | Not applicable | Forge commands indefinitely | Only that one pairing, and recorded sessions stay secret |
| Jam the radio | Yes | Yes | Yes |

Quanta-Link does not make a drone unhackable. It makes the radio link stop
being the weak point, so an attacker has to jam it, capture the hardware, or
find a bug in the implementation. What remains, and how the design handles it:

1. **Jamming.** No cryptography prevents it. The link fails closed, and the
   flight controller's own lost-link failsafe (return to launch or land) takes
   over.
2. **A captured drone.** Keys stored in flash could be extracted. Mitigations:
   ESP32 flash encryption and secure boot, a separate pairing per drone, and
   revocable keys.
3. **Implementation bugs.** The serious ones are nonce reuse, which breaks
   ChaCha20-Poly1305 completely, and the fragment reassembly parser, which is a
   classic overflow risk. Mitigations: a new session on every boot, constant-time
   libraries only, and fuzzing the parser.
4. **Other ways in.** Quanta-Link protects the MAVLink link only. The RC
   receiver link, a USB port, or an OpenHD MAVLink path left open are all
   separate ways to command the drone, and are out of scope.
5. **Downgrade.** There is no plaintext fallback to downgrade to.
6. **Traffic analysis.** Packet sizes and timing are visible, and can hint at
   the flight phase. Padding is a possible later addition.

### Compared with the alternatives

- **MAVLink 2 signing:** authentication only, 13 bytes per packet, one static
  key. Not broken by quantum computers, but it hides nothing.
- **WireGuard with a pre-shared key:** strong, but it needs an IP link (a SiK
  radio is serial), adds about 60 bytes per packet over IPv4, and its quantum
  protection depends on the static pre-shared key never leaking. ML-KEM gives
  forward secrecy against a quantum attacker as well.
- **OpenHD or wfb-ng encryption:** classical public-key cryptography, and
  OpenHD derives its keys from a shared bind phrase. Fine against today's
  attackers, not against recordings decrypted later.

## Hardware

| Stage | Needed |
|---|---|
| Specification, simulation, attack demo (M1–M3) | A laptop running ArduPilot SITL. No extra hardware. |
| Transport A, over OpenHD | A working OpenHD air and ground pair (Raspberry Pi and a supported Wi-Fi adapter on each end). The air node runs on the OpenHD air Pi. |
| Transport B, separate control link | A SiK telemetry radio pair on 433 MHz, the licence-free band in India (915 MHz is not). |
| Lightweight air node | An ESP32-S3 board (such as the DevKitC-1), powered from the flight controller's telemetry port. |
| Bench testing | An ArduPilot or PX4 flight controller on USB power, with no props; a JST-GH telemetry cable; a USB-UART adapter for debugging. |
| Over-the-air attack demo (optional) | A second SiK pair, one radio of which plays the attacker. |

## Roadmap

- [x] **M0 — Plan.** This README: problem, threat model, design, roadmap.
- [ ] **M1 — Specification.** `docs/SPEC.md`: frame format, handshake, key
  schedule, rekeying, fragmentation and failure behaviour.
- [ ] **M2 — Simulation.** Linux air and ground nodes, with ArduPilot SITL
  flying a mission through the tunnel over a simulated radio with packet loss
  and latency.
- [ ] **M3 — Attack demo.** Spoofing, replay and eavesdropping against plain
  MAVLink 2, signed MAVLink 2 and Quanta-Link, side by side.
- [ ] **M4 — Hardware.** Transport A on the OpenHD Pi, transport B on SiK
  radios, then the ESP32-S3 air node. Measure latency, handshake time,
  per-packet overhead, CPU load and power.
- [ ] **M5 — Write-up.** Formal analysis of the handshake (Tamarin or
  ProVerif), results, and the hook for a QKD key source.
- [ ] **M6 — Video channel.** Secure FPV video as a second Quanta-Link
  channel, not a separate protocol. The same handshake derives separate keys
  for video and telemetry in each direction. The encoded stream (H.264 or
  H.265 over RTP) is protected with forward error correction first, then each
  over-the-air packet is encrypted and authenticated on its own, so a lost
  packet never breaks the next one and forged packets are dropped before they
  reach the error-correction decoder. It runs as a bump in the wire in front
  of wfb-ng, which takes plain UDP input, or OpenHD if it can take video from
  a local process. It needs a Linux board with a hardware video encoder, such
  as a Raspberry Pi 4. Constant-bitrate padding, against traffic analysis of
  the bitrate, is optional.

Later: run both transports at once and fail over between them, so the drone
keeps a protected control link if one band is jammed.

## License

Apache License 2.0 — see [LICENSE](LICENSE) and [NOTICE](NOTICE).

Copyright 2026 Vishvajit S.
