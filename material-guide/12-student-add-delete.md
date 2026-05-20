# 12 — Full CRUD: admin can add + delete students

> **Commit:** `3d5d084` — *feat(students): full CRUD — admin can create + delete students*

## Why this step?

Up to now, students arrive in the database only via seeders. In the real workflow the admin needs to add a single student through the UI (e.g. a late registration the bulk import missed), and remove rows for typos or no-shows.

We open up the nested resource: drop `->only(['show', 'edit', 'update'])`, fill in the `StoreStudentRequest`, add a `create.blade.php`, and hang an `+ Add student` button + per-row `Remove` button off the graduation show page — both `@can`-gated so only admins see them.

## What you'll build

- Routes: `GET /graduations/{graduation}/students/create`, `POST .../students`, `DELETE .../students/{student}`
- `StoreStudentRequest` with rules + authorization
- `StudentController@create`, `@store`, `@destroy`
- `students/create.blade.php` (same form as edit, minus receipt field)
- `+ Add student` and per-row `Remove` buttons on `graduations/show.blade.php`
- 4 new tests

## Prerequisites

- Completed [11 — Pest tests](./11-pest-tests.md).

## Steps

### 1. Open up the nested resource routes

In `routes/web.php`, remove `->only([...])`:

```php
// Before
Route::resource('graduations.students', StudentController::class)
    ->only(['show', 'edit', 'update']);

// After
Route::resource('graduations.students', StudentController::class);
```

### 2. Fill in `StoreStudentRequest`

```php
<?php

declare(strict_types=1);

namespace App\Http\Requests;

use App\Models\Student;
use Illuminate\Foundation\Http\FormRequest;

class StoreStudentRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('create', Student::class);
    }

    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'ic' => ['required', 'string', 'size:12', 'unique:students,ic'],
            'email' => ['required', 'email', 'unique:students,email'],
            'matric_card' => ['required', 'string', 'max:100'],
            'phone' => ['required', 'string', 'max:20'],
        ];
    }
}
```

### 3. Add `create`, `store`, `destroy` to `StudentController`

```php
use App\Http\Requests\StoreStudentRequest;

public function create(Graduation $graduation): View
{
    $this->authorize('create', Student::class);

    return view('students.create', compact('graduation'));
}

public function store(StoreStudentRequest $request, Graduation $graduation): RedirectResponse
{
    $student = $graduation->students()->create($request->validated());

    return redirect()
        ->route('graduations.show', $graduation)
        ->with('status', "Added {$student->name}.");
}

public function destroy(Graduation $graduation, Student $student): RedirectResponse
{
    $this->authorize('delete', $student);

    $student->delete();

    return redirect()
        ->route('graduations.show', $graduation)
        ->with('status', 'Student removed.');
}
```

### 4. `students/create.blade.php`

Copy `students/edit.blade.php` and:

- Change action to `route('graduations.students.store', $graduation)`
- Remove `@method('PATCH')`
- Remove the `payment_receipt` field (admins attach receipts later via the edit page)
- Replace `old('name', $student->name)` with `old('name')` everywhere. Do the same with other inputs that using `$student`

### 5. Update `graduations/show.blade.php`

Add an action button next to the students table header:

```blade
<div class="flex justify-between items-center mb-4">
    <h3 class="text-lg font-semibold">Students ({{ $graduation->students->count() }})</h3>
    @can('create', App\Models\Student::class)
        <a href="{{ route('graduations.students.create', $graduation) }}"
           class="bg-indigo-600 text-white px-3 py-2 rounded text-sm">
            + Add student
        </a>
    @endcan
</div>
```

And inside each student row:

```blade
<td class="px-6 py-4 text-right">
    <a href="{{ route('graduations.students.show', [$graduation, $student]) }}"
       class="text-indigo-600 mr-3">View</a>

    @can('delete', $student)
        <form method="POST"
              action="{{ route('graduations.students.destroy', [$graduation, $student]) }}"
              onsubmit="return confirm('Remove {{ $student->name }}?')"
              class="inline">
            @csrf @method('DELETE')
            <button class="text-red-600">Remove</button>
        </form>
    @endcan
</td>
```

### 6. Add four tests

Append to `tests/Feature/GraduationFlowTest.php`:

```php
it('admin can add a student to a graduation', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();

    $this->actingAs($admin)
        ->post(route('graduations.students.store', $grad), [
            'name' => 'Ali Bin Ahmad',
            'ic' => '900101011234',
            'email' => 'ali@example.test',
            'matric_card' => 'M123456',
            'phone' => '0123456789',
        ])
        ->assertRedirect();

    expect($grad->students()->where('ic', '900101011234')->exists())->toBeTrue();
});

it('non-admin cannot add a student', function () {
    $user = User::factory()->create();
    $grad = Graduation::factory()->create();

    $this->actingAs($user)
        ->post(route('graduations.students.store', $grad), [
            'name' => 'Foo', 'ic' => '900101011235', 'email' => 'foo@x.test',
            'matric_card' => 'M1', 'phone' => '012',
        ])
        ->assertForbidden();
});

it('admin can delete a student', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();
    $student = Student::factory()->for($grad)->create();

    $this->actingAs($admin)
        ->delete(route('graduations.students.destroy', [$grad, $student]))
        ->assertRedirect();

    expect(Student::find($student->id))->toBeNull();
});

it('non-admin cannot delete a student', function () {
    $user = User::factory()->create();
    $grad = Graduation::factory()->create();
    $student = Student::factory()->for($grad)->create();

    $this->actingAs($user)
        ->delete(route('graduations.students.destroy', [$grad, $student]))
        ->assertForbidden();
});
```

## Expected output

```
php artisan test
```

…shows **38 tests / 84 assertions, all green**.

In the browser as admin: visit a graduation → click **+ Add student** → submit the form → see the new row in the table. Hover a row, click **Remove**, confirm → row disappears.

## Commit your work

```bash
git add .
git commit -m "feat(students): full CRUD — admin can create + delete students"
```

## Common pitfalls

- **Cannot submit `+ Add student` form** because the `ic` field rejects values not exactly 12 chars. Malaysian IC is 12 digits — `size:12` enforces it.
- **Duplicate `email` / `ic`** — the unique rule kicks in if you tried to add a row that already exists (incl. soft-deleted students; but Student is hard-deleted so this rarely bites here).
- **`Remove` button submits multiple times** — `onsubmit="return confirm(...)"` only fires once; multiple submits usually mean nested `<form>` elements. Move the form out of any wrapping form tag.

## What's next

Adding one student at a time is fine, but a real graduation has hundreds. Time to import a CSV roster: [13 — CSV import](./13-csv-import.md).
