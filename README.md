# Wheel Glitch Filter (AutoHotkey v2)

A low-latency, deterministic AutoHotkey v2 script that filters out spurious
mouse-wheel signals caused by a faulty rotary encoder — for example a real
`Down` scroll motion that occasionally reports:

```
Down, Down, Down, Up, Down, Down
```

instead of just `Down`. Built to be safe for latency-sensitive use in
**Minecraft** and general browsing.

## How the algorithm works

Every tick that matches the **current recognized direction** is forwarded
**immediately** — zero buffering, zero added delay — including its exact
notch count (`A_EventInfo`), so wheel speed/sensitivity is fully preserved.

Only **opposite-direction** ticks go through a decision, made instantly using
only past data:

- Each direction has a **confidence score** (capped at `DirectionConfidence`).
  It rises with every confirming tick and falls with every opposing tick.
- The weight of an opposing tick depends on **timing**: one arriving faster
  than `MaxGlitchInterval` is treated as a normal glitch-weight signal; one
  arriving slower gets extra weight, because a real direction change requires
  the hand to physically reverse motion, which takes measurable time.
- A direction reversal is only accepted when **both** conditions are true:
  the number of consecutive opposing ticks reaches `RequiredOppositeTicks`,
  **and** the confidence score has actually crossed over to the opposite
  side. This is the confidence-based system requested instead of a plain
  fixed counter.

Because the decision (send or drop) is made the instant an event arrives,
no tick — sent or dropped — is ever delayed. That's why the filter adds no
noticeable input latency.

## Settings (all at the top of the file)

| Setting | Purpose |
|---|---|
| `RequiredOppositeTicks` | Minimum consecutive opposite ticks needed to accept a direction reversal |
| `MaxGlitchInterval` | Time threshold (ms) below which an opposite tick is considered suspicious |
| `DirectionConfidence` | Confidence score cap for either direction |
| `ResetTimeout` | Idle time (ms) after which the internal state fully resets |
| `ConfidenceIncrement` / `ConfidenceDecrement` | How much confidence is gained/lost per tick |
| `EnableReversalBuffer` | Optional mode explained below (off by default) |

Hotkeys: **F8** toggles the filter on/off, **F9** toggles debug logging. Both
show a status tooltip.

## Testing

1. Press **F9** to enable debug mode, then tail the log live:
   `Get-Content ".\WheelFilterDebug.log" -Wait -Tail 20` (PowerShell), or
   reopen it periodically in any text editor.
2. Scroll normally in Notepad and watch for `SUPPRESS(glitch)` lines — this
   reveals your mouse's specific glitch "signature" (how many opposite ticks
   it typically fires, and how fast), which tells you how to tune the
   settings.
3. Do a fast, genuine direction reversal (scroll Down, then immediately Up)
   and confirm `SEND(flip)` appears quickly and feels responsive.
4. Test in Minecraft: fast hotbar-slot flicking to confirm no notches are
   lost, and no perceptible lag.
5. Press **F8** to confirm the wheel behaves 100% natively when the filter
   is off.

## Recommended starting values for Minecraft

The defaults already shipped in the script:
`RequiredOppositeTicks = 2`, `MaxGlitchInterval = 130`,
`DirectionConfidence = 10`, `ResetTimeout = 600` — a good balance between
rejecting single-tick glitches (the most common failure mode) and reacting
quickly to real reversals.

## Known limitation, and a better option

**The unavoidable trade-off:** since the decision uses only past data (no
lookahead, no delay), the very first tick of a genuine fast reversal is
mathematically indistinguishable from a glitch at the moment it arrives.
So each real direction change loses roughly `RequiredOppositeTicks` worth
of ticks at the start of the reversal. That's the price of a zero-latency
causal filter.

**A better option (already implemented, opt-in):** set
`EnableReversalBuffer := true`. This holds only the first ambiguous
opposite tick for up to `MaxGlitchInterval` ms; if it's confirmed as a real
reversal it's flushed in full, otherwise it's dropped. The added delay is
tiny and only ever happens at the exact moment of a direction change — not
during normal continuous scrolling — trading a barely perceptible pause for
zero lost ticks on real reversals.

**Why not machine learning:** this is a simple, well-understood
deterministic hardware fault (encoder contact bounce/glitch), not a pattern
that benefits from learning. A tuned heuristic state machine like this one
is faster, more predictable, and far easier to debug than any local ML
model, which would add real per-event latency for no real accuracy gain. If
glitches remain frequent even after tuning, the real fix is the mouse's
hardware itself (cleaning the encoder, firmware update, or replacement) —
this script is a workaround, not a repair.

## License

Feel free to use, modify, and redistribute.
