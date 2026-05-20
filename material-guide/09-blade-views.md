# 09 — Blade views + nav link

> **Commit:** `dbbcbe0` — *feat(ui): add graduation + student Blade views + nav link*

## Why this step?

We have routes, controllers, and authorization. We're one piece away from a working app: the actual HTML.

Six Blade views — three for graduations, two for students — that all extend Breeze's `<x-app-layout>` and reuse Breeze's existing form components (`x-input-label`, `x-text-input`, `x-primary-button`, `x-input-error`).

Two patterns to internalize from this step:

1. **`@can('ability', $model)` in Blade** — hide UI affordances students would 403 on anyway. The button never renders for non-admins, so it's not "client-side security theatre" — it's a UX thing.
2. **File-upload forms need `enctype="multipart/form-data"`** — without it, the file disappears between browser and Laravel.

## What you'll build

- `resources/views/graduations/index.blade.php` — paginated list with status badges
- `resources/views/graduations/create.blade.php` — form (admin only)
- `resources/views/graduations/edit.blade.php` — form + archive button
- `resources/views/graduations/show.blade.php` — detail + nested student table
- `resources/views/students/show.blade.php` — detail + verify button
- `resources/views/students/edit.blade.php` — form with file upload
- Updated `resources/views/layouts/app/sidebar.blade.php` — adds "Graduations" link to the Flux sidebar (works for both desktop and the collapsed mobile menu)

## Prerequisites

- Completed [08 — Routes](./08-routes.md).

## Steps

### 1. Add the navigation link

This project uses the **Livewire starter kit** with **Flux UI**, not Breeze. The nav lives in a single Flux sidebar component that handles both the desktop sidebar and the mobile collapsed menu — no separate desktop/mobile blocks to keep in sync.

Open `resources/views/layouts/app/sidebar.blade.php` and find the existing **Platform** group containing the Dashboard item:

```blade
<flux:sidebar.nav>
    <flux:sidebar.group :heading="__('Platform')" class="grid">
        <flux:sidebar.item icon="home" :href="route('dashboard')" :current="request()->routeIs('dashboard')" wire:navigate>
            {{ __('Dashboard') }}
        </flux:sidebar.item>
    </flux:sidebar.group>
</flux:sidebar.nav>
```

Add a sibling `flux:sidebar.item` for graduations:

```blade
<flux:sidebar.item icon="academic-cap" :href="route('graduations.index')" :current="request()->routeIs('graduations.*')" wire:navigate>
    {{ __('Graduations') }}
</flux:sidebar.item>
```

The `:current="request()->routeIs('graduations.*')"` highlights the link on any `graduations.index|create|show|edit` page. `wire:navigate` keeps navigation SPA-fast via Livewire's [navigate](https://livewire.laravel.com/docs/navigate) feature.

> If you started from a Breeze-based starter instead, your equivalent file is `resources/views/layouts/navigation.blade.php` — add an `<x-nav-link>` (and its `<x-responsive-nav-link>` mobile twin) pointing at `route('graduations.index')`.

### 2. `graduations/index.blade.php`

```blade
<x-app-layout>
    <x-slot name="header">
        <div class="flex justify-between items-center">
            <h2 class="font-semibold text-xl text-gray-800 leading-tight">
                {{ __('Graduations') }}
            </h2>
            @can('create', App\Models\Graduation::class)
                <a href="{{ route('graduations.create') }}"
                   class="bg-indigo-600 text-white px-4 py-2 rounded">
                    + New graduation
                </a>
            @endcan
        </div>
    </x-slot>

    <div class="py-12">
        <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
            @if (session('status'))
                <div class="mb-4 p-3 bg-green-100 text-green-800 rounded">
                    {{ session('status') }}
                </div>
            @endif

            <div class="bg-white shadow rounded">
                <table class="w-full">
                    <thead class="bg-gray-50">
                        <tr>
                            <th class="px-6 py-3 text-left">Title</th>
                            <th class="px-6 py-3 text-left">Date</th>
                            <th class="px-6 py-3 text-left">Fee</th>
                            <th class="px-6 py-3 text-left">Status</th>
                            <th class="px-6 py-3 text-left">Students</th>
                            <th class="px-6 py-3"></th>
                        </tr>
                    </thead>
                    <tbody>
                        @forelse ($graduations as $g)
                            @php
                                $colors = [
                                    'draft'  => 'bg-slate-100 text-slate-700',
                                    'open'   => 'bg-green-100 text-green-700',
                                    'closed' => 'bg-amber-100 text-amber-700',
                                ];
                            @endphp
                            <tr class="border-t">
                                <td class="px-6 py-4">{{ $g->title }}</td>
                                <td class="px-6 py-4">{{ $g->ceremony_date->format('d M Y') }}</td>
                                <td class="px-6 py-4">RM {{ number_format((float) $g->fee, 2) }}</td>
                                <td class="px-6 py-4">
                                    <span class="px-2 py-1 text-xs rounded {{ $colors[$g->status] }}">
                                        {{ ucfirst($g->status) }}
                                    </span>
                                </td>
                                <td class="px-6 py-4">{{ $g->students_count }}</td>
                                <td class="px-6 py-4 text-right">
                                    <a href="{{ route('graduations.show', $g) }}"
                                       class="text-indigo-600">View</a>
                                </td>
                            </tr>
                        @empty
                            <tr><td colspan="6" class="px-6 py-12 text-center text-gray-500">
                                No graduations yet.
                            </td></tr>
                        @endforelse
                    </tbody>
                </table>
            </div>

            <div class="mt-4">{{ $graduations->links() }}</div>
        </div>
    </div>
</x-app-layout>
```

### 3. `graduations/create.blade.php`

```blade
<x-app-layout>
    <x-slot name="header">
        <h2 class="font-semibold text-xl text-gray-800 leading-tight">New Graduation</h2>
    </x-slot>

    <div class="py-12 max-w-3xl mx-auto sm:px-6 lg:px-8">
        <div class="bg-white shadow rounded p-6">
            <form method="POST" action="{{ route('graduations.store') }}" class="space-y-4">
                @csrf

                <div>
                    <x-input-label for="title" :value="__('Title')" />
                    <x-text-input id="title" name="title" type="text"
                                  value="{{ old('title') }}" class="block mt-1 w-full" />
                    <x-input-error :messages="$errors->get('title')" class="mt-2" />
                </div>

                <div>
                    <x-input-label for="ceremony_date" :value="__('Ceremony date')" />
                    <x-text-input id="ceremony_date" name="ceremony_date" type="date"
                                  value="{{ old('ceremony_date') }}" class="block mt-1 w-full" />
                    <x-input-error :messages="$errors->get('ceremony_date')" class="mt-2" />
                </div>

                <div>
                    <x-input-label for="fee" :value="__('Fee (RM)')" />
                    <x-text-input id="fee" name="fee" type="number" step="0.01" min="0"
                                  value="{{ old('fee') }}" class="block mt-1 w-full" />
                    <x-input-error :messages="$errors->get('fee')" class="mt-2" />
                </div>

                <div>
                    <x-input-label for="status" :value="__('Status')" />
                    <select name="status" id="status" class="block mt-1 w-full rounded border-gray-300">
                        @foreach (['draft', 'open', 'closed'] as $s)
                            <option value="{{ $s }}" @selected(old('status') === $s)>
                                {{ ucfirst($s) }}
                            </option>
                        @endforeach
                    </select>
                    <x-input-error :messages="$errors->get('status')" class="mt-2" />
                </div>

                <x-primary-button>Save</x-primary-button>
            </form>
        </div>
    </div>
</x-app-layout>
```

### 4. `graduations/edit.blade.php`

Mirrors `create` but adds `@method('PATCH')` and a danger-zone block:

```blade
<form method="POST" action="{{ route('graduations.update', $graduation) }}" class="space-y-4">
    @csrf @method('PATCH')
    {{-- same fields as create.blade.php, but with old('title', $graduation->title) etc. --}}
    <x-primary-button>Update</x-primary-button>
</form>

@can('delete', $graduation)
    <form method="POST" action="{{ route('graduations.destroy', $graduation) }}"
          onsubmit="return confirm('Archive this graduation?')"
          class="mt-8 pt-6 border-t border-red-200">
        @csrf @method('DELETE')
        <h3 class="text-sm font-semibold text-red-700 mb-2">Danger zone</h3>
        <button class="px-4 py-2 bg-red-600 text-white rounded">Archive</button>
    </form>
@endcan
```

### 5. `graduations/show.blade.php`

Detail + a table of `$graduation->students`. Each row links to `route('graduations.students.show', [$graduation, $student])`.

### 6. `students/show.blade.php`

Detail with the verify action:

```blade
@if ($student->isVerified())
    <span class="px-2 py-1 text-xs rounded bg-green-100 text-green-700">Verified</span>
@elseif ($student->hasPaid())
    <span class="px-2 py-1 text-xs rounded bg-amber-100 text-amber-700">Pending review</span>
@else
    <span class="px-2 py-1 text-xs rounded bg-slate-100 text-slate-600">Not paid</span>
@endif

@can('verify', $student)
    <form method="POST" action="{{ route('graduations.students.verify', [$graduation, $student]) }}"
          class="mt-4">
        @csrf @method('PATCH')
        <x-primary-button>Verify payment</x-primary-button>
    </form>
@endcan
```

### 7. `students/edit.blade.php`

```blade
<form method="POST"
      action="{{ route('graduations.students.update', [$graduation, $student]) }}"
      enctype="multipart/form-data"
      class="space-y-4">
    @csrf @method('PATCH')

    {{-- name, ic, email, matric_card, phone fields with old() + @error --}}

    <div>
        <x-input-label for="payment_receipt" :value="__('Payment receipt (PDF / JPG / PNG, max 2MB)')" />
        <input id="payment_receipt" name="payment_receipt" type="file"
               accept=".pdf,image/jpeg,image/png" class="mt-1 block w-full">
        <x-input-error :messages="$errors->get('payment_receipt')" class="mt-2" />

        @if ($student->payment_receipt)
            <p class="mt-2 text-sm text-gray-600">
                Current receipt:
                <a class="text-indigo-600" target="_blank"
                   href="{{ Storage::url($student->payment_receipt) }}">view</a>
            </p>
        @endif
    </div>

    <x-primary-button>Save</x-primary-button>
</form>
```

The `enctype="multipart/form-data"` and `@method('PATCH')` are both required for the file upload to land in `$request->file('payment_receipt')`.

## Expected output

```bash
php artisan migrate:fresh --seed
php artisan serve
```

In another terminal: `npm run dev`.

Then walk through:

1. Log in as `admin@devhub.test` / `password`.
2. Click **Graduations** in the nav → see 3 rows with badges.
3. Click **+ New graduation** → fill in the form → submit → land on the show page.
4. Edit it → see the danger-zone Archive button at the bottom.
5. Click a student → see their detail page.
6. Log out, log in as `student@devhub.test` → no **+ New graduation** button visible.

## Commit your work

```bash
git add .
git commit -m "feat(ui): add graduation + student Blade views + nav link"
```

## Common pitfalls

- **File upload silently saves an empty `payment_receipt`** — you forgot `enctype="multipart/form-data"` on the form.
- **`@method('PATCH')` missing on file-upload form** — the route is `PATCH /graduations/{graduation}/students/{student}` but the browser only knows POST. `@method('PATCH')` rewrites it server-side.
- **Status badge shows `Slate` color for everything** — your `$colors` array key doesn't match `$g->status`. Cast/print `$g->status` to confirm.

## What's next

Receipts are saving, but they're not viewable in the browser yet. We need a symlink: [10 — Storage symlink](./10-storage-symlink.md).
