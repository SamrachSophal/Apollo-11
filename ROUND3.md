# ROUND3 — THE_LUNAR_LANDING.agc (line 317) → Modern JavaScript

## The original AGC code

From `Luminary099/THE_LUNAR_LANDING.agc`, lines 317–322, inside the `LANDJUNK`
sequence that follows the descent program flow:

```agc
		CAF	V06N43*		# ASTRONAUT:  NOW LOOK WHERE YOU ENDED UP
		TC	BANKCALL
		CADR	GOFLASH
		TCF	GOTOPOOH	# TERMINATE
		TCF	+2		# PROCEED
		TCF	-5		# RECYCLE
```

## What this little piece of logic does

1. `CAF V06N43*` loads the display code **Verb 06, Noun 43** (`V06N43*` is
   defined later as `VN 0643`) into the accumulator.
2. `TC BANKCALL` / `CADR GOFLASH` calls the `GOFLASH` routine across banks,
   which **flashes** that verb/noun on the DSKY so the astronaut can review
   where the spacecraft "ended up" before committing to the landing guidance
   (P57) initialization.
3. When the astronaut responds, control fans out to exactly one of three
   destinations:
   - **TERMINATE** → `TCF GOTOPOOH` — abandon the current major mode and go
     back to P00 (program selection). `GOTOPOOH` literally means
     *"way to P00"*.
   - **PROCEED** → `TCF +2` — continue two words ahead into the P57
     landing-initialization sequence (`TC INTPRET`, initializing `GSAV`,
     `YNBSAV`, `ZNBSAV`, and `ATTFLAG` for P57).
   - **RECYCLE** → `TCF -5` — jump back five words and flash the `V06N43*`
     prompt again (ask the astronaut again).

## Modern JavaScript rewriting

```js
// ============================================================================
// THE_LUNAR_LANDING.agc (Luminary099) — lines 317-322 — rewritten in modern JS.
//
// Original AGC:
//
//   CAF	V06N43*		# ASTRONAUT:  NOW LOOK WHERE YOU ENDED UP
//   TC	BANKCALL
//   CADR	GOFLASH
//   TCF	GOTOPOOH	# TERMINATE
//   TCF	+2		# PROCEED
//   TCF	-5		# RECYCLE
// ============================================================================

// Verb 06, Noun 43 -- the "look where you ended up" display code
// (defined in the AGC as:  V06N43*  VN  0643).
const V06N43 = Object.freeze({ verb: '06', noun: '43' });

// The three ways the astronaut can answer the flashed prompt.
const RESPONSE = Object.freeze({
  TERMINATE: Symbol('TERMINATE'),
  PROCEED: Symbol('PROCEED'),
  RECYCLE: Symbol('RECYCLE'),
});

// GOTOPOOH -- "way to P00": terminate the current major mode and return
// to program selection. (See BURN_BABY_BURN: "WAY TO P00 (GOTOPOOH RELINTS)".)
async function GOTOPOOH(reason) {
  console.log(`GOTOPOOH (${reason}): returning to P00, program select`);
}

// What the PROCEED branch (+2) leads to in the original source: enter the
// P57 landing-initialization sequence (GSAV / REFMF -> YNBSAV, ZNBSAV,
// ATTFLAG), then prompt the astronaut to select P57.
async function proceedToP57Landing() {
  console.log('P57 landing guidance initialized -- ASTRONAUT:  PLEASE SELECT P57');
}

// ----------------------------------------------------------------------------
// The translated logic -- the "little piece" from line 317.
// ----------------------------------------------------------------------------

/**
 * Flash V06N43 on the DSKY and route the astronaut's answer exactly like the
 * three-way TCF fan-out in LANDJUNK:
 *   TERMINATE -> GOTOPOOH (P00), PROCEED -> P57 init, RECYCLE -> re-flash.
 *
 * @param {{ goFlash: (code: unknown) => Promise<symbol> }} dsky
 *        Minimal DSKY: goFlash() shows a flashing verb/noun and resolves with
 *        the astronaut's RESPONSE.
 * @returns {Promise<void>}
 */
async function lookWhereYouEndedUp(dsky) {
  for (;;) {
    // ASTRONAUT:  NOW LOOK WHERE YOU ENDED UP
    const response = await dsky.goFlash(V06N43); // CAF V06N43* / TC BANKCALL / CADR GOFLASH

    switch (response) {
      case RESPONSE.TERMINATE:
        // TERMINATE
        return GOTOPOOH('crew terminated the V06N43 prompt'); // TCF GOTOPOOH

      case RESPONSE.PROCEED:
        // PROCEED
        return proceedToP57Landing();                        // TCF +2

      case RESPONSE.RECYCLE:
        // RECYCLE
        continue; // TCF -5 -> loop back and flash the prompt again

      default:
        throw new Error(`Unknown DSKY response: ${String(response)}`);
    }
  }
}

// ----------------------------------------------------------------------------
// Minimal runnable demo (mock DSKY) so the function above can be exercised.
// ----------------------------------------------------------------------------
async function main() {
  let presses = 0;

  const mockDsky = {
    async goFlash({ verb, noun }) {
      presses += 1;
      console.log(`DSKY: flashing V${verb} N${noun} (ask #${presses})...`);

      // Simulate the astronaut: first RECYCLE (look again), then PROCEED.
      return presses === 1 ? RESPONSE.RECYCLE : RESPONSE.PROCEED;
    },
  };

  await lookWhereYouEndedUp(mockDsky);
}

main();
```
