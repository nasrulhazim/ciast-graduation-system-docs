# 13 — Admin can bulk-import students from CSV

> **Commit:** `e6f25b9` — *feat(students): admin can bulk-import students from CSV*

## Why this step?

The university already has the student roster in a spreadsheet. Asking the admin to retype 300 rows is unrealistic. We let them upload a CSV with these headers:

```
name,ic,email,matric_card,phone
```

We use **`spatie/simple-excel`** as the CSV parser — same library mentioned in the production upgrade path of `build-spec.md`. It streams row-by-row so memory stays flat even for huge files.

Important design decision: **rows are validated individually, and invalid rows are silently skipped**. The controller redirects with a flash summary like `Imported 287. Skipped 13.` so the admin can fix the spreadsheet without re-uploading the whole batch. Throwing on the first bad row would be cruel UX.

## What you'll build

- Install `spatie/simple-excel`
- A new `POST /graduations/{graduation}/students/import` route (registered **before** the resource route)
- `StudentController@import` that parses, per-row-validates, and bulk-inserts the valid rows
- A small upload form on the graduation show page
- 3 tests

## Prerequisites

- Completed [12 — Student add/delete](./12-student-add-delete.md).

## Steps

### 1. Install Spatie Simple Excel

```bash
composer require spatie/simple-excel
```

### 2. Register the import route **before** the resource line

In `routes/web.php`:

```php
Route::post(
    'graduations/{graduation}/students/import',
    [StudentController::class, 'import']
)->name('graduations.students.import');

// The resource line comes AFTER, so /students/import isn't matched as /students/{student}
Route::resource('graduations.students', StudentController::class);
```

Order matters. Laravel matches routes top-to-bottom. If `Route::resource()` came first, `/students/import` would be matched as `show` with `import` interpreted as a UUID and 404.

### 3. Add the `import` method to `StudentController`

```php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Validator;
use Spatie\SimpleExcel\SimpleExcelReader;

public function import(Request $request, Graduation $graduation): RedirectResponse
{
    $this->authorize('create', Student::class);

    $request->validate([
        'csv' => ['required', 'file', 'mimes:csv,txt', 'max:2048'],
    ]);

    $imported = 0;
    $skipped = 0;

    SimpleExcelReader::create($request->file('csv')->getRealPath(), 'csv')
        ->getRows()
        ->each(function (array $row) use ($graduation, &$imported, &$skipped) {
            $validator = Validator::make($row, [
                'name' => ['required', 'string', 'max:255'],
                'ic' => ['required', 'string', 'size:12', 'unique:students,ic'],
                'email' => ['required', 'email', 'unique:students,email'],
                'matric_card' => ['required', 'string', 'max:100'],
                'phone' => ['required', 'string', 'max:20'],
            ]);

            if ($validator->fails()) {
                $skipped++;
                return;
            }

            $graduation->students()->create($validator->validated());
            $imported++;
        });

    return redirect()
        ->route('graduations.show', $graduation)
        ->with('status', "Imported {$imported}. Skipped {$skipped}.");
}
```

### 4. Add the upload form on the graduation show page

Near the **+ Add student** button:

```blade
 @can('create', App\Models\Student::class)
    <div class="flex flex-wrap items-center gap-2">
        <form method="POST"
                action="{{ route('graduations.students.import', $graduation) }}"
                enctype="multipart/form-data"
                class="inline-flex items-center gap-2">
            @csrf
            <input type="file" name="csv" accept=".csv,text/csv" required
                    class="block text-sm text-gray-700 file:mr-3 file:rounded-md file:border-0 file:bg-slate-100 file:px-3 file:py-1.5 file:text-sm file:font-medium file:text-slate-700 hover:file:bg-slate-200 dark:text-neutral-300 dark:file:bg-neutral-800 dark:file:text-neutral-200" />
            <button type="submit"
                    class="inline-flex items-center rounded-md bg-slate-600 px-3 py-1.5 text-sm font-medium text-white hover:bg-slate-700">
                Import CSV
            </button>
        </form>

        <a href="{{ route('graduations.students.create', $graduation) }}"
            class="inline-flex items-center rounded-md bg-indigo-600 px-3 py-1.5 text-sm font-medium text-white hover:bg-indigo-700">
            + Add student
        </a>
    </div>
@endcan
```

### 5. Sample CSV (for manual testing)

Save this as `~/Desktop/students-sample.csv`:

```csv
name,ic,email,matric_card,phone
Aiman Yusof,900101011234,aiman@example.test,M100001,0123456789
Siti Nurhaliza,910202022345,siti@example.test,M100002,0129876543
Ali Hassan,!!!,ali@example.test,M100003,0123344556
```

Row 3 has an invalid IC (`size:12` fails) — it should be reported as skipped.

### 6. Add three tests

```php
it('admin can bulk import students from CSV', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();

    $csv = "name,ic,email,matric_card,phone\n"
        . "A,900101011111,a@x.test,M1,012\n"
        . "B,900101012222,b@x.test,M2,013\n";

    $file = UploadedFile::fake()->createWithContent('roster.csv', $csv);

    $this->actingAs($admin)
        ->post(route('graduations.students.import', $grad), ['csv' => $file])
        ->assertRedirect();

    expect($grad->students()->count())->toBe(2);
});

it('skips invalid rows and reports a count', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();

    // existing student so the duplicate row fails unique check
    Student::factory()->for($grad)->create([
        'ic' => '900101019999',
        'email' => 'dup@x.test',
    ]);

    $csv = "name,ic,email,matric_card,phone\n"
        . "Good,900101011111,good@x.test,M1,012\n"
        . "BadIC,!!!,bad@x.test,M2,012\n"
        . "Dup,900101019999,dup@x.test,M3,012\n";

    $file = UploadedFile::fake()->createWithContent('roster.csv', $csv);

    $this->actingAs($admin)
        ->post(route('graduations.students.import', $grad), ['csv' => $file])
        ->assertRedirect()
        ->assertSessionHas('status', 'Imported 1. Skipped 2.');
});

it('non-admin cannot import', function () {
    $user = User::factory()->create();
    $grad = Graduation::factory()->create();

    $file = UploadedFile::fake()->createWithContent('roster.csv', "name\nx\n");

    $this->actingAs($user)
        ->post(route('graduations.students.import', $grad), ['csv' => $file])
        ->assertForbidden();
});
```

## Expected output

```
php artisan test
```

**41 tests / 89 assertions, all green.**

Browser flow:
1. Log in as admin → click a graduation.
2. Choose `students-sample.csv` → click **Import CSV**.
3. Flash message: `Imported 2. Skipped 1.`
4. The table now shows the two valid students.

## Commit your work

```bash
git add .
git commit -m "feat(students): admin can bulk-import students from CSV"
```

## Common pitfalls

- **Route collision**: if the import POST 404s and you see "No query results for model [Student]", you put the route *after* the resource line. Swap the order.
- **All rows skipped** — your CSV's first line isn't the header. `SimpleExcelReader` treats row 1 as headers; if your file starts with data, you'll get empty `$row` keys.
- **Memory blow-up on huge CSVs** — only happens if you don't use `getRows()->each(...)` (a lazy collection). `->toArray()` would load everything; avoid it.
- **Unicode names corrupted** — your CSV is in CP1252, not UTF-8. Re-export from Excel as "CSV UTF-8".

## What's next

Importing a roster of 500 students makes the table unscrollable. We need pagination and search: [14 — Search + pagination](./14-search-pagination.md).
