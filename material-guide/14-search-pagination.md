# 14 — Searchable + paginated students table

> **Commit:** `251fa62` — *feat(students): searchable + paginated students table on graduation page*

## Why this step?

After importing 500 students, the graduation show page renders an unscrollable table. Two fixes, one commit:

1. **Pagination** — 15 per page, "newest first" by default
2. **Search** — admin types a name / IC / email / matric → OR'd `LIKE` query

`withQueryString()` on the paginator preserves `?search=` across page links, so clicking page 2 doesn't lose your filter.

## What you'll build

- `GraduationController@show` now receives `Request $request` and runs a paginated, filtered query off the students relation
- The Blade table swaps from a relation collection to a paginator
- A GET search form sits above the table, with a Clear link when active
- 3 tests

## Prerequisites

- Completed [13 — CSV import](./13-csv-import.md).

## Steps

### 1. Update `GraduationController@show`

```php
use Illuminate\Http\Request;

public function show(Graduation $graduation, Request $request): View
{
    $this->authorize('view', $graduation);

    $students = $graduation->students()
        ->when($request->filled('search'), function ($q) use ($request) {
            $term = '%' . $request->string('search')->trim() . '%';
            $q->where(function ($inner) use ($term) {
                $inner->where('name', 'like', $term)
                    ->orWhere('ic', 'like', $term)
                    ->orWhere('email', 'like', $term)
                    ->orWhere('matric_card', 'like', $term);
            });
        })
        ->latest()
        ->paginate(15)
        ->withQueryString();

    return view('graduations.show', compact('graduation', 'students'));
}
```

The inner `$q->where(function ($inner) ...)` block groups the OR clauses so they don't accidentally fan out across other future `where()`s.

### 2. Update `graduations/show.blade.php`

Three things to swap from the relation (`$graduation->students`) over to the paginator (`$students`):

1. The **count** in the section header
2. The **`@forelse` loop**
3. A new **pagination links** block below the table

```blade
{{-- header count: was {{ $graduation->students->count() }} --}}
<h3 class="text-sm font-semibold text-gray-900 dark:text-neutral-100">
    Students ({{ $students->total() }})
</h3>

<div class="flex justify-between items-center mb-4 gap-4">
    <form method="GET" action="{{ route('graduations.show', $graduation) }}"
          class="flex gap-2 flex-1">
        <input type="text" name="search" value="{{ request('search') }}"
               placeholder="Search name, IC, email, matric…"
               class="rounded border-gray-300 text-sm w-full max-w-sm">
        <button class="bg-slate-600 text-white px-3 py-2 rounded text-sm">Search</button>

        @if (request('search'))
            <a href="{{ route('graduations.show', $graduation) }}"
               class="text-sm text-slate-600 self-center">Clear</a>
        @endif
    </form>

    {{-- existing + Add student / Import CSV buttons --}}
</div>

<table class="w-full">
    {{-- ... thead ... --}}
    <tbody>
        @forelse ($students as $student)
            <tr>{{-- ... cells ... --}}</tr>
        @empty
            <tr><td colspan="..." class="px-6 py-12 text-center text-gray-500">
                @if (request('search'))
                    No students match "{{ request('search') }}".
                @else
                    No students yet.
                @endif
            </td></tr>
        @endforelse
    </tbody>
</table>

<div class="mt-4">{{ $students->links() }}</div>
```

The empty state contextualises whether it's a zero-match (search returned nothing) or a truly empty roster. Use `$students->total()` (not `count()`) on a paginator — `count()` only returns the rows on the current page.

### 3. Add three tests

```php
it('filters students table by name', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();

    Student::factory()->for($grad)->create(['name' => 'Alice Tan']);
    Student::factory()->for($grad)->create(['name' => 'Bob Lim']);

    $this->actingAs($admin)
        ->get(route('graduations.show', ['graduation' => $grad, 'search' => 'Alice']))
        ->assertOk()
        ->assertSee('Alice Tan')
        ->assertDontSee('Bob Lim');
});

it('filters by matric or ic', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();

    Student::factory()->for($grad)->create(['matric_card' => 'M111111']);
    Student::factory()->for($grad)->create(['matric_card' => 'M222222']);

    $this->actingAs($admin)
        ->get(route('graduations.show', ['graduation' => $grad, 'search' => 'M11']))
        ->assertSee('M111111')
        ->assertDontSee('M222222');
});

it('paginates with more than 15 students', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();
    Student::factory()->count(20)->for($grad)->create();

    $this->actingAs($admin)
        ->get(route('graduations.show', $grad))
        ->assertOk()
        ->assertSee('page=2');     // pagination link present
});
```

## Expected output

```
php artisan test
```

**44 tests / 96 assertions, all green.**

Browser:
- Visit a graduation with 20+ students → see only 15 rows + a `1 2 ›` pagination control at the bottom.
- Type a name fragment in the search box → results filter; URL becomes `...?search=ali`.
- Click "page 2" → URL keeps `?search=ali&page=2`.
- Click **Clear** → search disappears, all rows return.

## Commit your work

```bash
git add .
git commit -m "feat(students): searchable + paginated students table on graduation page"
```

## Common pitfalls

- **Blade still loops `$graduation->students`** — the relationship returns *all* rows, bypassing your search and pagination. The query in the controller runs, but the view never uses its result. Symptom: typing in the search box does nothing, and the table never paginates. Fix: loop `$students` (the paginator), use `$students->total()` for the count, and render `{{ $students->links() }}`.
- **`like` is case-sensitive on PostgreSQL** but not on SQLite or MySQL. If you migrate to Postgres later, switch to `ilike`.
- **`->withQueryString()` missing** — pagination links drop the search term. Easy to fix; easy to forget.
- **`request()->string('search')->trim()`** returns a `Stringable`, but string concat with `'%' . ...` calls `__toString()`. If you see `Object of class Stringable could not be converted to string`, your PHP version is wrong — needs 8.0+.

## What's next

We can search. Next: filter by payment status — admins want to see "who hasn't paid yet" in one click: [15 — Status filter](./15-status-filter.md).
