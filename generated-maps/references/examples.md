# Worked examples

Real cases from the project the method comes from (a C# game mod, about 1,400 types, about 6,300 tests,
almost all AI-written under one person's direction). One per mechanism.

**Why maps at all.** "How friendly must a character be to count as a friend?" had five answers:
`FriendThreshold = 10` private in three files, `BestFriendThreshold = 20` and `FriendRelationThreshold = 20` in
two more. Every class was clean; the concept was scattered. The instructions file listed 16 prompt sections;
the code had 108. A generated rules document was ten days and about 600 rules behind because nothing forced
its regeneration.

**Day one of the maps.** Found by reading them, none by a user: a feature declared in six places out of seven
(unreachable for months); a gate reading the wrong one of two scores over the same range; gratitude granted to
infants where every comparable path excluded children; a payment ceiling in five places; a dependency cycle
spanning 37 namespaces.

**Update mode hides drift.** For days the model ran the regeneration first, then the tests. Every informing map
compared against itself and passed. The owner: "if you generate the maps before running the tests, you will
never see a diff in inform mode." Hence the fixed order: plain run, read, regenerate.

**Contract guard, first live case.** A public member left the API map with no sound (harmless: it had never
shipped). The guard was built against the RELEASED snapshot. Demonstrated the same day: renaming `PlaceId` to
`Place` was refused even in update mode: "These public members shipped in v2.5.23 and are no longer there."

**Ratchet, loud false positive.** The bare-number regard ratchet caught four new constants named `...Regard`.
They were shifts applied to the score, not thresholds. Fixed by renaming them (`BetrayalPenalty`,
`ReleasedGratitude`), not by raising the baseline. Lesson: name a delta as a delta.

**Silent miss, the other face.** The same name-based detector never saw 33 thresholds compared inline
(`regards.Average() < 0`, `< 10`, `>= 60`). A fact-based pass (what the code actually compares against the score)
found them; a second ratchet now holds that count, and it went 33 -> 27 as rungs got names.

**Blind map, fix the code.** The "which score" column of the decision map was "?" for 10 of 12 rules. The code
was implicit, so the parameters were typed; the map now reads the answer from the type. On the way it exposed a
parameter misnamed for the other score, and a generator that read only a doc comment's first sentence.

**A map proposing architecture.** The event-listener map showed the same "temporary unit" shape three times
(spawn, steer, never saved, despawn before save, suppress the destroy event). It became one shared registry;
before-save listeners went 3 -> 2. The oldest of the three was left for the owner to decide.

**What no map caught.** While consolidating a calendar constant, the model put a game-specific value into the
engine-agnostic layer. The owner caught it within the hour by reading the code. The maps did list every
consequence of the fix (four new mutable statics), which is where that cost belongs.

## Second project: a zero-dependency JavaScript app (Relais, 26/10/2026)

The skill's first run on another codebase, plain JavaScript with no npm dependency
([A-Postman-Alternative](https://github.com/LogRaam/A-Postman-Alternative)). Bootstrapped in the order above in
one session; what the first loop showed:

- **Silent generator error on day one.** The inventory reported a file's header comment as the doc of its first
  constant. Nothing failed; it read plausibly. Fixed by counting a doc only when it touches its declaration.
- **Implicit code, fixed in the code.** Twelve private helpers showed "no doc". One line each, not a generator change.
- **Generators are code too.** One map named a test file that did not exist; another broke its own Markdown table
  on `number|null`. Both were visible only by READING the first maps, which is why step 3 of the loop is reading.
- **Guards proven, not assumed.** On a throwaway copy, a released export field removed was refused by the test and
  by the update command; a duplicated top-level name was refused by the build.

## Third project: a personal Python/Tkinter desktop tool (11/11/2026)

A one-person Windows tool that sorts video files into folders and keeps a persistent transfer queue
(about 2,000 lines of Python, no tests before the maps, built in one long session before the skill was loaded).
Bootstrapped afterwards; what the first loop showed:

- **Line numbers made the first constants map noisy.** Entries were located as `module:326`; any line added
  above would have turned the map red. Relocated by enclosing function (`Row.render_meta`).
- **Numbers alone were noise, shapes were signal.** "Same number in several places" grouped the `3` of a
  3-second delay with the `3` of a three-way split. "Same expression shape" found the real duplicate on the
  first run: a duration tolerance `max(120, 0.1 * _)` written once in the display code and once in the file
  comparison, so changing one would silently leave the other behind. It needed one generator fix to be seen:
  a nested `max(d1, d2)` had to count as a plain value.
- **A contract with no key list.** The saved queue was a dict literal written in one method and read with
  `d.get(...)` in another. Keys were classified from syntax (read with a default = optional). The first snapshot
  equalled the 17 keys of the user's real saved file. Proven on a throwaway copy: a released key removed was
  refused by the test and by the update command; a new key read as required was refused; the override passed.
- **The real fixture was personal data.** Kept structure and types, every string replaced by `"<key>"`.
- **No releases, no behaviour tests.** "Release" was defined as replacing the running executable; the rules
  index was left out until a first behaviour test exists.

## Same first project, two months later (14/11/2026)

**Asked for "SOLID maps".** The owner asked for maps that would put SOLID and Tell Don't Ask into practice. What
was built: two guards named after incidents, not principles. (1) Decisions made in the glue against a bare number
(a fact read off an object compared with a literal outside the pure layer), counted by a ratchet: a naive pass gave
330 lines, the filtered one 16, and the first run showed a town's loyalty threshold written in two files (`< 50` in
one, `>= 50` in the other: the same 50, agreeing today, free to drift) and the engine's default coming-of-age age
written as a fallback in five places. The first report to the owner overstated both ("decided the opposite way", "a
second age rule"); reading the lines corrected it the next turn. (2) The swap-risk script moved into the
suite (below). Declined, with reasons: a Demeter map (the engine API is nothing but chains), interface maps (almost
none exist), a type-size ratchet (splitting is the owner's call).

**A manual tool rots.** The swap-risk script (public members with three same-typed parameters in a row) had taken
the pure layer to zero two months earlier. Nothing ran it afterwards. Ported into the suite, it found 33; the
script, run the same day, listed the same 33, which validated the port. One of them was a key function widened that
same morning by the model writing the port. The ratchet starts at 33. On the way: an `out` parameter is part of a
signature's shape (`string a, out string b` cannot be swapped), and the first port flagged exactly that.

**The cycle ratchet, twice in one day.** A new public enum used from two namespaces drew them into the main cycle;
a new namespace for a small behaviour was drawn in the moment the startup code and a self-test used it. Both times
the fix was to place the code in a namespace already in the group (or mirror the enum in the pure layer), never to
widen the baseline. Both times the message said only "a namespace joined a cycle"; naming the closing edges would
have saved the guessing (now a generator requirement).

**The compiler as a map.** Translating settings choices meant giving each list item an id, but the code compared
the chosen item's text ("Off", "Mature"). Changing the list's item type from a string to a small type (id for the
display, English for the code) made the compiler list every read and write; one self-test looked items up with
`IndexOf("Occasionally")` and would have returned -1, silently, for every one.

## Fourth project: a series of programmed training videos (24/11/2026)

Each episode is its own repository, copied from the previous one: a React/Remotion animation keyed to the words
of a TTS narration, Python tools around it. Three maps (inventory, narration cues, every word the learner reads or
hears) and blocking checks: every cue resolves in the script, and no out-of-scope word in the script, on screen or
in the voice's transcript. Episode 3 was the first built from a copy; what it showed:

- **The screen-text map was blinder than its header said.** The header promised "literals passed to tr()"; no
  code read them. Labels typeset through `tr()`, a module-level list of phase names rendered by a loop, and a
  string placed straight in the markup (`{today ? "AUJOURD'HUI" : ...}`) were all missing, so the blocking scope
  check never scanned them. Found only because the map's list looked short next to the stills. Fixed in the
  generator (`tr()` arguments, literals in JSX, module lists that look like display text), blind spots restated.
- **A blocking check run after the step it protects.** Two cues written without their elision (`'assurance'` for
  the token `L'assurance`) were found when a stills render crashed on the first one. The cue test, run next,
  listed both at once. It should have run before the render.
- **The update command stopped halfway.** It printed each diff before writing the map; an arrow in the diff
  crashed the cp1252 console under a redirect, after some maps were written and before the others.
- **Forking made the first diff ~500 lines** (old scenes deleted, new ones added, in one change) and the informing
  test stayed red most of the session: the conditions for regenerating by reflex.
- **The episode's banned-word list changed on purpose.** "agile" had been banned (other episodes' subject) and
  became the topic; DevOps/SAFe vocabulary and framework names were added. Each pattern was tried on words it
  must catch and words it must pass; the final review still found one too wide (`trains?` would block
  "en train de").
- **The inventory caught a misplaced doc again**: one comment describing two geometry constants sat on the first
  only, leaving the second undocumented (the same failure as in episode 2).
- **The map as an arbiter.** AI video reviews reported typos that did not exist ("les essai", "Pas dans le
  suivante", "suivente-", a capital "Itération"); the screen-text map ruled out six claims and confirmed the real
  defects (an overflowing card, a "marchait" on screen where the voice said "fonctionnait").

**Episodes 4 to 7 of the same series (same day).** Each episode copied the previous one, then rewrote its frise.
- Running the blocking checks before the render (`npm run render` with a `prerender` step) held: no render was
  wasted on an unresolved cue across four episodes.
- Reading the inventory before each new frise decided reuse versus creation: the sprint loop extended the agile loop
  (same `LoopFrise`, new stations); the kanban board, the release train and the DevOps loop were new concepts, and
  each previous frise left the inventory in the same commit.
- **A stale map came with a copy.** Episode 7 was copied from episode 6 before episode 6's maps were regenerated: the
  copy's baseline maps described neither project, and its first diff showed a removal (`BoardFrise`) nobody made in it.
- **Labels kept in data stayed invisible.** Steps of a sequence held in an array inside a scene, and rows of a card
  passed as tuples in a prop, reached the screen but not the screen-text map, so the scope check never scanned them.
- **Fixes did not travel backward.** Two tool fixes made in later episodes (UTF-8 output of the review script, a
  trademark kept in its case in upper-cased tags) never reached the earlier episodes; no single repo's maps could show it.
- The map as an arbiter again: about half of the AI review findings were false (typos, hyphens, "missing" facts that
  were in the narration); the screen-text map and the voice transcript ruled each one out, and confirmed the real
  ones (a Definition of Done gate drawn after the sprint review, a marker looping through an 8-12 week interval).
