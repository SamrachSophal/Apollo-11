OFF TO SEE THE WIZARD... - THE_LUNAR_LANDING.agc 254
SEE IF HE'S LYING - THE_LUNAR_LANDING.agc 249
SILLY THING AROUND - THE_LUNAR_LANDING.agc 246
NOW LOOK WHERE YOU ENDED UP - THE_LUNAR_LANDING.agc 317
TEMPORARY, I HOPE HOPE HOPE - LUNAR_LANDING_GUIDANCE_EQUATIONS.agc 179 180
NUMERO MYSTERIOSO - LUNAR_LANDING_GUIDANCE_EQUATIONS.agc 666
NO ROOM IN THE INN - WAITLIST.agc 335
TRASHY LITTLE SUBROUTINES - LUNAR_LANDING_GUIDANCE_EQUATIONS.agc 1375
COME AGAIN SOON - BURN_BABY_BURN--MASTER_IGNITION_ROUTINE.agc 925
DO NOT USE GOPROG2 OR ENEMA WITHOUT CONSULTING POOH PEOPLE. - FRESH_START_AND_RESTART.agc 353

Round 2:

## The code (lines 251–255) - THE_LUNAR_LANDING

```asm
P63SPOT4	TC	BANKCALL	# ENTER		INITIALIZE LANDING RADAR
		CADR	SETPOS1

		TC	POSTJUMP	# OFF TO SEE THE WIZARD...
		CADR	BURNBABY
```

**What it does:**
- `SETPOS1` (via `BANKCALL`) initializes landing-radar software so the radar can lock onto the surface.
- `TC POSTJUMP / CADR BURNBABY` = permanent, cross-bank jump (bank 32 → bank 36) with **no return**.
- `BURNBABY` is the shared Master Ignition Routine that flies the actual descent burn (TIG countdown, engine ignition, throttle-up, steering).
- P63-specific behavior comes from `WHICH` → `P63TABLE`, set at P63 start.

**What calls it:**
- Nothing — it's the terminal fall-through of P63's own setup chain.
- Chain: `P63LM` (entry, IMU check) → `IGNALG` (compute ignition time) → `P63SPOT2` (burn attitude) → `P63SPOT3` (LR antenna check) → `P63SPOT4` (radar init) → **here** → `BURNBABY`.

**Why "OFF TO SEE THE WIZARD…":**
- Wizard of Oz joke: P63 (Dorothy) finishes the prep work and sets off to meet the "great and powerful" BURNBABY (the Wizard) — the behind-the-curtain magic that does the real landing.
- Typical playful humor of the Apollo code.

## The code (lines 666 ) - LUNAR_LANDING_GUIDANCE_EQUATIONS

**Line 666 is in `CGCALC`** (`LUNAR_LANDING_GUIDANCE_EQUATIONS.agc`, page 810).

**What it does:** `CGCALC` re-erects the 3×3 `CG` matrix that rotates from the landing-site vertical ("guidance") frame to the IMU platform ("stable member") frame, once per guidance pass. The line itself computes a "predicted aim point": `a = unit(LAND − R + ANGTERM·(TTF/8)·GAINBRAK,1)`, where `ANGTERM` is surface-relative velocity and `TTF/8` is time-to-go. The matrix's down-range axis is built to *lead* the landing site along the ground track by that predicted-displacement term.

**What calls it:** The post-guidance jump table `AFTRGUID` (lines 71–73), reached every pass for the ignition, braking (P63), and approach (P64) phases via `BRSPOT4 → AFTRGUID → CGCALC`. Not called in the vertical phase (P65–67).

**Why the comment:** `GAINBRAK,1` is a 2-word braking-phase gain stored as padload data (no value in the source; no derivation documented). It's an empirically tuned fudge factor that controls how far ahead of the landing site the guidance frame aims. The MIT programmers didn't know/record exactly why that number, so they wryly tagged it "NUMERO MYSTERIOSO" ("the mysterious number") — one of the famous funny comments in the Apollo source.

## The code (lines 353) - FRESH_START_AND_RESTART

- File: `Luminary099\FRESH_START_AND_RESTART.agc` (LM AGC; the only copy with the line-353 comment).

**GOPROG2 (line 284)**
- Software-restart entry point.
- Runs `STARTSB2` (gentle init preserving ENGINE/IMU/telemetry state), then `LIGHTSET` + clears FLGWRD10 bits, then flows into common GOPROG3.
- GOPROG3 verifies phase table (alarm 1107 if bad), shows major mode, re-schedules active restart groups.
- Called here by Verb 37 (major-mode change) via `GOGOPROG` (lines 898–899, 936) — the "tear down current program, start new one" hammer.

**ENEMA (line 281)**
- Legacy "software restart on major mode change," sibling of GOPROG2.
- Runs fuller `STARTSB1` init, then joins GOPROG2A → LIGHTSET → GOPROG3.
- Not referenced as a call target anywhere in this file (unwired/legacy entry).

**Why the comment**
- Both are destructive forced restarts that wipe running programs and need a correctly set phase table (checked at GOPROG3 → alarm 1107).
- "POOH people" = owners of the P00 / major-mode forced-restart logic (`GOTOPOOH` line 642, `POOH` line 850, the P00 "POOH request" checks in V37).
- Warning: don't use them as a casual reset; coordinate with the POOH/P00 maintainers first or you'll corrupt machine state.