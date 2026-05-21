# 24 — Hide verify once done + admin can revoke

> **Commit:** `e7d1061` — *feat(verify): hide button once verified + admin can revoke*

## Why this step?

Two real-world problems:

1. **The Verify button keeps showing after verification.** Click it twice → duplicate audit row, duplicate email to the student. Embarrassing.
2. **Admins occasionally need to undo a wrong verification.** Today they have to ask a developer to run an UPDATE. Give them a Revoke button instead.

We push both rules into the policy — not into Blade — so the UI never has to decide. The policy already drove the verify gate; we tighten it and add a symmetric `revoke`.

```php
verify  →  isAdmin && receipt !== null && verified_at === null
revoke  →  isAdmin                     && verified_at !== null
```

Mutually exclusive by construction: at most one button can render.

We also harden bulk verify so an already-verified student in a selection is silently skipped — no double-email, no duplicate audit.

**Why no PaymentRevoked email?** Revoking is an admin correction, not a customer-facing event. Telling the student "actually we un-verified you" creates more confusion than it solves. We log it; we don't broadcast it.

## What you'll build

- `StudentPolicy::verify` gains `&& $student->verified_at === null`
- New `StudentPolicy::revoke` method (admin + verified)
- New `StudentController::revoke()` action
- Route `PATCH /graduations/{g}/students/{s}/revoke`
- Blade renders Verify *or* Revoke, never both
- Bulk verify skips already-verified
- 6 new tests

## Prerequisites

- Completed [23 — Graduation search](./23-graduation-search.md).

## Steps

### 1. Tighten `StudentPolicy::verify` + add `revoke`

In `app/Policies/StudentPolicy.php`:

```php
public function verify(User $user, Student $student): bool
{
    return $user->isAdmin()
        && $student->payment_receipt !== null
        && $student->verified_at === null;
}

public function revoke(User $user, Student $student): bool
{
    return $user->isAdmin() && $student->verified_at !== null;
}
```

### 2. Add the revoke route

In `routes/web.php`, next to the verify route:

```php
Route::patch(
    'graduations/{graduation}/students/{student}/revoke',
    [StudentController::class, 'revoke']
)->name('graduations.students.revoke');
```

### 3. Add `revoke()` to `StudentController`

```php
public function revoke(Graduation $graduation, Student $student): RedirectResponse
{
    $this->authorize('revoke', $student);

    $student->update(['verified_at' => null]);

    Audit::record('revoked', $student);

    return redirect()
        ->route('graduations.students.show', [$graduation, $student])
        ->with('status', 'Verification revoked. Student moved back to Pending review.');
}
```

### 4. Harden the bulk-verify branch

In `StudentController@bulk`, tighten the verify filter:

```php
if ($action === 'verify') {
    $toVerify = (clone $scope)
        ->whereNotNull('payment_receipt')
        ->whereNull('verified_at')
        ->get();

    $skipped = (clone $scope)
        ->where(function ($q) {
            $q->whereNull('payment_receipt')
              ->orWhereNotNull('verified_at');
        })
        ->count();

    foreach ($toVerify as $student) {
        $student->update(['verified_at' => now()]);
        Audit::record('verified', $student, ['via' => 'bulk']);
        $student->notify(new PaymentVerified($student));
    }

    $tail = $toVerify->isNotEmpty() ? ' Email notifications sent.' : '';

    return redirect()->route('graduations.show', $graduation)
        ->with('status',
            "Verified {$toVerify->count()}. Skipped {$skipped} (no receipt or already verified).{$tail}");
}
```

### 5. Update `students/show.blade.php`

Replace the verify form block with this paired block. The Livewire starter kit uses Flux UI, not the Breeze `<x-primary-button>` component, so render the buttons as plain `<button>` tags styled with Tailwind utility classes — matching the rest of this codebase:

```blade
@can('verify', $student)
    <form method="POST"
          action="{{ route('graduations.students.verify', [$graduation, $student]) }}"
          class="mt-4">
        @csrf
        @method('PATCH')
        <button type="submit"
                class="inline-flex items-center rounded-md bg-emerald-600 px-3 py-1.5 text-sm font-medium text-white hover:bg-emerald-700">
            Verify payment
        </button>
    </form>
@endcan

@can('revoke', $student)
    <form method="POST"
          action="{{ route('graduations.students.revoke', [$graduation, $student]) }}"
          onsubmit="return confirm('Revoke verification for {{ $student->name }}?')"
          class="mt-4">
        @csrf
        @method('PATCH')
        <button type="submit"
                class="inline-flex items-center rounded-md bg-rose-600 px-3 py-1.5 text-sm font-medium text-white hover:bg-rose-700">
            Revoke verification
        </button>
    </form>
@endcan
```

`@can` enforces mutual exclusion: a verified student fails `verify` and passes `revoke`; an unverified student does the opposite.

> ⚠️ **Don't use `<x-primary-button>` here.** That component ships with the Breeze starter kit; this project uses the Livewire / Flux starter kit, which has no `<x-primary-button>` and will blow up with `Unable to locate a class or view for component [primary-button]`. Either use the inline button + Tailwind classes shown above, or switch to Flux primitives (`<flux:button variant="primary">…</flux:button>`).

### 6. Add six tests

```php
it('verify is denied for an already verified student', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();
    $student = Student::factory()->for($grad)->verified()->create();

    $this->actingAs($admin)
        ->patch(route('graduations.students.verify', [$grad, $student]))
        ->assertForbidden();
});

it('admin can revoke a verified student', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();
    $student = Student::factory()->for($grad)->verified()->create();

    $this->actingAs($admin)
        ->patch(route('graduations.students.revoke', [$grad, $student]))
        ->assertRedirect();

    expect($student->fresh()->verified_at)->toBeNull();
});

it('revoke writes an audit', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();
    $student = Student::factory()->for($grad)->verified()->create();

    $this->actingAs($admin)
        ->patch(route('graduations.students.revoke', [$grad, $student]));

    expect(App\Models\Audit::where('action', 'revoked')->count())->toBe(1);
});

it('revoke is denied for a never-verified student', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();
    $student = Student::factory()->for($grad)->create();

    $this->actingAs($admin)
        ->patch(route('graduations.students.revoke', [$grad, $student]))
        ->assertForbidden();
});

it('non-admin cannot revoke', function () {
    $user = User::factory()->create();
    $grad = Graduation::factory()->create();
    $student = Student::factory()->for($grad)->verified()->create();

    $this->actingAs($user)
        ->patch(route('graduations.students.revoke', [$grad, $student]))
        ->assertForbidden();
});

it('bulk verify skips already-verified students', function () {
    Notification::fake();

    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();
    $alreadyVerified = Student::factory()->for($grad)->verified()->create();
    $pending = Student::factory()->for($grad)->paidUnverified()->create();

    $this->actingAs($admin)
        ->post(route('graduations.students.bulk', $grad), [
            'action' => 'verify',
            'ids' => [$alreadyVerified->uuid, $pending->uuid],
        ]);

    Notification::assertSentTo($pending, App\Notifications\PaymentVerified::class);
    Notification::assertNotSentTo($alreadyVerified, App\Notifications\PaymentVerified::class);
    expect(App\Models\Audit::where('action', 'verified')->count())->toBe(1);
});
```

## Expected output

```
php artisan test
```

**76 tests / 180 assertions, all green.**

Browser:
1. Verify a pending student → button disappears, Revoke (red) button appears.
2. Click Revoke → confirm → student returns to Pending review, Verify button is back.
3. Bulk-verify a selection containing an already-verified student → flash: `Verified 2. Skipped 1 (no receipt or already verified). Email notifications sent.`

## Commit your work

```bash
git add .
git commit -m "feat(verify): hide button once verified + admin can revoke"
```

## Common pitfalls

- **Both buttons render at once** — `verify` policy isn't checking `verified_at === null`. Re-read step 1.
- **Bulk verify double-fires email** — your bulk filter still uses only `whereNotNull('payment_receipt')`. Add `whereNull('verified_at')`.
- **Revoke confirms but doesn't redirect** — the `revoke()` action returns void instead of `RedirectResponse`. Type the return value explicitly so it shows up next time.
- **Audit row reads `verified` instead of `revoked`** — copy-pasted from `verify()`. Always rename when copy-pasting handlers.
- **`Unable to locate class or view [primary-button]`** — you copied an older snippet that uses `<x-primary-button>` (a Breeze component). The Livewire starter kit doesn't ship it. Replace with a plain `<button>` + Tailwind classes (as shown in step 5) or with `<flux:button variant="primary">`.

## What's next

Privacy issue: non-admins still see the full students table on the graduation show page — other people's IC, email, matric numbers. Fix that in [25 — Restrict roster to admins](./25-restrict-roster.md).
