# Livewire round-trips: state that survives dehydration

The single most common way a Livewire feature ships broken while every test stays
green: the component is correct on the **first** render and wrong on **every render
after**, because the public property carrying the data did not survive the round-trip.

## The mechanism

Livewire dehydrates a public property holding Eloquent models or collections through a
model synthesizer. It persists **only the primary keys**, and rehydrates with
`Model::newQueryForRestoration($keys)` — which issues a bare
`select * from <table> where id in (...)` with no global scopes, no eager loads, and no
appended subselects.

Everything you attached at load time is therefore gone after the first interaction:

| Lost on round-trip | Why |
|---|---|
| `withCount('x as y')` | subselect, not a real column |
| `withSum()`, `withAggregate()`, `withMax()` | same |
| `with('relation')` / `withCount('rel as n')` | relation not eager-loaded on restore |
| computed `Attribute` appends, `select('… as alias')` | not persisted |

Anything you read off such a model afterwards is `null` / `0` / `''`. With a
`?? 0` fallback this does not error — it silently renders every row as empty.

## The rule

**Never carry derived or relation data on a public model property.** Keep it in a
scalar map keyed by id, and read that from the view:

```php
// public array $counts = [];   <-- plain JSON, survives hydration
public function loadData(): void
{
    $ids = $this->visible->pluck('id');
    $this->counts = Model::whereIn('id', $ids)
        ->selectRaw('id, count(*) as cnt')
        ->groupBy('id')
        ->pluck('cnt', 'id')
        ->toArray();
}
```

The alternative — re-applying the eager loads in every action that mutates the
property — is correct but easy to forget at the next call site; prefer the scalar map
for anything computed.

## Finding it by inspection

```bash
# public props that hold models/collections on the component
grep -nE 'public\s+(\??Collection|\\?[A-Z]\\\\?[A-Za-z]*\s+\$|\\?array\s+\$\w+)' path/to/component.blade.php

# then for each, ask what was attached at load time
grep -nE 'withCount|withSum|withAggregate|with\(|append\(' path/to/service.php path/to/component.blade.php
```

Any public prop holding a model that was populated via `withCount`/`with` is a finding.

## Proving it — the round-trip probe

The bug is invisible to `assertSee` on first mount. Force a round-trip and diff:

```php
$c = Livewire::test('comp', [...])->assertStatus(200);
$before = $c->html();
fwrite(STDERR, "[BEFORE] '{$expected} نفر' = " . (str_contains($before, "{$expected} نفر") ? 'YES' : 'NO') . "\n");

$c->call('toggle', (string) $someId);   // any action -> dehydrate + rehydrate

$after = $c->html();
fwrite(STDERR, "[AFTER ] '{$expected} نفر' = " . (str_contains($after, "{$expected} نفر") ? 'YES' : 'NO') . "\n");
```

**Compute the expected value from the database, never hardcode it.** A hardcoded
number that is off by one (a seeder or factory already created a row) makes the probe
report a clean bill of health for broken code. Print the DB count in the same run:

```php
fwrite(STDERR, "\n[DB] rows=" . (int) Model::where('fk', $id)->count() . "\n");
```

A `BEFORE = NO` is itself the tell that your probe is wrong, not the code.

## Probe fixture traps

- **Seeded fixtures are rarely empty.** `User::factory()` typically creates a linked
  person row, so a "fresh" unit is not 0. Measure first, then assert.
- **Deleting rows to reset a count hits foreign keys.** `Person::where('u_id', $id)->delete()`
  raises `users_n_code_fk` when a user references that person. Build a **dedicated
  subtree** with the exact count you want instead of mutating existing rows.
- Delete the probe file when done; it must not survive into the diff.

## Test shapes that hide the bug

| Shape | Why it passes against broken code |
|---|---|
| `assertSee('1 نفر')` with no `->call()` | only first render; round-trip never exercised |
| `toContainText('نفر')` | presence-only, passes with any number including `0 نفر` |
| assert a "negative" marker exists (e.g. `خالی` / "empty") | if the bug makes it appear on **every** row, the assertion is vacuous |
| `expect($n)->toBeGreaterThan(0)` on a page that always renders | never reaches the zero case |

Fixes: add a `->call(...)` round-trip before asserting; assert the **exact** rendered
value; for a negative marker assert it is present on the empty row **and absent** on
the populated row, or assert the total count of markers.
