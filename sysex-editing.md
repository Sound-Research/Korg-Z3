# Korg Z3 SysEx editing

Reference for editing the Korg Z3 over MIDI SysEx,
covering both per-string MIDI assignments and the internal FM voice parameters.
All byte-level facts in this document are quoted from the *Z3 Owner's Manual* MIDI Implementation section (`Z3_OM_E.pdf`, pp. 23–32).

## What's actually inside the Z3

The Z3 is two distinct subsystems sharing one chassis:

- **Hex-to-MIDI converter** — takes the ZD3 24-pin divided-pickup signal and emits MIDI.
- **Internal tone generator** — a Yamaha **YM2414 (OPZ)** 4-operator FM chip, 8-voice polyphonic.
  Same chip used in the Yamaha TX81Z, DX11, and YS200/V50 family.

Both subsystems are addressed through the same SysEx interface — there is no separate "hex converter SysEx" vs "FM chip SysEx" data path.
Different *function codes* within the Korg SysEx wrapper address the different parameter spaces (see [SysEx architecture](#sysex-architecture) below).

## Rear-panel switches

The Z3 has two separate switches on the rear that affect MIDI / edit behaviour. They are commonly confused.

### Mode A / Mode B switch (recessed)

- A small **recessed** switch, designed to require deliberate action — toggle with a pen or thin tool **with the unit powered off**, then power on.
- **Mode A** — Play mode. The 128 internal Programs are selectable but cannot be edited; the front panel is locked to performance functions.
- **Mode B** — Edit mode. Programs become editable from the front panel. The per-string MIDI channel, sound selection, bend range, transpose, velocity curve, sensitivity, hold, output level, and program-change-on-select fields are all exposed.
- This switch is **not** a SysEx routing switch. SysEx editing of the FM voice (Sound parameters) works regardless of Mode A/B — but writing edited Programs to memory requires Mode B.

### MIDI IN / FRONT REMOTE IN switch

- A separate switch that selects what the **MIDI IN jack** does.
- **MIDI IN** — accepts incoming MIDI from a sequencer/computer (this is what you want for SysEx editing).
- **FRONT REMOTE** — accepts the ZD3 driver's remote/footswitch control via the front-panel jack.
- For any SysEx work, leave this switch on **MIDI IN**. For normal guitar playing, switch back to **FRONT REMOTE**.

## SysEx architecture

### Header (always the same)

Every Z3 SysEx message starts with the four bytes:

```
F0  42  3n  1D
│   │   │   └── Z3 model ID
│   │   └────── Format ID (n = basic MIDI channel, 0–15)
│   └────────── Korg manufacturer ID
└────────────── SysEx start
```

This is **distinct from a Yamaha TX81Z header** (`F0 43 …`), which is why TX81Z editors do not work with the Z3 even though the underlying FM chip is the same.

### Three parameter spaces

The Z3 exposes three independent address spaces, each with its own function codes:

| Space | What it holds | Dump from buffer | Dump from memory | Parameter change | Buffer dump request | Memory dump request | Write |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **Sound** | One 4-op FM voice (50 params) | `40h` | `4Ch` | `41h` | `10h` | `1Ch` | `11h` |
| **Program** | One per-string setup (55 params) | `49h` | `4Dh` | `59h` | `19h` | `1Dh` | `0Ah` |
| **Patch** | Bank arrangement (8 banks × 8 patches → Program no.) | `5Ch` | `5Dh` | — | `0Ch` | `0Dh` | `0Fh` |

Function code goes in the byte after the header. Acknowledgement messages: `21h` write completed, `22h` write error, `23h` data load completed, `24h` data load error, `26h` exclusive format error.

## Putting each string on its own MIDI channel

This lives in the **Program** space, parameters 7–12:

| Parameter no. | String | Field | Data byte (in change msg) |
| --- | --- | --- | --- |
| 7 | String 1 (low E) | MIDI Channel | `000C CCCC` |
| 8 | String 2 (A) | MIDI Channel | `000C CCCC` |
| 9 | String 3 (D) | MIDI Channel | `000C CCCC` |
| 10 | String 4 (G) | MIDI Channel | `000C CCCC` |
| 11 | String 5 (B) | MIDI Channel | `000C CCCC` |
| 12 | String 6 (high E) | MIDI Channel | `000C CCCC` |

Values: `0`–`15` → MIDI channel 1–16 respectively. In the in-memory edit buffer the byte is `FFh` to disable output for that string; the change command treats `0` as channel 1.

### Front-panel route (no SysEx needed)

Switch to Mode B, then assign each string a different MIDI Channel parameter from the per-string edit page.
The manual notes a shortcut for resetting all six strings to the same channel: hold **FUNCTION** + press **STRING SELECT key 2**.
In Mode A this collapses every string to MIDI channel 1; in Mode B it collapses them all to string #1's current channel.

### SysEx route (single-parameter change)

Send a Program Parameter Change (`59h`):

```
F0  42  3n  1D  59  pp  dd  F7
                    │   └── data: 0–15 = MIDI ch 1–16
                    └────── parameter no. 07–0C (string 1–6 MIDI Channel)
```

Example: set String 1 to MIDI channel 1 with basic channel 1 →
`F0 42 30 1D 59 07 00 F7`.

To send all six in one go, use **Program Parameter Dump** (`49h`) with the full 55-byte payload — see the *Program Parameter Dump Data Format* table in the manual (p. 28).

## Editing the FM voice (the Yamaha chip)

The Sound parameter space exposes the full 4-operator OPZ voice model in 88 parameter slots (numbered 0–87).
The four operators are ordered **M-1, M-2, C-1, C-2** (two modulators, two carriers).
Single-parameter edits use Sound Parameter Change (`41h`):

```
F0  42  3n  1D  41  pp  hh  [ll]  F7
                    │   │    └── low byte (LFO Rate only — param 11)
                    │   └─────── data hi byte
                    └─────────── parameter no. 00–57h
```

### Sound parameter map

Global / voice-level params (one per voice):

| No. | Name | Range |
| --- | --- | --- |
| 0–7 | Voice name (8 ASCII chars) | — |
| 8 | Algorithm | 0–7 (8 algorithms) |
| 9 | Feedback | 0–7 |
| 10 | LFO Waveform | 0–3 |
| 11 | LFO Rate (16-bit) | 0–255 |
| 12 | PMD (pitch mod depth) | 0–127 |
| 13 | PMS (pitch mod sensitivity) | 0–7 |
| 14 | AMD (amp mod depth) | 0–127 |
| 15 | AMS (amp mod sensitivity) | 0–3 |

Per-operator params (4 entries each, ordered M-1 / M-2 / C-1 / C-2):

| No. range | Name | Range | Notes |
| --- | --- | --- | --- |
| 16–19 | OSC Waveform | 0–7 | **OPZ extra**: 8 waveforms per operator, not just sine |
| 20–23 | MUL1 | 0–15 | Integer part of frequency multiplier |
| 24–27 | MUL2 | 0–15 | **OPZ extra**: fractional part for non-integer ratios |
| 28–31 | Detune1 | 0–7 | Coarse detune |
| 32–35 | Detune2 | 0–3 | Fine detune |
| 36–39 | Total Level | 0–127 | Operator output level |
| 40–43 | Attack Rate | 0–31 | |
| 44–47 | Decay 1 Rate | 0–31 | |
| 48–51 | Sustain Level | 0–15 | |
| 52–55 | Decay 2 Rate | 0–31 | |
| 56–59 | Release Rate | 0–15 | |
| 60–63 | Key Scale | 0–3 | |
| 64–67 | AMS Enable | 0/1 | |
| 68–71 | EG Shift | 0–3 | |
| 72–75 | Reverb Level | 0–15 | Z3 internal reverb send |
| 76–79 | Reverb Rate | 0–7 | |
| 80–83 | Velocity Int | 0–15 | Velocity sensitivity |
| 84–87 | Keyboard Track | 0–15 | |

To audition a change, send Sound Parameter Change (`41h`) — it writes to the edit buffer, so the next note you play will use it.
To save the edit to one of the 128 Sound memory slots, send Sound Write Req (`11h`) with the destination sound number.

## Existing editors

There is no actively maintained commercial editor for the Z3 on any modern platform.
The two historical editors and the realistic ways to run them today:

### Emagic SoundDiver 3.0.5.4 + Z3 adaptation — recommended

- **The editor.** Final Windows release `3.0.5.4` (Dec 2002) is preserved on the [Internet Archive](https://archive.org/details/emagicsounddiver3.0.5.4). Free.
- **The Z3 adaptation.** A community-built SoundDiver adaptation gives librarian, full sound editor, and a per-string Program panel. Hosted on [gr300.com](https://gr300.com/korgz3.htm). Free. Adaptation file format is cross-platform — the Mac adaptation loads into Windows SoundDiver and vice versa.
- **Running it.** Several documented options on modern macOS:
  - **Wine** (or **Porting Kit** / **Kegworks** / **CrossOver**) — same approach that works for the Fantom editor. Documented working for SoundDiver 3.0.5.4 on Apple Silicon. Known gotcha: opening the MIDI Preferences dialog has been reported to crash the app under some Wine versions; workarounds exist (pre-configure MIDI before opening Settings, or pin to an earlier Wine version).
  - Direct Windows VM (UTM/Parallels) — heavier setup, more reliable.
- **Why this one over Unisyn.** Free, the adaptation is purpose-built for the Z3, the platform survives on Wine.

### MOTU Unisyn — not recommended in 2026

- Was the standard commercial editor; last Mac release ~2008, last Windows release 1.5 (1997, Win 3.1/95/98/ME/2000).
- No documented Wine / CrossOver success.
- Out of print; second-hand boxed copies turn up but the install/auth flow is fragile.
- If you can find a working install and need an alternative to SoundDiver, fine — but not worth chasing.

### Coffee Shopped Patch Base — file a vote

- Z3 is **not** currently supported, but Patch Base is actively developed for modern macOS / iOS and the team adds new synths from their request queue.
- Worth submitting a request via their voting page. Long shot, but zero-effort.

### Sound Quest Midi Quest 13

- Z3 is **not** in their supported instrument list (Korg Z1 is). Their "custom instrument" framework is technically there but constructing a Z3 definition is DIY work — not an "off the shelf" product path.

### Fallback if no editor runs

For *just* the per-string MIDI channel goal, Mode B's front-panel edit covers it without any external tool. For FM voice editing, *Snoize SysEx Librarian* (free, Apple Silicon native) can send hand-built SysEx files using the byte tables in this document — useful for bulk dumps and one-off parameter pokes but not a substitute for a real editor GUI.

## TX81Z compatibility myth

Because the Z3 uses the same YM2414 chip as the TX81Z, it is occasionally suggested that TX81Z editors might work.
They do not.
The Korg SysEx wrapper (`F0 42 …`) and the Yamaha SysEx wrapper (`F0 43 …`) are entirely separate — a TX81Z voice dump sent to a Z3 is silently ignored.
The voice *parameter layout* is broadly OPZ-shaped, so a TX81Z patch could in principle be transcoded into a Z3 Sound dump by hand or script, but no off-the-shelf tool does this transcoding.
