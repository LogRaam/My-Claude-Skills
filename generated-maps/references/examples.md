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
