# Rollback Netcode Layer — Design & Build Guide

**Project:** Deterministic lockstep networking with rollback, client-side prediction, and lag compensation for 2–8 player matches
**Language:** C++ (or Rust)
**Status of this document:** planning + reference

---

## Table of contents

1. [Executive summary and scope](#1-executive-summary-and-scope)
2. [Reality check](#2-reality-check)
3. [Determinism](#3-determinism)
4. [State save and restore](#4-state-save-and-restore)
5. [The rollback loop](#5-the-rollback-loop)
6. [Rollback-safe simulation](#6-rollback-safe-simulation)
7. [Transport and topology](#7-transport-and-topology)
8. [Desync detection and diagnosis](#8-desync-detection-and-diagnosis)
9. [The server-authoritative alternative](#9-the-server-authoritative-alternative)
10. [Performance](#10-performance)
11. [Tech stack and setup](#11-tech-stack-and-setup)
12. [Repository layout](#12-repository-layout)
13. [Milestone ladder](#13-milestone-ladder)
14. [Reference implementations](#14-reference-implementations)
15. [Testing](#15-testing)
16. [Stretch goals](#16-stretch-goals)
17. [References](#17-references)

---

## 1. Executive summary and scope

### The original statement

> Deterministic lockstep networking with rollback, client-side prediction, and lag compensation for 2–8 player matches.

Five findings reshape this, and the first is that the statement describes two incompatible systems:

1. **"Deterministic lockstep with rollback" and "lag compensation" are different architectures, and you pick one.** Rollback lockstep has no authoritative server — every peer simulates the identical game from inputs alone. **Lag compensation is a server-authoritative technique**: the server keeps ~1 second of position history, estimates when a client's command was issued, and rewinds *other players'* hitboxes to that moment for hit detection. There is no server in lockstep to do that rewinding, and nothing to rewind to, because everyone already agrees on the state. See section 2.1.
2. **"Client-side prediction" means two different things in the two models**, which is probably the source of the conflation. In rollback it means *carry-forward prediction of remote inputs* — assume the opponent keeps doing what you last heard. In server-authoritative netcode it means *predicting your own movement ahead of server confirmation*. Same phrase, different mechanism. See section 2.2.
3. **Rollback quality is set by the worst peer, which makes 8 players a structurally bad fit.** If each player independently has a 10% chance of a poor link, the probability that at least one is bad rises from **19% at 2 players to 57% at 8**. Everyone rolls back on the worst connection. This is why GGPO's canonical use is 1v1 fighting games and why serious rollback projects ship 2-player-only. See section 2.3.
4. **Rollback demands two things simultaneously that most engines can't provide**: a bit-exact deterministic simulation, *and* the ability to save and restore the entire game state cheaply every single frame — in practice, one contiguous `memcpy`. That second requirement reaches into every gameplay system you will ever write. See section 4.
5. **Floating point is the main obstacle to determinism, and the practical answer is fixed-point arithmetic.** Transcendentals are not correctly rounded and differ across libm implementations, FMA contraction changes results, `-ffast-math` reassociates, and x87 80-bit intermediates differ from SSE. Current deterministic netcode libraries ship 32.32 fixed-point plus a seeded PRNG for exactly this reason. See section 3.

### Revised project statement

> A deterministic rollback lockstep layer for **2–4 players** built on a 32.32 fixed-point simulation core with a contiguous POD state arena, carry-forward input prediction, adaptive input delay, redundant UDP input transport over a relay-capable topology, and hierarchical desync detection with automatic state-dump diffing — **plus a clearly documented decision record explaining why lag compensation is not part of this architecture**, and what to build instead if 5–8 player support is required.

### The architecture decision

Make this explicitly, in writing, before anything else:

| | **Rollback lockstep** (GGPO-style) | **Server-authoritative** (Source-style) |
|---|---|---|
| Authority | None — all peers equal | Server |
| Sent over the wire | Inputs only | State snapshots (down), inputs (up) |
| Determinism required | **Absolutely** | Not at all |
| Prediction | Remote inputs (carry-forward) | Your own movement |
| Correction | Rollback + re-simulate | Server reconciliation |
| **Lag compensation** | **Not applicable** | **Yes — rewind hitboxes** |
| Cheat resistance | Poor (full state on every client) | Good |
| Scales to 8 players | Poorly (§2.3) | Yes |
| Server cost | None or a cheap relay | Real |
| Best for | Fighting games, tightly-coupled melee, block/parry/dodge | Shooters, large player counts |

**Recommendation: rollback lockstep, scoped to 2–4 players.** It is the more interesting engineering problem, it's what the "deterministic" and "rollback" words in the statement point at, and it's genuinely the right answer for tightly-coupled competitive mechanics. If 8-player support is a hard requirement, section 9 describes what you'd build instead — and it's a different project, not a mode.

### Explicit non-goals

- **Not both architectures.** A layer that does rollback *and* server-authoritative lag compensation well is two libraries.
- **Not cheat-resistant.** Lockstep gives every client the full game state. Map hacks are trivial. Acceptable for 1v1 with reputation systems; unacceptable for ranked FPS.
- **Not an engine.** You provide the netcode layer; the game provides a deterministic simulation that satisfies your contract.
- **Not floating-point.** See section 3.2.
- **Not late-join in v1.** Joining a running lockstep match requires full state transfer and a resync protocol. Real, and separable.

---

## 2. Reality check

### 2.1 Lag compensation belongs to the other architecture ★

Valve's description of lag compensation, paraphrased: the server keeps a history of every player's recent positions for about one second. When it executes a user command, it estimates when that command was created — `server_time − packet_latency − client_view_interpolation` — moves all *other* players back to where they were at that moment, performs hit detection, then restores them.

Every part of that requires a server that holds authoritative history and can rewind other entities relative to one client's perspective.

**In rollback lockstep there is no such thing.** Every peer simulates the same game from the same inputs and arrives at the same state. There is no "other player's position according to the server" to rewind, because there is no server and no disagreement to reconcile. When a remote input arrives late and differs from the prediction, you don't compensate — you roll back and re-run, and everyone ends up at the same state.

If you find yourself wanting lag compensation in a lockstep design, that's a signal you actually want the server-authoritative model (§9).

### 2.2 Two things called "client-side prediction"

| | Rollback lockstep | Server-authoritative |
|---|---|---|
| What is predicted | **Remote players' inputs** | **Your own movement** |
| Prediction method | Carry-forward: assume the last input repeats | Run the same movement code the server will run |
| When it's wrong | Roll back to the last confirmed frame, re-simulate | Server sends a correction; client replays local inputs from it |
| Visible artifact | Remote characters snap/teleport | Your own character snaps |

Carry-forward prediction works well specifically because fighting-game inputs are **digital or heavily quantized** — a d-pad direction plus a few buttons. A player holding forward keeps holding forward most frames, so the prediction is usually right and rollbacks are usually short.

**It works much worse with continuous analog input.** A mouse-look delta changes every frame, so carry-forward is wrong every frame, so you roll back every frame. If your game has free-aim mouse input, that's a strong argument against lockstep before you consider anything else.

### 2.3 The player-count math ★

Two compounding problems.

**Rollback depth is set by the worst peer.** You must roll back far enough to cover the latency of whoever is furthest behind. One player at 150 ms means everyone rolls back roughly 9 frames at 60 Hz — including the two players sitting next to each other on fiber.

**More peers means a worse worst peer.** If each player independently has a 10% chance of a poor connection:

| Players | P(at least one bad link) |
|---|---|
| 2 | **19%** |
| 4 | 34% |
| 6 | 47% |
| **8** | **57%** |

At 8 players, more than half of matches have someone degrading the experience for everyone. There is no isolation — a lockstep session is only as good as its worst member.

**And the CPU cost is multiplicative.** Rollback cost per frame is `rollback_frames × simulation_cost`, on top of the current frame:

```
60 Hz budget:         16.67 ms
Allow rollback to use: ~4 ms (25%)
With 8 frames of rollback: simulation must cost ≤ 0.5 ms
```

That's a hard ceiling on simulation cost, and it tightens as rollback depth grows. A simulation that takes 2 ms per frame — entirely reasonable — blows the budget at 8 frames of rollback.

**Plus the topology.** Full-mesh P2P at 8 players is 28 connections, each needing NAT traversal. At 2 players it's one.

The conclusion is the one the field already reached: rollback is for small player counts with tightly-coupled mechanics.

### 2.4 The failure modes

| Symptom | Cause |
|---|---|
| **Permanent desync mid-match** | One non-deterministic operation, anywhere (§3) |
| Desync only on some hardware | Float transcendentals or x87/SSE differences (§3.1) |
| Desync only in release builds | `-ffast-math` or FMA contraction (§3.1) |
| Desync 1 match in 1000 | Uninitialized memory, or hash-map iteration order (§3.3) |
| Frame rate tanks under load | Rollback depth × sim cost over budget (§2.3) |
| Hit sound plays 8 times | Audio triggered during rollback frames (§6.2) |
| Remote characters teleport constantly | Analog input defeating carry-forward prediction (§2.2) |
| Everyone lags when one player does | Inherent to lockstep — no isolation (§2.3) |
| Memory grows over a match | State snapshots not bounded (§4.4) |
| Connection fails for some players | NAT traversal without a relay fallback (§7.3) |

---

## 3. Determinism ★

### 3.1 Floating point is the enemy

A single divergence desyncs everyone, permanently, and compounds. The main sources:

| Cause | Mechanism |
|---|---|
| **Transcendentals** | `sin`, `cos`, `pow`, `exp`, `log` are not correctly rounded and differ between libm implementations, versions, and platforms |
| **FMA contraction** | The compiler may fuse `a*b+c` into a single fused multiply-add with different intermediate rounding |
| **`-ffast-math`** | Permits reassociation, which changes results |
| **x87 vs SSE** | 80-bit intermediates versus 64-bit, on the same source |
| **Auto-vectorization** | Changes the order of reductions, which changes float sums |
| **Different compilers** | Different optimization decisions, different results |

Basic IEEE-754 operations — `+ − × ÷ sqrt` — *are* deterministic if you control the rounding mode and prevent contraction. Everything else is not.

### 3.2 Use fixed-point ★

The practical answer, and what current deterministic netcode libraries ship: **32.32 fixed-point** (a 64-bit integer with 32 fractional bits) plus a seeded PRNG, giving bit-identical results across platforms.

```cpp
struct Fixed64 {
    int64_t raw;                                  // 32.32

    static constexpr int SHIFT = 32;
    static constexpr int64_t ONE = int64_t(1) << SHIFT;

    static Fixed64 FromInt(int32_t v)  { return { int64_t(v) << SHIFT }; }
    int32_t ToInt() const              { return int32_t(raw >> SHIFT); }

    Fixed64 operator+(Fixed64 o) const { return { raw + o.raw }; }
    Fixed64 operator-(Fixed64 o) const { return { raw - o.raw }; }

    Fixed64 operator*(Fixed64 o) const {
        // Needs a 128-bit intermediate or the product overflows.
        return { int64_t((__int128(raw) * __int128(o.raw)) >> SHIFT) };
    }

    Fixed64 operator/(Fixed64 o) const {
        return { int64_t((__int128(raw) << SHIFT) / __int128(o.raw)) };
    }
};
```

Range and precision at 32.32: roughly ±2.1 billion with a resolution of about 2.3 × 10⁻¹⁰. Ample for game-world coordinates.

**The 128-bit intermediate in multiply is not optional.** `raw * o.raw` overflows 64 bits for almost any meaningful values, and the resulting wraparound is a silent, spectacular desync. On MSVC use `_mul128`; on compilers without `__int128`, split into 32-bit halves.

**Transcendentals become tables.** Sine, cosine, and square root are lookup tables with fixed-point interpolation, or fixed-point Newton iteration. They're identical everywhere by construction because they're integer arithmetic.

```cpp
// 4096-entry quarter-wave table, linearly interpolated. Deterministic
// because every operation is integer.
Fixed64 FixedSin(Fixed64 radians);
Fixed64 FixedSqrt(Fixed64 x);      // integer Newton-Raphson
```

**The alternative — disciplined floats** — means a single compiler, a single architecture, `-ffp-contract=off`, no `-ffast-math`, no auto-vectorization in the sim, and your own transcendentals anyway. That's most of the fixed-point discipline without the cross-platform guarantee. Fixed-point is the better trade.

### 3.3 The non-float determinism killers

Easy to overlook and just as fatal:

| Killer | Fix |
|---|---|
| **Hash-map iteration order** | Never iterate an unordered container in the sim. Arrays or sorted keys. |
| **Pointer values in sorts or hashes** | Use stable integer IDs, never addresses |
| **Uninitialized memory** | Zero the state arena. A padding byte in a checksum is a phantom desync. |
| **`std::sort` instability** | Not specified to produce identical output for equal elements. Use a stable sort with a total order. |
| **Thread scheduling** | Single-threaded simulation, or a deterministic parallel scheme |
| **Wall-clock time** | Use the frame counter only |
| **Unsynced RNG** | One seeded PRNG in the state arena, advanced only by the sim |
| **Allocation addresses** | No heap allocation during simulation at all (§4.2) |
| **Different library versions** | Vendor anything the sim touches |

The uninitialized-padding one deserves emphasis. If you `memcpy` a state struct that has padding bytes, and you checksum the whole thing, two peers with identical logical state can produce different checksums. Either zero the arena on init and never leave gaps, or checksum fields explicitly rather than raw bytes.

### 3.4 The determinism contract

Write it down, because the game team will need to follow it:

```
Simulation code MUST:
  - use Fixed64 for all continuous quantities
  - use the sim's seeded PRNG for all randomness
  - operate only on state within the arena
  - iterate only over arrays or explicitly-ordered collections
  - be single-threaded (or use the approved parallel scheme)

Simulation code MUST NOT:
  - use float or double, anywhere
  - allocate
  - read the clock, the file system, or the network
  - call rand(), std::random_device, or any unseeded source
  - depend on pointer values
  - trigger audio, VFX, logging, or any side effect (§6)
```

**Enforce it mechanically.** A lint pass or a static-analysis rule that rejects `float` and `double` in the simulation directory catches the accidental violation before it becomes a 1-in-1000 desync.

---

## 4. State save and restore ★

### 4.1 The requirement

Rollback means: restore the state from frame N, then re-simulate frames N+1 through the present, every time a remote input arrives that contradicts a prediction. At 60 Hz with 8 frames of rollback, that's a restore plus 8 simulation steps, potentially every frame.

Saving must therefore be **cheap enough to do every frame** and restoring cheap enough to do many times per second.

### 4.2 One contiguous POD arena ★

The only approach that is reliably fast enough: put the entire simulation state in a single contiguous block of plain old data, and save it with one `memcpy`.

```cpp
// Everything the simulation reads or writes lives here. Nothing else.
struct GameState {
    uint32_t  frame;
    uint64_t  rngState;

    Player    players[MAX_PLAYERS];      // fixed-size, never a vector
    Projectile projectiles[MAX_PROJECTILES];
    uint32_t  projectileCount;

    // Free-list allocation over fixed arrays — indices, never pointers.
    int16_t   freeProjectiles[MAX_PROJECTILES];
    uint16_t  freeProjectileCount;
};
static_assert(std::is_trivially_copyable_v<GameState>);
```

The consequences reach into every gameplay system:

- **No pointers.** Use indices into the arrays. A pointer is meaningless after a restore.
- **No `std::vector`, `std::string`, or anything that heap-allocates.** Fixed-capacity arrays with explicit counts.
- **No dynamic allocation during simulation, ever.** Free lists over pre-sized pools.
- **No virtual functions in state objects.** A vtable pointer is an address.
- **Fixed maximum counts for everything**, chosen up front. Running out of projectiles is a design constraint, not a crash.

This is a significant discipline, and it is not optional. A rollback layer bolted onto a simulation with heap-allocated object graphs will not work, and discovering that in month four is expensive.

### 4.3 Save cost

```
State size × 60 saves/sec = memcpy bandwidth
```

| State size | Bandwidth at 60 Hz |
|---|---|
| 64 KB | 3.8 MB/s — negligible |
| 512 KB | 30 MB/s — fine |
| 4 MB | 240 MB/s — noticeable |
| 32 MB | 1.9 GB/s — a problem |

**Keep state small.** It's usually achievable: player state, projectiles, and a few counters. Everything visual — meshes, textures, particle systems, audio — lives outside the arena and is derived from it.

If state does get large, incremental or delta snapshots are possible but substantially more complex, and they interact badly with the simplicity that makes rollback debuggable. Shrink the state first.

### 4.4 Bounding the ring

Keep a fixed ring of saved frames, sized to the maximum rollback depth:

```cpp
class StateRing {
    static constexpr int CAPACITY = MAX_ROLLBACK_FRAMES + 2;
    GameState frames[CAPACITY];
    uint32_t  frameNumbers[CAPACITY];
public:
    void Save(const GameState& s, uint32_t frame) {
        const int idx = frame % CAPACITY;
        std::memcpy(&frames[idx], &s, sizeof(GameState));
        frameNumbers[idx] = frame;
    }
    bool Restore(GameState& out, uint32_t frame) const {
        const int idx = frame % CAPACITY;
        if (frameNumbers[idx] != frame) return false;   // rolled off the ring
        std::memcpy(&out, &frames[idx], sizeof(GameState));
        return true;
    }
};
```

Fixed capacity is the point. An unbounded history grows until a long match runs out of memory, and the `frameNumbers` check catches the case where you're asked to roll back further than you kept — which should stall rather than silently restore the wrong frame.

---

## 5. The rollback loop

### 5.1 The core algorithm

```
each frame:
    collect local input for frame F
    send local input (with redundancy — §7.1) to all peers
    receive any remote inputs that arrived

    if a received input for frame P < F differs from what we predicted:
        restore state from frame P
        for f in P..F:
            simulate(f) using real inputs where known, predictions otherwise
            save state at f
    else:
        predict remote inputs for frame F
        simulate(F)
        save state at F

    render interpolated from the latest simulated state
```

```cpp
void Session::AdvanceFrame(const Input& localInput) {
    inputQueue.AddLocal(currentFrame, localInput);
    transport.SendInputs(inputQueue.RecentLocal(REDUNDANCY_FRAMES));
    transport.ReceiveInputs(inputQueue);

    const int32_t firstIncorrect = inputQueue.FirstIncorrectPredictionFrame();
    if (firstIncorrect >= 0) {
        Rollback(firstIncorrect);
    }

    Simulate(currentFrame);
    stateRing.Save(state, currentFrame);
    ++currentFrame;
}

void Session::Rollback(uint32_t toFrame) {
    if (!stateRing.Restore(state, toFrame)) {
        // Asked to go further back than we kept. Stall rather than desync.
        OnRollbackWindowExceeded(toFrame);
        return;
    }
    const uint32_t target = currentFrame;
    for (uint32_t f = toFrame; f < target; ++f) {
        Simulate(f);                 // uses corrected inputs now
        stateRing.Save(state, f);
    }
    stats.rollbacks++;
    stats.rolledBackFrames += (target - toFrame);
}
```

### 5.2 Carry-forward prediction

```cpp
Input InputQueue::Predict(PlayerId p, uint32_t frame) const {
    // Assume the remote player keeps doing what we last heard.
    // Correct most frames for digital input; poor for analog (§2.2).
    return lastKnownInput[p];
}
```

Simple, and effective for quantized inputs. Two refinements worth considering:

- **Decay held inputs after N frames.** A player who appears to hold forward for 30 predicted frames has probably stopped; continuing to predict forward makes the eventual rollback large.
- **Predict buttons as released, directions as held.** Buttons are usually taps; directions are usually holds. Asymmetric prediction reduces average rollback depth measurably.

### 5.3 Input delay is the other half of the dial ★

Rollback and input delay trade against each other, and the tradeoff is the main tuning knob:

| Input delay | Rollback depth needed | Local feel |
|---|---|---|
| 0 frames | Full latency | Instant, maximum rollback artifacts |
| 2 frames | Latency − 2 | Slightly delayed, fewer artifacts |
| 4 frames | Latency − 4 | Noticeably delayed, smooth |

Every frame of input delay removes a frame of rollback. GGPO-style games typically run 2–4 frames.

**Adaptive delay** measures RTT and adjusts:

```cpp
int RecommendedInputDelay(float rttMs, float jitterMs) {
    const float oneWayFrames = (rttMs * 0.5f + jitterMs) / (1000.0f / 60.0f);
    return std::clamp(int(std::floor(oneWayFrames * 0.5f)), 0, MAX_INPUT_DELAY);
}
```

Covering roughly half the latency with delay and the rest with rollback is a reasonable default. **Change it only between matches or at clearly-marked moments** — adjusting input delay mid-match changes the feel under the player's hands and is more disruptive than the rollback it prevents.

### 5.4 When the window is exceeded

If a peer falls further behind than your ring holds, you cannot roll back far enough. The options:

- **Stall.** Stop advancing until their inputs arrive. Everyone freezes briefly. Ugly but correct, and it's what lockstep does by nature.
- **Disconnect them.** After a threshold.

Do not silently continue — that's a desync. Surface stalls in the stats (§8.4); a session with frequent stalls has a peer who should be disconnected.

---

## 6. Rollback-safe simulation ★

### 6.1 The simulation runs multiple times

Frame 100 may be simulated once speculatively and then re-simulated four more times as corrections arrive. **Anything with a side effect outside the state arena will happen five times.**

### 6.2 Audio and VFX during rollback

The classic bug: a hit connects on a predicted frame, the hit sound plays, a rollback occurs, the hit connects again during re-simulation, the sound plays again. The player hears the same impact five times in a row.

The fix is a **confirmed-frame** concept. The simulation never triggers presentation directly; it records events into the state, and presentation consumes them only once the frame is confirmed:

```cpp
// The simulation records — it never triggers.
void Simulate(uint32_t frame) {
    if (HitDetected(a, b)) {
        state.events[state.eventCount++] = { EventType::Hit, frame, a, b };
    }
}

// Presentation runs only for confirmed frames, exactly once each.
void PresentConfirmedFrames(uint32_t confirmedThrough) {
    for (uint32_t f = lastPresentedFrame + 1; f <= confirmedThrough; ++f) {
        for (const auto& ev : EventsForFrame(f)) {
            audio.Play(ev);
            vfx.Spawn(ev);
        }
    }
    lastPresentedFrame = confirmedThrough;
}
```

Events live in the state arena, so they're rolled back with everything else — a hit that didn't really happen disappears when its frame is corrected.

The cost is that confirmed audio lags the visual by the confirmation delay. That's usually acceptable and is what shipped rollback games do. The alternative — playing on prediction and cancelling on rollback — sounds worse, because a truncated sound is more noticeable than a slightly late one.

### 6.3 The broader rule

During simulation, forbid: audio, VFX spawning, network sends, file I/O, logging, allocation, and any mutation outside the arena.

**A debug build should assert on all of these.** An allocation hook that fires an assertion when called from inside `Simulate` catches the violation immediately rather than as a desync three weeks later.

---

## 7. Transport and topology

### 7.1 Input redundancy replaces retransmission ★

Inputs are tiny and latency-critical, so the right protocol is unreliable UDP with **every packet carrying the last N frames of input**. A dropped packet is covered by the next one, with no retransmission round-trip.

```cpp
struct InputPacket {
    uint32_t startFrame;
    uint8_t  frameCount;          // typically 8-16
    Input    inputs[MAX_REDUNDANCY];
    uint32_t ackFrame;            // highest frame we've received from them
    uint32_t checksum;            // periodic — §8.1
};
```

The bandwidth is trivial. At 2 bytes per frame of input, 16 frames of redundancy is 32 bytes plus header, sent 60 times a second — roughly **3 KB/s per peer direction**. Even a full 8-player mesh is about 21 KB/s up per player.

This is why lockstep's bandwidth story is so good, and it's the one dimension where 8 players is not a problem.

### 7.2 Topology

| | **Full mesh P2P** | **Relay / star** |
|---|---|---|
| Connections at 8 players | 28 | 8 |
| Latency | Direct — lowest | One extra hop |
| NAT traversal | Hard, and 28× harder | Trivial |
| Server cost | None | Small (forwarding only) |
| Still rollback? | Yes | **Yes** — the relay is not authoritative |

The relay option is underappreciated. **A relay that only forwards input packets is still deterministic lockstep** — it holds no authority and makes no decisions. You pay one hop of latency and get NAT traversal, connection reliability, and a natural place to detect and log desyncs.

At 2 players, direct P2P with a relay fallback. At 4+, consider relay-first.

### 7.3 NAT traversal

Consistently underestimated. Direct P2P needs STUN for address discovery, hole punching, and a TURN relay fallback for the cases that fail — which is a meaningful fraction of real-world connections, particularly symmetric NATs and carrier-grade NAT on mobile networks.

Budget real time for this, and **build the relay fallback first**. A working relayed connection that you later optimize into a direct one is a much better position than a direct connection that fails for 20% of players.

### 7.4 Clock synchronization and frame pacing

Peers must run at approximately the same rate or one drifts ahead and the other stalls constantly.

The standard technique: exchange frame advantage — "I am N frames ahead of the inputs I have from you" — and have the peer that's ahead occasionally skip a frame or run slightly slow.

```cpp
void Session::AdjustPacing() {
    const int localAdvantage = currentFrame - lastConfirmedFrame;
    const int remoteAdvantage = peer.reportedFrameAdvantage;
    const int diff = localAdvantage - remoteAdvantage;

    // If meaningfully ahead, wait a frame so they can catch up.
    if (diff > FRAME_ADVANTAGE_THRESHOLD) {
        framesToStall = std::min(diff / 2, MAX_STALL_FRAMES);
    }
}
```

Halving the difference rather than correcting it fully avoids oscillation, where both peers overcorrect and alternate stalling.

---

## 8. Desync detection and diagnosis ★

### 8.1 Detection is easy

Exchange a state checksum periodically — every frame during development, every 30 or 60 frames in release — and compare for frames both peers have confirmed.

```cpp
uint32_t ChecksumState(const GameState& s) {
    // Checksum FIELDS, not raw bytes — struct padding is uninitialized
    // and produces phantom desyncs (§3.3).
    uint32_t h = 2166136261u;
    for (const auto& p : s.players) {
        h = Hash(h, p.posX.raw); h = Hash(h, p.posY.raw);
        h = Hash(h, p.velX.raw); h = Hash(h, p.velY.raw);
        h = Hash(h, p.stateId);  h = Hash(h, p.health);
    }
    for (uint32_t i = 0; i < s.projectileCount; ++i) { /* ... */ }
    h = Hash(h, s.rngState);
    return h;
}
```

### 8.2 Diagnosis is the hard part ★

A checksum tells you *that* you desynced. It tells you nothing about *why*, and by the time you notice, the states have diverged arbitrarily far.

**Hierarchical checksums narrow it down.** Instead of one number, send a small vector:

```cpp
struct StateChecksums {
    uint32_t total;
    uint32_t players;
    uint32_t projectiles;
    uint32_t rng;
    uint32_t physics;
};
```

Now a mismatch says "the RNG diverged" or "player state diverged but projectiles didn't," which is the difference between an afternoon and a week.

**Full state dumps plus binary search** is the definitive technique. On detecting a mismatch at frame F, both peers dump their full state history for the last N frames. Offline, binary search for the first differing frame, then diff the two states field by field. The first field that differs, on the first frame it differs, is almost always directly adjacent to the bug.

```
tools/desync_diff.py peer_a_dump.bin peer_b_dump.bin
  → first divergence at frame 8412
  → field: players[1].velX   A: 0x0000_1A3F   B: 0x0000_1A40
  → one LSB in a velocity — check the fixed-point division at Player.cpp:214
```

Build this tooling early. Desyncs are inevitable, and the difference between a project that ships and one that doesn't is often how fast you can diagnose them.

### 8.3 The 1-in-1000 desync

The worst kind. It doesn't reproduce, so you can't debug it interactively.

Defenses:

- **Automatic state dump and upload on desync**, from release builds, with consent. You need the data from the wild because you cannot reproduce it locally.
- **A desync rate metric** tracked across all sessions. A regression shows up as the rate moving, long before anyone reports it.
- **A soak test** running thousands of headless matches with randomized inputs overnight, checksumming every frame (§15.2). This finds the rare path that manual testing never will.

### 8.4 Session statistics

Expose these; they're how you tell a netcode problem from a game problem:

```cpp
struct SessionStats {
    float    avgRollbackFrames;
    uint32_t maxRollbackFrames;
    float    rollbacksPerSecond;
    uint32_t stallFrames;              // §5.4
    float    rttMs[MAX_PLAYERS];
    float    jitterMs[MAX_PLAYERS];
    float    packetLossPct[MAX_PLAYERS];
    uint32_t desyncCount;
    int      currentInputDelay;
};
```

Show them in a debug overlay, and consider a simplified version in the player-facing UI. A player who can see that one opponent has 180 ms of jitter understands why the match feels bad, and that's better than assuming the game is broken.

---

## 9. The server-authoritative alternative

If 5–8 players is a hard requirement, this is what you build instead — and it's a different project, not a mode of the same one.

### 9.1 The model

```
Client                                    Server
  │ input (with a sequence number) ───────▶│
  │                                        │ simulate authoritatively
  │◀────── state snapshot + last-acked seq │
  │
  │ apply the correction at the acked seq,
  │ replay unacked local inputs forward
```

- **Client-side prediction**: run your own movement locally without waiting for the server.
- **Server reconciliation**: when a correction arrives for input sequence N, rewind to it and replay inputs N+1..current.
- **Entity interpolation**: render *other* players slightly in the past, between two received snapshots, so their motion is smooth.
- **Lag compensation**: the server keeps ~1 s of position history and rewinds other players to the shooter's view time for hit detection.

### 9.2 Why it scales where lockstep doesn't

- **No determinism requirement at all.** Floats are fine. This alone removes most of section 3.
- **Bad peers are isolated.** One player's poor connection affects only them.
- **Cheat resistance.** Clients only know what the server tells them.
- **Bandwidth grows with state, not players squared.** Delta compression and relevancy filtering keep it manageable.

### 9.3 What it costs

- **Servers.** Real infrastructure, real money, real operations.
- **The "shot behind cover" problem.** Lag compensation means you can be hit after reaching cover on your own screen, because the server rewound to when the shooter saw you. It is a genuine, unavoidable tradeoff, not a bug — favouring the shooter necessarily disfavours the target.
- **Snapshot and delta encoding**, relevancy, and interpolation are each substantial subsystems.

### 9.4 Don't build a hybrid

A layer that supports both architectures well is two libraries sharing a name. The state models, authority models, prediction mechanisms, and correction mechanisms are all different.

**Pick one. Document why.** The decision record in appendix A is the artifact.

---

## 10. Performance

### 10.1 The budget

```
16.67 ms per frame at 60 Hz
  render                  ~8 ms
  current frame sim        X ms
  rollback re-simulation   rollback_frames × X ms
  state save               state_size / memcpy_bandwidth
  network                  <0.1 ms
```

The rollback term dominates and is the one you control least — it depends on the worst peer's latency.

```
sim_cost ≤ rollback_budget / max_rollback_frames
```

With a 4 ms rollback budget and 8 frames, simulation must cost 0.5 ms. Design to that number from the beginning.

### 10.2 Making the simulation cheap

- **Small state** (§4.3) — helps save cost and cache behavior together
- **Structure of arrays** for anything iterated in bulk
- **Fixed-point is fast.** 64-bit integer multiply-shift is comparable to float multiply, and the 128-bit intermediate costs little on x86-64.
- **No allocation** — already required for determinism
- **Avoid branchy per-entity logic** in the hot path

### 10.3 Optimizations specific to rollback

**Skip redundant re-simulation.** If a corrected input is identical to what you predicted, you don't need to roll back at all — that's the `FirstIncorrectPredictionFrame` check in §5.1, and it means most frames cost nothing extra.

**Save less often than every frame.** Saving every 2 frames halves the save cost at the price of one extra simulation step on restore. Worth it when state is large and simulation is cheap.

**Cap rollback depth.** Beyond a threshold, stall instead (§5.4). An unbounded rollback depth means an unbounded frame time, which is worse than a visible hitch.

### 10.4 Measure the right things

Wall-clock frame time hides the structure. Track separately: current-frame simulation cost, rollback simulation cost, save cost, and restore cost — and track rollback depth as a distribution, not a mean. The p99 depth is what determines whether you hitch.

---

## 11. Tech stack and setup

| Layer | Choice | Why |
|---|---|---|
| **Language** | **C++17** or **Rust** | Control over memory layout and arithmetic |
| **Fixed-point** | Own 32.32 implementation | Small, and you need to control every operation |
| **Transport** | Raw UDP, or `ENet`/`laminar` for the plumbing | Unreliable is correct here |
| **NAT traversal** | STUN + your own hole punching, or a managed service | §7.3 |
| **Relay** | A small forwarding server | §7.2 |
| **Serialization** | Direct `memcpy` of POD | The state arena is already the format |
| **Testing** | Headless deterministic soak harness | §15.2 |

Three setup notes:

**Build the layer engine-agnostic.** It takes inputs, calls a `Simulate` callback, manages state save/restore through callbacks, and handles the network. That makes it testable headlessly and reusable, and it keeps the determinism contract explicit at the boundary.

**Get the soak test running before anything else works.** Thousands of headless matches with randomized inputs, checksumming every frame, is the only thing that finds rare desyncs. It should exist in week two.

**Study `ggrs` or GGPO's source.** GGPO is the reference implementation of this architecture and reading it will save weeks. Projects that build on a mature rollback crate rather than reimplementing report it as the cheap path — and they still only ship 2-player.

---

## 12. Repository layout

```
Rollback-Netcode-Layer/
├── README.md
├── docs/
│   ├── design.md                ← this document
│   ├── determinism-contract.md  ← ★ the §3.4 rules, for game teams
│   ├── architecture-decision.md ← ★ why lockstep, why not lag compensation
│   └── desync-playbook.md       ← how to diagnose one
├── fixed/                       ← ★ standalone, no dependencies
│   ├── fixed64.h
│   ├── trig.h                   ← table-based, deterministic
│   └── tests/
├── core/
│   ├── session.h/.cpp           ← the rollback loop (§5)
│   ├── input_queue.h/.cpp       ← prediction, confirmation
│   ├── state_ring.h             ← ★ bounded snapshots (§4.4)
│   ├── sync.h/.cpp              ← frame pacing (§7.4)
│   └── checksum.h               ← ★ hierarchical (§8.2)
├── transport/
│   ├── udp.cpp
│   ├── redundancy.cpp           ← ★ (§7.1)
│   ├── nat.cpp                  ← STUN, hole punching
│   └── relay_client.cpp
├── relay/                       ← the forwarding server
├── tools/
│   ├── desync_diff.py           ← ★ state dump diffing (§8.2)
│   └── soak.py                  ← ★ headless match runner (§15.2)
└── sample/
    └── minimal_game/            ← a tiny deterministic sim for testing
```

`fixed/` having no dependencies and its own tests is what makes it verifiable in isolation — and the fixed-point layer is where a subtle bug does the most damage.

---

## 13. Milestone ladder

### M0 — The architecture decision ★ **before any code**
**Est. 2–3 days**

Write `docs/architecture-decision.md`. Choose lockstep or server-authoritative. Document why lag compensation is or isn't part of it. Set the player-count target and justify it against §2.3.

**Done when:** someone reading it understands why the project doesn't do the thing the original statement said.

---

### M1 — Fixed-point math ★
**Est. 1 week**

32.32 with 128-bit intermediates, vectors, trig tables, sqrt, a seeded PRNG. Standalone, fully tested, cross-platform verified.

**Verify bit-exactness across platforms in CI** — Windows, Linux, macOS, x86-64 and ARM. This is the foundation and a divergence here poisons everything.

---

### M2 — Deterministic sample simulation
**Est. 1 week**

A minimal game — a few players, movement, collision, projectiles — in a contiguous POD arena, obeying the determinism contract.

**Done when:** running the same inputs twice produces identical checksums, and running on two different platforms does too.

---

### M3 — Save, restore, and local rollback ★
**Est. 1 week**

The state ring, save/restore, and a local rollback test: simulate 100 frames, roll back 30, re-simulate, assert the state matches the original run exactly.

**No networking yet.** Rollback correctness is much easier to establish locally, and every bug you find here is one you don't have to find over a network.

---

### M4 — The rollback loop
**Est. 1.5 weeks**

Input queue with prediction and confirmation, the advance-frame algorithm, input delay.

**Done when:** two local sessions with simulated latency stay in sync indefinitely.

---

### M5 — Transport
**Est. 2 weeks**

UDP, input redundancy, ack handling, frame pacing, connection lifecycle. Test under simulated loss, latency, jitter, and reordering.

---

### M6 — NAT and relay
**Est. 2 weeks**

Relay server first, then STUN and hole punching with relay fallback.

**Relay first.** A working relayed connection beats a direct one that fails for a fifth of players.

---

### M7 — Desync tooling ★
**Est. 1.5 weeks**

Hierarchical checksums, automatic state dumps, the diffing tool, the soak harness, telemetry.

**This milestone determines whether the project ships.** Desyncs are inevitable; diagnosis speed is the variable.

---

### M8 — Rollback-safe presentation
**Est. 1 week**

Confirmed-frame events, audio and VFX gating, debug assertions on side effects during simulation.

---

### M9 — Tuning and stats
**Est. 1 week**

Adaptive input delay, prediction refinements, session stats and overlay, stall handling.

---

### M10 — Scale testing
**Est. 1 week**

4-player sessions under realistic network conditions. Measure rollback depth distributions. Decide honestly whether the player-count target holds.

---

## 14. Reference implementations

### 14.1 The input queue

```cpp
class InputQueue {
    static constexpr int QUEUE_SIZE = 128;

    Input    inputs[MAX_PLAYERS][QUEUE_SIZE];
    bool     confirmed[MAX_PLAYERS][QUEUE_SIZE];
    Input    predicted[MAX_PLAYERS][QUEUE_SIZE];
    uint32_t firstIncorrect = UINT32_MAX;

public:
    void AddRemote(PlayerId p, uint32_t frame, const Input& in) {
        const int idx = frame % QUEUE_SIZE;
        if (confirmed[p][idx]) return;              // duplicate from redundancy

        // If we predicted this frame and were wrong, remember the earliest.
        if (frame < currentFrame && predicted[p][idx] != in) {
            firstIncorrect = std::min(firstIncorrect, frame);
        }
        inputs[p][idx] = in;
        confirmed[p][idx] = true;
    }

    Input Get(PlayerId p, uint32_t frame) {
        const int idx = frame % QUEUE_SIZE;
        if (confirmed[p][idx]) return inputs[p][idx];
        const Input guess = lastKnown[p];           // carry-forward (§5.2)
        predicted[p][idx] = guess;                  // record it so we can compare
        return guess;
    }
};
```

Recording the prediction is what makes `firstIncorrect` possible. Without it you'd have to roll back on every received input rather than only on the ones that contradict you — and most predictions are correct.

### 14.2 A determinism assertion for debug builds

```cpp
#if ROLLBACK_DEBUG
struct SimulationGuard {
    SimulationGuard()  { g_inSimulation = true;  }
    ~SimulationGuard() { g_inSimulation = false; }
};

void* operator new(size_t) {
    assert(!g_inSimulation && "allocation during simulation breaks rollback");
    // ...
}

void AudioSystem::Play(SoundId) {
    assert(!g_inSimulation && "audio during simulation plays N times on rollback");
    // ...
}
#endif
```

These assertions catch §3.4 and §6.3 violations at the moment they're introduced rather than as a desync or a stuttering sound effect weeks later. They cost nothing in release.

---

## 15. Testing

### 15.1 Fixed-point, exhaustively

```cpp
TEST_CASE("fixed multiply never overflows for in-range values") {
    for (int64_t a : InterestingValues())
      for (int64_t b : InterestingValues()) {
        const Fixed64 r = Fixed64{a} * Fixed64{b};
        REQUIRE(r.raw == ReferenceMultiply(a, b));    // 128-bit reference
      }
}

TEST_CASE("trig tables are identical across platforms") {
    uint32_t h = 0;
    for (int i = 0; i < 65536; ++i) h = Hash(h, FixedSin(Fixed64::FromRaw(i)).raw);
    REQUIRE(h == 0xDEADBEEF);   // a checked-in constant, verified on every platform
}
```

The checked-in hash constant is the mechanism that catches a platform difference in CI rather than in a player's match.

### 15.2 The soak harness ★

The single most valuable test in the project.

```python
# tools/soak.py — run thousands of matches overnight
def run_match(seed):
    a = HeadlessSession(seed, player=0)
    b = HeadlessSession(seed, player=1)
    net = LossyLink(latency_ms=random(20, 200),
                    jitter_ms=random(0, 50),
                    loss_pct=random(0, 5))

    for frame in range(MATCH_FRAMES):
        a.advance(random_input(seed, 0, frame))
        b.advance(random_input(seed, 1, frame))
        net.pump(a, b)

        if a.confirmed_frame == b.confirmed_frame:
            if a.checksum() != b.checksum():
                dump_and_report(seed, frame, a, b)
                return FAIL
    return PASS
```

Randomize latency, jitter, loss, and inputs. Run it continuously in CI and nightly at volume. **A desync found by the soak test costs an hour; the same desync found by a player costs a week** — because you have the seed and can reproduce it exactly.

### 15.3 Rollback equivalence

```cpp
TEST_CASE("rollback produces identical state to a clean run") {
    auto clean = RunFrames(inputs, 0, 100);

    auto rolled = RunFrames(inputs, 0, 100);
    rolled.RollbackTo(70);
    rolled.ResimulateTo(100);

    REQUIRE(Checksum(clean.state) == Checksum(rolled.state));
}
```

Do this for every rollback depth from 1 to the maximum. A depth-specific bug — an off-by-one in the ring, say — hides until exactly that depth occurs in a real match.

### 15.4 Network condition matrix

Test across the grid: latency 20–300 ms, jitter 0–100 ms, loss 0–10%, plus reordering and duplication. Assert: no desync, rollback depth within expectation, and stalls only above the configured threshold.

### 15.5 Cross-platform determinism

Run the identical input sequence on every target platform in CI and compare final checksums. This is the test that justifies the fixed-point investment, and it should fail loudly the day someone adds a `float`.

---

## 16. Stretch goals

| Feature | Effort | Value |
|---|---|---|
| **Spectators** | Small | A read-only peer receiving inputs. Nearly free given the architecture. |
| **Replays** | Small | Inputs plus a seed *is* the replay. Verify with checksums. |
| **Late join** | Large | Full state transfer plus a resync protocol |
| **Reconnect** | Medium | Same machinery as late join |
| **Rollback-aware animation blending** | Medium | Hide correction pops on remote characters |
| **Better prediction** | Medium | Input-pattern models beat carry-forward, measurably |
| **Cross-play determinism** | Large | ARM and x86 bit-identical — fixed-point already gets you most of the way |
| **Anti-cheat for lockstep** | Large | Input-timing analysis; fundamentally limited (§9.2) |
| **Server-authoritative mode** | Very large | §9 — a separate library |

Replays deserve a second look: in a deterministic input-only architecture, a replay is the input log plus a seed, which you already have. It's nearly free, it's a genuinely popular feature, and it doubles as a bug-report format — "here's the replay" reproduces the exact match.

---

## 17. References

### Rollback and lockstep

- **Tony Cannon**, "Fight The Lag! The Trick Behind GGPO's Low Latency Netcode" — the original explanation
- **GGPO** source, and **ggrs** (Rust) — the reference implementations; read one before writing your own
- **Ricky Pusch**, "Explaining How Fighting Games Use Delay-Based and Rollback Netcode" — the clearest public explanation of the tradeoff
- **Glenn Fiedler** (Gaffer On Games) — deterministic lockstep, reliable UDP, and the networked-physics series
- Modern deterministic netcode libraries shipping 32.32 fixed-point plus seeded PRNGs — a current reference for the §3.2 approach

### Server-authoritative

- **Valve Developer Wiki**, "Source Multiplayer Networking" — the canonical lag-compensation description used in §2.1
- **Gabriel Gambetta**, "Fast-Paced Multiplayer" — prediction, reconciliation, interpolation, compensation, explained with interactive demos
- **Yahn Bernier**, "Latency Compensating Methods in Client/Server In-game Protocol Design" (GDC 2001)

### Determinism

- "Floating Point Determinism" (Fiedler) — why cross-platform float determinism is not achievable in general
- Goldberg, "What Every Computer Scientist Should Know About Floating-Point Arithmetic"
- Compiler documentation on `-ffp-contract`, `-ffast-math`, and strict FP semantics

### Choosing a model

- **mas-bandwidth**, "Choosing the right network model for your multiplayer game" — direct guidance that tightly-coupled melee and fighting games want GGPO-style lockstep, and shooters want client-server

---

## Appendix A — Decision record

| Decision | Rationale |
|---|---|
| **Rollback lockstep, not a hybrid** | Lockstep and server-authoritative are different authority models with different state, prediction, and correction mechanisms. Supporting both well is two libraries. |
| **Lag compensation is explicitly out of scope** | It's a server-authoritative technique — the server rewinds *other players'* hitboxes to the shooter's view time. There's no server in lockstep and nothing to rewind, because all peers already agree. |
| **Player count scoped to 2–4, not 8** | Rollback depth is set by the worst peer, and P(at least one bad link) rises from 19% at 2 players to 57% at 8. Plus 28 P2P connections and a multiplicative CPU cost. |
| Document the 8-player alternative rather than building it | §9 is a different project; saying so is more useful than a mode that does it badly |
| **32.32 fixed-point, no floats anywhere in the sim** | Transcendentals aren't correctly rounded, FMA contracts, `-ffast-math` reassociates, x87 differs from SSE. One divergence desyncs permanently. |
| 128-bit intermediate in fixed multiply | `raw * raw` overflows 64 bits for almost any real values — a silent, spectacular desync |
| Trig and sqrt as integer tables and Newton iteration | Identical everywhere by construction |
| **Lint-enforced ban on `float`/`double` in the sim directory** | Catches the accidental violation before it becomes a 1-in-1000 desync |
| Checksum fields, never raw bytes | Struct padding is uninitialized and produces phantom desyncs |
| **All state in one contiguous POD arena** | Rollback needs cheap save/restore every frame; one `memcpy` is the only reliably fast approach |
| No pointers, no vectors, no allocation in the sim | A pointer is meaningless after restore, and allocation addresses are nondeterministic |
| Fixed maximum counts for everything | Running out is a design constraint, not a crash |
| Bounded state ring with a frame-number check | Unbounded history exhausts memory; the check catches asking for a frame that rolled off |
| **Carry-forward input prediction** | Correct most frames for digital input. Analog input defeats it — a reason not to choose lockstep at all for mouse-look games. |
| Asymmetric prediction: buttons released, directions held | Reduces average rollback depth for free |
| **Input delay as the other half of the dial, adjusted between matches only** | Each frame of delay removes a frame of rollback; changing it mid-match is more disruptive than the rollback it prevents |
| Stall rather than desync when the window is exceeded | Silently continuing is the one unacceptable outcome |
| **Confirmed-frame events for audio and VFX** | Otherwise a hit sound plays once per re-simulation. A slightly late sound beats a truncated one. |
| Debug assertions on allocation, audio, and I/O during simulation | Catches contract violations at introduction, not as a desync weeks later |
| **Redundant input packets instead of retransmission** | 16 frames of input is ~32 bytes; a dropped packet is covered by the next with no round-trip |
| Relay-capable topology; relay is still lockstep | A forwarding relay holds no authority. Costs one hop, buys NAT traversal and a desync observation point. |
| Build the relay before hole punching | A working relayed connection beats a direct one that fails for a fifth of players |
| Halve the frame-advantage difference when pacing | Full correction makes both peers overcorrect and alternate stalling |
| **Hierarchical checksums, not one number** | "The RNG diverged" versus "something diverged" is the difference between an afternoon and a week |
| Automatic state dumps with offline binary-search diffing | The definitive diagnosis technique, and desyncs are inevitable |
| **Soak harness built in week two** | A desync found by the soak test costs an hour because you have the seed; found by a player it costs a week |
| Checked-in cross-platform checksum constants in CI | Catches a platform divergence in the build, not in a match |
| Test rollback equivalence at every depth | Off-by-one ring bugs hide until exactly that depth occurs |

---

## Appendix B — Quick reference card

```
ARCHITECTURE — pick ONE
  ROLLBACK LOCKSTEP          SERVER-AUTHORITATIVE
  no authority               server is truth
  inputs only                snapshots down, inputs up
  determinism REQUIRED       determinism irrelevant
  predict REMOTE inputs      predict YOUR OWN movement
  correction = rollback      correction = reconciliation
  lag compensation: N/A      lag compensation: YES
  2–4 players                8+ players
  fighting games, melee      shooters

★ lag compensation = server rewinds OTHER players' hitboxes to
  (server_time − latency − interpolation). No server ⇒ not applicable.

PLAYER COUNT MATH
  rollback depth is set by the WORST peer — no isolation
  P(≥1 bad link) at 10% each:  2p → 19%   4p → 34%   8p → 57%
  cost = rollback_frames × sim_cost, EVERY frame
    16.67ms budget, 4ms for rollback, 8 frames ⇒ sim ≤ 0.5 ms
  full mesh at 8 players = 28 NAT traversals

DETERMINISM
  ★ NO float/double in the sim — lint-enforce it
  32.32 fixed-point, 128-bit intermediate on multiply (or it overflows)
  trig/sqrt = integer tables + Newton
  killers: transcendentals · FMA contraction · -ffast-math · x87 vs SSE
           hash-map iteration · pointer values · uninit padding
           std::sort instability · threads · wall clock · unseeded RNG
  checksum FIELDS not bytes (padding is uninitialized)

STATE
  ★ one contiguous POD arena, saved with ONE memcpy
  no pointers (indices) · no vectors · no allocation · no vtables
  fixed max counts, free lists over pools
  64 KB @ 60 Hz = 3.8 MB/s  ·  4 MB @ 60 Hz = 240 MB/s
  bounded ring, check the frame number on restore

THE LOOP
  send local input → receive remote → firstIncorrectPrediction?
    yes: restore(P), resimulate P..F
    no:  predict, simulate F
  carry-forward prediction (great for digital, poor for analog)
  input delay trades 1:1 against rollback depth; GGPO games use 2–4
  window exceeded ⇒ STALL, never continue

ROLLBACK-SAFE SIM
  ★ frames are simulated MULTIPLE times
  sim RECORDS events into state; presentation consumes on CONFIRMED frames
  otherwise the hit sound plays once per re-simulation
  debug-assert on allocation / audio / I/O inside Simulate()

TRANSPORT
  UDP, every packet carries the last 8–16 frames of input
  ⇒ ~3 KB/s per peer direction; loss needs no retransmission
  relay forwarding is STILL lockstep (no authority) — one hop for NAT sanity
  frame pacing: halve the advantage difference, don't fully correct

DESYNC
  detection is easy, DIAGNOSIS is the project
  hierarchical checksums: total · players · projectiles · rng · physics
  on mismatch: dump both sides, binary-search the first differing frame
  ★ soak test: thousands of headless matches, random latency/loss/inputs
    found by soak = 1 hour (you have the seed)
    found by a player = 1 week
```
