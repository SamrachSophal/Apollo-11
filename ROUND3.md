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

## Translation fidelity — the human-in-the-loop

The original `GOFLASH` call is a **hard gate**: the crew must physically
verify the flashing V06N43 on the DSKY and press a key before the software is
allowed to advance into the P57 landing sequence. That gate is **not** dropped
in the translation. The translated function `await`s a `goFlash()` promise
that only a real DSKY keypress may resolve — there is **no auto-proceed
default** — and the RECYCLE branch re-arms the prompt (loops and re-awaits)
instead of advancing. The astronaut survives as a value returned by an
I/O callback (`RESPONSE.TERMINATE | PROCEED | RECYCLE`).

One honest caveat: the runnable demo at the bottom replays a *scripted*
astronaut mock so the snippet can be executed headlessly for validation. That
mock is a **test harness, not part of the translated logic** — in a real Node
deployment you would implement `goFlash` by wiring it to `readline`/the
keyboard (or a DSKY-emulator port), so the bot waits on a live human keypress
exactly as `GOFLASH` waited on the astronaut.

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
---

## Short answers

**Q1 — Does the 1969 comment still make sense on 2026 code?**
No on its own; yes only as a labeled artifact. `ASTRONAUT:  NOW LOOK WHERE YOU
ENDED UP` means nothing to a modern reader without provenance — it works in
ROUND3.md only because the original `.agc` block sits right beside it. The
branch comments (`TERMINATE`/`PROCEED`/`RECYCLE`) survive better, since they
still name three real control-flow targets. All four are kept verbatim because
that was the requirement, not because they'd be good docs written fresh.

**Q2 — Did the check-on-the-human survive, or did it get dropped?**
It survived. The translated `goFlash()` can only resolve on a live keypress —
there is no auto-proceed default — and the `RECYCLE` branch re-arms the prompt
instead of advancing, so the astronaut decides every time. The only scripted
human is in the runnable demo (a labeled test harness, not part of the
translated logic). Wire `goFlash` to `readline`/keyboard input and the human is
end-to-end real.

### Archive Comment in 1969 Apollo style:
Here's the comment, written as a mission-control systems report you can paste straight into the code (it's a valid JS block comment)

APOLLO 11 — SEARCH MODULE SYSTEMS STATUS REPORT
MISSION CONTROL, HOUSTON
TRANSMITTED 15 OCTOBER 2026, 20:17 UTC
THE SEARCH MODULE IS OPERATIONAL. ITS CAPABILITIES ARE
STRICTLY LIMITED. ALL PERSONNEL ARE ADVISED OF THE
FOLLOWING SYSTEMS CONSTRAINTS BEFORE COMMENCING SEARCH
OPERATIONS. THIS IS A NO-GO FOR ADVANCED QUERIES. OVER.

ITEM 01 — SEARCH SCOPE: TITLE ONLY
THE PROBE SCANS THE TITLE FIELD AND THE TITLE FIELD ALONE.
DESCRIPTION, PLACE, AND CONTRIBUTOR ARE NOT WITHIN THE
SCAN ENVELOPE, THOUGH THEY ARE DISPLAYED ON EVERY CARD.
A STORY ABOUT THE MEKONG WILL NOT BE FOUND BY SEARCHING
"MEKONG" UNLESS THE MEKONG IS IN THE TITLE. THIS IS NOT
A BUG. IT IS A LIMITATION. WE COPY.

ITEM 02 — NO FUZZY MATCHING / NO TYPO TOLERANCE
THE MATCH IS RAW AND LITERAL. "BONN" WILL NOT FIND
"បុណ្យ". ONE MISSPELLED CHARACTER AND THE MISSION
RETURNS ZERO RESULTS. THERE IS NO FORGIVENESS IN THIS
MODULE.

ITEM 03 — NO MULTI-TERM LOGIC
THE ENTIRE QUERY IS ONE SINGLE SUBSTRING. "NEW YEAR"
MUST APPEAR IN EXACTLY THAT ORDER. THERE IS NO OR.
THERE IS NO AND. THERE IS NO SPLITTING OF WORDS. THE
MODULE DOES NOT DECOMPOSE. IT ONLY COMPARES.

ITEM 04 — NO RELEVANCE RANKING
RESULTS ARE ALWAYS SORTED BY YEAR, DESCENDING. THE
MODULE DOES NOT ASK WHICH RESULT BEST MATCHES THE QUERY.
IT ASKS ONLY WHICH YEAR IS NEWEST. RELEVANCE IS NOT
A FACTOR IN THIS FLIGHT.

ITEM 05 — NO PAGINATION
EVERY MATCHING ROW IS FETCHED IN A SINGLE PASS. THERE
IS NO SECOND STAGE. THERE IS NO THIRD STAGE. THE MODULE
COMMITS ITS ENTIRE PAYLOAD AT ONCE.

ITEM 06 — NO MATCH HIGHLIGHTING / NO FILTERS
MATCHES ARE NOT ILLUMINATED. FILTERS BY YEAR, PLACE,
OR CONTRIBUTOR CANNOT BE COMBINED WITH THE SEARCH. THE
SEARCH FLIES ALONE.

ITEM 07 — KHMER DIACRITICS ARE LITERAL
"បុណ្យ" WITH SLIGHTLY DIFFERENT VOWEL MARKS WILL NOT
MATCH "បុណ្យ". THE MODULE DOES NOT NORMALIZE. IT
COMPARES BYTE FOR BYTE, MARK FOR MARK.

END OF TRANSMISSION. THE MODULE IS GO FOR SEARCH, BUT
ONLY WITHIN THE ENVELOPE DESCRIBED ABOVE. OVER.
