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

Write each map first, print its diff after, and force the output encoding (Python:
`sys.stdout.reconfigure(encoding="utf-8", errors="replace")`): a print that fails must not leave the maps half
updated.

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

## Repeated threshold by expression shape (fact-based)

Find every comparison and every `min`/`max` that contains a literal number, replace what varies with `_`, and
group by the resulting text. The same shape in two functions is almost always one concept written twice.

```python
class Shape(ast.NodeTransformer):
    def generic_visit(self, node):
        if isinstance(node, (ast.Name, ast.Attribute, ast.Subscript)):
            return ast.Name("_")
        if isinstance(node, ast.Call):
            keep = isinstance(node.func, ast.Name) and node.func.id in ("max", "min", "abs") and has_number(node)
            if not keep:
                return ast.Name("_")      # any call without a number, max(a, b) included, is just a value
            node.args = [self.visit(a) for a in node.args]
            return node                   # keep the function name: max(120, 0.1 * _)
        return super().generic_visit(node)

for node in ast.walk(module):
    if (isinstance(node, ast.Compare) or is_min_max(node)) and has_number(node):
        shapes[ast.unparse(Shape().visit(copy.deepcopy(node)))].add(enclosing_qualname(node))
report = {shape: where for shape, where in shapes.items() if len(where) > 1}
```

- `has_number`, `is_min_max` and `enclosing_qualname` are left to write (a literal number anywhere below the
  node; a call to `min`/`max`; the dotted name of the enclosing class and function).
- `ast.walk` also visits the `max` nested inside a comparison, so the same duplicate can be listed twice (as the
  comparison and as the `max`). Harmless; merge them if the noise bothers the reader.
- Without the "call without a number is a value" rule, `max(120, 0.1 * max(d1, d2))` and
  `max(120, 0.1 * duration)` get different shapes and the real duplicate is missed.
- Locate by enclosing function, not by line (see Generator requirements).
- Keep the weaker "same number in several functions" list below it, labelled as needing judgement.

## Contract keys without a key list (Python)

Writes: the string keys of the dict literals in the save function. Reads: in the load functions,
`var.get("k", ...)` and `var.get("k")` mark `k` optional; `var["k"]` marks it required unless a `.get` or an
`"k" in var` test of the same key exists (then the bare read is guarded). `Cls(**var)` makes every key required:
list it in the blind spots rather than guessing. The guard compares both against the released snapshot: a released key no
longer written or read is lost data; a key read as required that is not in the snapshot breaks old files.

## Decisions outside the pure layer (Tell Don't Ask, fact-based)

Count, in the glue only (not the pure rules layer, not diagnostics or the console), every relational comparison
(`<`, `<=`, `>`, `>=`, and relational patterns such as `{Loyalty: >= 50}`) where one side is a numeric literal and
the other READS A FACT OFF AN OBJECT: a member access, a conditional access, or a call to a method named `Get...`.

- Leave out, measured as noise: 0, 1 and -1 (emptiness and sign tests); numbers under 0.01 (epsilons); `Count`,
  `Length`, `Capacity` (list caps, truncation); comparisons inside a `for`/`while` condition.
- List by namespace as `Type.Member`: `expression`; ratchet the total.
- Blind spots for the header: a fact read into a local first (`var rel = ...; if (rel >= 30)`), named constants in
  the glue (the constants map has them), equality tests.

## Swap risk

For every public method, constructor and delegate of the pure layer, take the parameter shapes (modifier plus type,
whitespace removed: `outstring` differs from `string`) and the longest run of identical consecutive shapes. A run of
three or more is a call that can pass two arguments the wrong way round and still compile. Ratchet the count; the fix
is a type the caller already has, or a facts object with named, init-only properties.

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
| Decisions outside the pure layer (fact vs bare number in the glue) | a threshold re-decided in the glue, drifting from its twin |
| Swap risk (same-typed parameter runs) | two arguments passed the wrong way round, compiling |
