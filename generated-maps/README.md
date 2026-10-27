# Generated maps

**A way to keep a codebase understandable when an AI writes most of it.** This folder is a skill an AI agent
loads (`SKILL.md`); this README is for the humans who want to understand the idea before trusting it.

## The problem

When most of the code is written by a model, the first thing to rot is not code quality. It is knowing what
already exists.

A real example from the project this comes from. The question "how friendly does a character have to be to
count as a friend?" had five answers in the code:

```
FriendThreshold = 10            (a private constant, in three separate files)
BestFriendThreshold = 20        (a fourth file)
FriendRelationThreshold = 20    (a fifth file)
```

Every class was clean. Nobody made a mistake. The concept was scattered, and changing "what a friend is"
meant finding five files and deciding which of them meant the same thing.

This gets worse with an AI in the loop, because of an asymmetry:

- **You** have a partial but persistent memory. You remember having seen something like this three weeks ago,
  and you can go looking.
- **The model** has no memory between sessions. It knows only what it reads this time. It *could* search for
  the existing concept, but nothing prompts it to, so it writes a new one, confidently, because its solution is
  locally correct.

Typing `x >= 20` by hand costs a few seconds, and in those seconds a person wonders whether the rule already
exists. Generating it costs nothing, and the question never comes up.

## Why the usual answers fall short

| Usual answer | Why it is not enough |
|---|---|
| A rules file for the AI ("always reuse existing thresholds") | Advice, not a constraint. It only works if the model reads it at the right moment, and it ages: the project's instructions file listed 16 prompt sections when the code had 108. |
| Clean code, SOLID | Helps each class locally. All five classes above were well formed. The duplication was conceptual. |
| A hand-written architecture document | Rots silently and is still trusted. |
| A generated document | Better, but only if something forces the regeneration. The project had one; it was ten days and about 600 rules behind, because nothing made anyone regenerate it. |

## What a map is

A map is **a view of the system generated from the source code, committed to the repository as an ordinary
file, and checked by a test that regenerates it and fails when the committed copy is different.**

Three parts, and the third is the one that matters:

1. **A generator**: a small program that reads the source and writes a Markdown file.
2. **The committed map**: that Markdown file, in `docs/maps/`, reviewed like code.
3. **The test**: it regenerates the map and compares it with the committed copy. If the code moved and the map
   did not, the suite goes red.

The test is the trigger. Without it you have written one more document that will rot. With it, the map is
always as true as the code, because nothing gets committed until the suite is green.

### What one looks like

An excerpt of a real constants map. It opens with the names declared in more than one place, which is exactly
the duplication the example above was about:

```markdown
# Constants: every number the system is tuned by

GENERATED. Do not edit. A test fails when this file no longer matches the code.

1464 numeric constants in 425 types. Look here BEFORE adding a threshold.

## One name, several homes

| Name | Where, and what it is worth there |
|---|---|
| `ArrivalDistance` | `AmbushBehavior` = 2.5f; `MissionBehavior` = 2.0f; `PatrolBehavior` = 2.0f; `CourierService` = 2.0f |
| `BaseDecayPerDay` | `AppreciationPolicy` = 0.2; `GrudgePolicy` = 0.4 |
| `CooldownDays` | `EncounterService` = 1; `AmbitionPolicy` = 30; `DuelPolicy` = 7 |
```

Some of those rows are one concept that should have one home. Some are two honest questions that happen to
share a word. Either way, it is now a decision someone can see, instead of an accident nobody can.

## Why it helps

**The model reads the map before it writes.** That is the main use. The instruction "check the constants map
before adding a threshold" is still just advice, but it points at something the test keeps true. An old rules
document was advice pointing at something stale; the same sentence pointing at a tested map is worth a lot
more. The skill also makes the agent write in the commit message what it found ("no existing threshold for
this"), so a skipped look shows up in review.

**Every change gets two diffs.** The code diff says what was edited. The map diff says what it meant for the
structure: a new dependency between two modules, a second constant with an existing name, a feature registered
in six places out of the seven it needs. For a reviewer, the map diff is often the only one worth reading, and
it is much shorter.

**An unchanged map is information too.** If the dependencies map did not move, the change added no coupling.

**It caught real bugs on day one**, just from reading the first maps: a feature declared in six places out of
seven and so unreachable for months; a check reading the wrong one of two scores that share the same range;
a payment ceiling defined in five places, where changing one silently broke another; a dependency cycle
spanning 37 namespaces.

## Two kinds of test: informing and blocking

This distinction is what keeps the method from turning into paperwork.

**Informing maps** (inventory, dependencies, constants...) accept whatever the code now says. They fail, you
read the diff, you regenerate, you commit. Their value is the attention the diff buys, and that value decays:
within weeks "regenerate and commit" becomes a reflex. Plan for that.

**Blocking tests** refuse to absorb a change just because the code made it:

- **Contracts**: promises to people outside the project. A key in users' saved data that disappears means
  their data is silently never read again. A public API member that disappears breaks somebody else's code.
  These are compared with what was *released*, not with the last commit, and only an explicit override (with a
  reason in the commit) lets one go.
- **Ratchets**: a number that may only get better. "The set of modules caught in a dependency cycle may not
  grow." "The count of thresholds written as bare numbers may only go down." A ratchet needs no human reader at
  all, which is why it is the best answer to review fatigue. When the count drops, the baseline is lowered in
  the same commit, so the improvement is locked in.

A third kind sits between: a **decision table**. Take a value the system decides by (a score from -100 to
+100), call every rule that depends on it at every step, and commit the table of answers. When you refactor,
the expected diff is *nothing*. That turns "I only moved code around" from a promise into something the test
checks.

## What maps do not solve

- **The same concept under two different names.** `HostileThreshold = -30` and `EnemyLimit = -25` both pass
  every check. The map puts them near each other; a person (or an agent told to look) decides they are one.
- **A behaviour that changed with no test changing.** No structural map sees it. That is a missing test.
- **The same model writing the code and its tests.** The tests tend to confirm what the code does, not what it
  should do. What helps is external: real bug reports turned into tests, and a human reading the diffs.
- **Duplication itself.** Maps make it visible. A dedicated *type* (instead of a bare `int`) makes it
  impossible to write `x >= 20` somewhere else at all. Maps and types work together: the map shows where to
  start, a ratchet stops it from getting worse while you migrate.
- **A map can be wrong.** One that finds things by name gives false positives when a name misleads it, and
  silently misses things that have no telltale name. Good maps pair name-based detection with reading what the
  code actually does, and say on each line which one found it.

## What it costs

On the original project (two repositories, about 1,400 types, 6,300 tests): eleven generators at first, about a
thousand lines, written by the AI itself in a day. The test suite went from about one second to seven. Each
structural change carries one more step, reading the map diff, and that friction is the point.

## How to start

Smallest useful version, in order. Each step is worth having alone.

1. **One map**: every type by module, with its one-line summary. Ask your AI to write the generator; it is an
   afternoon. Generators should read the source as text (a syntax parser such as Roslyn, tree-sitter or Python's
   `ast`), so they need no build and keep the doc comments.
2. **One test** that regenerates it and compares. This is the step that makes it real.
3. **One command** that regenerates and prints the diff of what moved, not just a count.
4. **Your contracts**: saved data, public API, wire formats. Promises your tests probably do not check today.
5. **Then one map per kind of bug you have actually had twice.** Look at your last twenty bug reports and
   group them. Your list will not be ours.

Or install the skill (see the [repository README](../README.md)) and ask your agent to "set up generated maps
on this project": it follows that order.

## Using the skill

Once installed, the agent uses it on its own: before adding a rule, a threshold or a public member, it reads
the matching map; after the change, it runs the map tests in compare mode *before* regenerating (regenerating
first makes every map compare with itself and pass, which hides the drift), reads what moved, then commits the
maps with the change.

Your part as the human: read the map diffs at review time, and treat a blocking failure as a question to
answer, not a test to make green.

## Further reading

- [`SKILL.md`](SKILL.md): what the agent is taught.
- [`references/generator-recipes.md`](references/generator-recipes.md): code patterns for the test, the guard,
  the ratchet and the decision table.
- [`references/examples.md`](references/examples.md): one real case per mechanism.
- The idea has a name in the literature: **architecture fitness functions**. Off-the-shelf tools cover parts
  of it (for .NET: `PublicApiAnalyzers` for the API contract, ArchUnitNET or NDepend for dependency rules,
  Verify for approval tests). Use them where they fit; the maps approach is for what they cannot see, such as
  code that cannot be compiled in the test context, or questions specific to your project.
