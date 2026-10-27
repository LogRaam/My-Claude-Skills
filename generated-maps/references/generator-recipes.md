# Generator recipes

Language-neutral patterns; examples in pseudo-C#/Python. Adapt names to the project.

## Layout

```
docs/maps/README.md            hand-written: one row per map ("read it to answer..."), which ones block
docs/maps/inventory.md         generated
docs/maps/dependencies.md      generated
docs/maps/<lens>.md            generated, one per bug family
docs/maps/api-released.md      written ONLY at release cut
tests/Maps/<Name>Map.cs        one generator per map: pure function source -> string
tests/Maps/MapsAreCurrentTests one test per map
tests/Maps/RatchetTests        baselines that may only shrink
tools/update-maps              regenerate + print what moved
```

## The compare test

```csharp
[Test] public void InventoryIsCurrent() => AssertMapCurrent("inventory.md", InventoryMap.Generate(SourceRoots));

static void AssertMapCurrent(string file, string generated)
{
    if (!SourceTreeComplete()) Assert.Ignore("partial copy of the sources: nothing to compare with");
    var path = Path.Combine(MapsDir, file);
    if (Env("UPDATE_MAPS") == "1") { File.WriteAllText(path, generated); return; }
    var committed = File.ReadAllText(path);
    Assert.That(Normalize(committed), Is.EqualTo(Normalize(generated)),
        $"{file} drifted. Read the diff, then run tools/update-maps and commit the map WITH the change.\n"
        + ShortDiff(committed, generated, maxLines: 40));
}
```

- `Normalize` only line endings. Anything else you normalize is something the map stops seeing.
- The failure message carries the diff, so a plain run is readable on its own.

## The update command

Sets the update flag, runs only the maps tests, then prints for each map: lines added/removed, and the full
diff when it is 40 lines or fewer. It must leave blocking failures red and say so.

## Contract guard (persisted keys, public API)

```csharp
var released = Parse("api-released.md");        // snapshot written at the last release cut
var now      = ApiMap.Members(SourceRoots);
var vanished = released.Except(now).ToList();   // a changed signature counts as vanished
if (vanished.Any() && Env("ALLOW_API_REMOVAL") != "1")
    Assert.Fail($"These public members shipped in {released.Version} and are no longer there: ...");
```

- Runs in update mode too. That is what makes it blocking.
- The snapshot is written only by an explicit release step (`update-maps --record-released-api`), never by a
  plain regeneration.
- Persisted keys: same shape against the committed save-format map, additive only.

## Ratchet

```csharp
const int CycleNamespacesBaseline = 46;     // lowered by hand, same commit as the improvement
[Test] public void CyclesDoNotGrow()
{
    var n = DependencyMap.NamespacesInCycles(SourceRoots).Count;
    Assert.That(n, Is.LessThanOrEqualTo(CycleNamespacesBaseline), "a namespace newly joined a cycle");
    if (n < CycleNamespacesBaseline) Assert.Fail($"now {n}: lower the baseline in this commit");
}
```

Forcing the baseline down when the count drops is deliberate: the improvement is locked in at once.

## Decision table

```python
for score in range(-100, 101):
    row = [rule(score) for rule in RULES_THAT_READ_SCORE]
```

Emit only the rows where some answer changes (the turning points) plus a summary. Last section: rules that
could be swept vs thresholds that exist; that gap measures how much logic is still compared inline.

## Rules index from test names

Turn every test name into a sentence (`Refuses_gift_when_regard_below_friend` -> "Refuses gift when regard below
friend"), grouped by fixture, with each fixture's doc comment as its reason for existing. Its diff at release
time is the best summary of what the release promises.

## Starter lenses worth checking for (only if the bug family exists)

| Lens | Bug family it answers |
|---|---|
| Constants, names declared in more than one type first | the same threshold minted in several places |
| Event listeners, with which of an event's reasons each one names | a listener filtering on one reason out of several |
| Registration / verb table, one column per place a feature must be declared | a feature declared in six places out of seven |
| Shared state (statics, caches, singletons) | a value captured once and read forever after |
| Prompt / pipeline sections in order, with cache zone | a fact placed where its consumer cannot see it |
| Gates: what opens an action vs what the executor demands | the two halves of one decision disagreeing |
