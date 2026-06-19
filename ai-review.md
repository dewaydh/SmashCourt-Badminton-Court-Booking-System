# Laravel Code Quality & Best Practices Review

Below is a thorough review of the `app/` directory, `routes/web.php`, and database migrations. The focus is on finding bugs, data integrity issues, performance bottlenecks, and areas where Laravel best practices can improve robustness.

---

## 1. Data-Integrity Problems

### A. Destructive Cascading Deletes
**File:** `database/migrations/0001_01_01_000002_create_bookings_table.php`  
**Severity:** High  
**Why it matters:** The migrations use `cascadeOnDelete()` for both `court_id` and `user_id`. If an admin deletes a court, or a user account is removed, **all associated bookings are permanently deleted**. This destroys historical, financial, and operational records.  
**Suggested Improvement:** 
Use Laravel's **Soft Deletes** on the `Court` and `User` models so records are hidden rather than destroyed. Alternatively, make the foreign keys nullable and use `nullOnDelete()`, or restrict deletion if bookings exist.

### B. Privilege Escalation Risk (Mass Assignment)
**File:** `app/Models/User.php`  
**Severity:** High  
**Why it matters:** The `role` attribute is included in the `$fillable` array. While your current `AuthController` explicitly defines the array passed to `User::create()`, if any developer ever uses `User::update($request->all())` in the future, a malicious user could submit a form with `role=admin` to elevate their privileges.  
**Suggested Improvement:** 
Remove `'role'` from `$fillable`. Assign it manually when creating users:
```php
$user = new User($data);
$user->role = User::ROLE_CUSTOMER;
$user->save();
```

---

## 2. Bugs and Logic Errors

### A. Booking Overlap Race Condition
**File:** `app/Http/Controllers/BookingController.php` (in `store()`)  
**Severity:** High  
**Why it matters:** The code checks for overlapping bookings using a `SELECT` query, then creates the booking using an `INSERT`. Under heavy load, two users might try to book the exact same slot at the exact same millisecond. Both will pass the `SELECT` check and insert conflicting bookings.  
**Suggested Improvement:** 
Wrap the booking creation in a database transaction with a pessimistic lock. For example, you can lock the Court row (`Court::where('id', ...)->lockForUpdate()->first()`) so concurrent requests for the same court queue up and evaluate sequentially.

### B. Unrestricted Booking Cancellation
**File:** `app/Http/Controllers/BookingController.php` (in `cancel()`)  
**Severity:** Medium  
**Why it matters:** A user can cancel their booking at any time, even 1 minute before the start time, or potentially *after* the start time if the status hasn't been updated to 'Completed' yet. This leads to lost revenue.  
**Suggested Improvement:** 
Add a time-based validation check to ensure cancellations are only allowed, for example, 24 hours in advance.
```php
if (Carbon::parse($booking->date . ' ' . $booking->start_time)->isPast()) {
    return back()->withErrors('Cannot cancel a booking that has already started.');
}
```

---

## 3. Missing Validation and Security

### A. No Rate Limiting on Authentication
**File:** `app/Http/Controllers/Auth/AuthController.php` & `routes/web.php`  
**Severity:** Medium  
**Why it matters:** Your custom `login` method does not utilize rate limiting. A bot could brute-force passwords without being blocked.  
**Suggested Improvement:** 
Apply Laravel's `throttle` middleware to the login route in `web.php` (e.g., `Route::post('/login', ...)->middleware('throttle:5,1');`), or better yet, use Laravel Breeze/Fortify which comes with secure, pre-configured `LoginRequest` classes that handle rate limiting automatically.

---

## 4. Performance Concerns

### A. Missing Pagination (N+1 memory bloat)
**Files:** 
- `app/Http/Controllers/Admin/BookingController.php` (`index()`)
- `app/Http/Controllers/BookingController.php` (`index()`)
- `app/Http/Controllers/CourtController.php` (`index()`)  
**Severity:** Medium  
**Why it matters:** Calling `->get()` retrieves *all* records from the database into memory. For the Admin bookings index, this will load the entire history of the business. As the database grows, this will cause Out Of Memory (OOM) errors and extremely slow page loads.  
**Suggested Improvement:** 
Replace `->get()` with `->paginate(20)` in your controllers, and use `{{ $bookings->links() }}` in your Blade views.

---

## 5. Code Smells & Maintainability

### A. Hardcoded Authorization Checks
**File:** `app/Http/Controllers/BookingController.php` (in `cancel()`)  
**Severity:** Low  
**Why it matters:** You manually check `$booking->user_id !== $request->user()->id` and throw an `abort(403)`. While functional, spreading authorization logic throughout controllers makes it hard to maintain and test.  
**Suggested Improvement:** 
Generate a Laravel Policy (`php artisan make:policy BookingPolicy`).
```php
// In BookingPolicy.php
public function cancel(User $user, Booking $booking) {
    return $user->id === $booking->user_id;
}

// In BookingController.php
$this->authorize('cancel', $booking);
// Or in PHP 8.1+ / Laravel 10+:
Gate::authorize('cancel', $booking);
```

### B. Duplicated Validation Rules
**File:** `app/Http/Controllers/CourtController.php`  
**Severity:** Low  
**Why it matters:** The validation rules for a Court are duplicated identically in both `store()` and `update()`. If you add a new field (like `description`), you have to remember to update it in two places.  
**Suggested Improvement:** 
Use Laravel Form Requests (`php artisan make:request CourtRequest`) to encapsulate the validation logic in one reusable class.

### C. Manual Time Calculations vs Carbon
**File:** `app/Http/Controllers/BookingController.php` (in `store()`)  
**Severity:** Low  
**Why it matters:** You are using native PHP `strtotime()` to calculate the duration.
```php
$start = strtotime($data['start_time']);
$end = strtotime($data['end_time']);
$hours = ($end - $start) / 3600;
```
**Suggested Improvement:** 
Leverage Laravel's built-in `Carbon` library for robust and readable date/time manipulation.
```php
$start = Carbon::createFromTimeString($data['start_time']);
$end = Carbon::createFromTimeString($data['end_time']);
$hours = $start->diffInMinutes($end) / 60;
```
