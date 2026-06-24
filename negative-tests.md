# Laravel PHPUnit Negative Test Cases

## Test Cases Summary

| NT ID | Module | Scenario | Precondition | Input/Action | Expected Result | Severity |
|---|---|---|---|---|---|---|
| NT-AUTH-01 | Auth | Login with empty credentials | User exists but doesn't provide input | POST `/login` with empty payload | 302 Redirect with validation errors for email, password | High |
| NT-AUTH-02 | Auth | Login with invalid credentials | User exists | POST `/login` with wrong password | 302 Redirect with validation error for email | High |
| NT-AUTH-03 | Auth | Register with missing fields | None | POST `/register` with empty payload | 302 Redirect with validation errors for name, email, password | High |
| NT-AUTH-04 | Auth | Register with duplicate email | User with email exists | POST `/register` with existing email | 302 Redirect with validation error for email | High |
| NT-AUTH-05 | Auth | Register with unconfirmed password | None | POST `/register` with password and different password_confirmation | 302 Redirect with validation error for password | Medium |
| NT-AUTH-06 | Auth | Register with short password | None | POST `/register` with password length < 6 | 302 Redirect with validation error for password | Medium |
| NT-AUTH-07 | Auth | Access protected route unauthenticated | User is unauthenticated | GET `/dashboard` | 302 Redirect to `/login` | High |
| NT-COURT-01 | Court | Customer access admin courts index | Authenticated as Customer | GET `/courts` | 403 Forbidden | High |
| NT-COURT-02 | Court | Customer tries to create a court | Authenticated as Customer | POST `/courts` | 403 Forbidden | High |
| NT-COURT-03 | Court | Customer tries to update a court | Authenticated as Customer, Court exists | PUT `/courts/{id}` | 403 Forbidden | High |
| NT-COURT-04 | Court | Customer tries to delete a court | Authenticated as Customer, Court exists | DELETE `/courts/{id}` | 403 Forbidden | High |
| NT-COURT-05 | Court | Admin create court without name | Authenticated as Admin | POST `/courts` missing name | 302 Redirect with validation error for name | Medium |
| NT-COURT-06 | Court | Admin create court with invalid type | Authenticated as Admin | POST `/courts` with type 'Space' | 302 Redirect with validation error for type | Medium |
| NT-COURT-07 | Court | Admin create court with negative price | Authenticated as Admin | POST `/courts` with price_per_hour = -10 | 302 Redirect with validation error for price_per_hour | Medium |
| NT-COURT-08 | Court | Admin update court with invalid status | Authenticated as Admin, Court exists | PUT `/courts/{id}` with status 'Pending' | 302 Redirect with validation error for status | Medium |
| NT-BOOK-01 | Booking | Unauthenticated booking creation | User is unauthenticated | POST `/book` | 302 Redirect to `/login` | High |
| NT-BOOK-02 | Booking | Create booking missing court_id | Authenticated as Customer | POST `/book` missing court_id | 302 Redirect with validation error for court_id | High |
| NT-BOOK-03 | Booking | Create booking with invalid court_id | Authenticated as Customer | POST `/book` with non-existent court_id | 302 Redirect with validation error for court_id | High |
| NT-BOOK-04 | Booking | Create booking with past date | Authenticated as Customer | POST `/book` with yesterday's date | 302 Redirect with validation error for date | High |
| NT-BOOK-05 | Booking | Create booking invalid time format | Authenticated as Customer | POST `/book` with start_time '12 PM' | 302 Redirect with validation error for start_time | Medium |
| NT-BOOK-06 | Booking | Create booking end time before start | Authenticated as Customer | POST `/book` start_time '14:00', end_time '13:00' | 302 Redirect with validation error for end_time | High |
| NT-BOOK-07 | Booking | Create booking on inactive court | Authenticated as Customer, Inactive Court | POST `/book` with inactive court_id | 302 Redirect with custom error 'court_id' | High |
| NT-BOOK-08 | Booking | Create booking overlapping time slot | Authenticated as Customer, Existing Booking | POST `/book` with overlapping time on same court & date | 302 Redirect with custom error 'start_time' | High |
| NT-HIST-01 | History | Customer access admin bookings | Authenticated as Customer | GET `/admin/bookings` | 403 Forbidden | High |
| NT-HIST-02 | History | Customer update admin booking status | Authenticated as Customer, Booking exists | PATCH `/admin/bookings/{id}/status` | 403 Forbidden | High |
| NT-HIST-03 | History | Customer delete booking | Authenticated as Customer, Booking exists | DELETE `/admin/bookings/{id}` | 403 Forbidden | High |
| NT-HIST-04 | History | Customer cancel another user's booking | Authenticated as Customer, Booking belongs to other | PATCH `/my-bookings/{id}/cancel` | 403 Forbidden | High |
| NT-HIST-05 | History | Admin update status to invalid value | Authenticated as Admin, Booking exists | PATCH `/admin/bookings/{id}/status` with status 'Unknown'| 302 Redirect with validation error for status | Medium |


## Full Laravel PHPUnit Test Code

### `tests/Feature/NegativeAuthTest.php`

```php
<?php

namespace Tests\Feature;

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class NegativeAuthTest extends TestCase
{
    use RefreshDatabase;

    public function test_login_with_empty_credentials()
    {
        $response = $this->post('/login', []);

        $response->assertSessionHasErrors(['email', 'password']);
    }

    public function test_login_with_invalid_credentials()
    {
        $user = User::factory()->create([
            'email' => 'test@example.com',
            'password' => bcrypt('password123'),
        ]);

        $response = $this->post('/login', [
            'email' => 'test@example.com',
            'password' => 'wrongpassword',
        ]);

        $response->assertSessionHasErrors('email');
        $this->assertGuest();
    }

    public function test_register_with_missing_fields()
    {
        $response = $this->post('/register', []);

        $response->assertSessionHasErrors(['name', 'email', 'password']);
    }

    public function test_register_with_duplicate_email()
    {
        User::factory()->create(['email' => 'duplicate@example.com']);

        $response = $this->post('/register', [
            'name' => 'John Doe',
            'email' => 'duplicate@example.com',
            'password' => 'password123',
            'password_confirmation' => 'password123',
        ]);

        $response->assertSessionHasErrors('email');
    }

    public function test_register_with_unconfirmed_password()
    {
        $response = $this->post('/register', [
            'name' => 'John Doe',
            'email' => 'john@example.com',
            'password' => 'password123',
            'password_confirmation' => 'different123',
        ]);

        $response->assertSessionHasErrors('password');
    }

    public function test_register_with_short_password()
    {
        $response = $this->post('/register', [
            'name' => 'John Doe',
            'email' => 'john@example.com',
            'password' => '12345',
            'password_confirmation' => '12345',
        ]);

        $response->assertSessionHasErrors('password');
    }

    public function test_access_protected_route_unauthenticated()
    {
        $response = $this->get('/dashboard');

        $response->assertRedirect('/login');
    }
}
```

### `tests/Feature/NegativeCourtTest.php`

```php
<?php

namespace Tests\Feature;

use App\Models\Court;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class NegativeCourtTest extends TestCase
{
    use RefreshDatabase;

    private User $customer;
    private User $admin;
    private Court $court;

    protected function setUp(): void
    {
        parent::setUp();

        $this->customer = User::factory()->create(['role' => User::ROLE_CUSTOMER]);
        $this->admin = User::factory()->create(['role' => User::ROLE_ADMIN]);
        $this->court = Court::factory()->create();
    }

    public function test_customer_cannot_access_courts_index()
    {
        $response = $this->actingAs($this->customer)->get('/courts');

        $response->assertStatus(403);
    }

    public function test_customer_cannot_create_court()
    {
        $response = $this->actingAs($this->customer)->post('/courts', [
            'name' => 'New Court',
            'type' => 'Indoor',
            'price_per_hour' => 100,
            'status' => 'Active',
        ]);

        $response->assertStatus(403);
    }

    public function test_customer_cannot_update_court()
    {
        $response = $this->actingAs($this->customer)->put("/courts/{$this->court->id}", [
            'name' => 'Updated Court',
            'type' => 'Outdoor',
            'price_per_hour' => 150,
            'status' => 'Inactive',
        ]);

        $response->assertStatus(403);
    }

    public function test_customer_cannot_delete_court()
    {
        $response = $this->actingAs($this->customer)->delete("/courts/{$this->court->id}");

        $response->assertStatus(403);
    }

    public function test_admin_cannot_create_court_without_name()
    {
        $response = $this->actingAs($this->admin)->post('/courts', [
            'type' => 'Indoor',
            'price_per_hour' => 100,
            'status' => 'Active',
        ]);

        $response->assertSessionHasErrors('name');
    }

    public function test_admin_cannot_create_court_with_invalid_type()
    {
        $response = $this->actingAs($this->admin)->post('/courts', [
            'name' => 'Court 1',
            'type' => 'Space',
            'price_per_hour' => 100,
            'status' => 'Active',
        ]);

        $response->assertSessionHasErrors('type');
    }

    public function test_admin_cannot_create_court_with_negative_price()
    {
        $response = $this->actingAs($this->admin)->post('/courts', [
            'name' => 'Court 1',
            'type' => 'Indoor',
            'price_per_hour' => -10,
            'status' => 'Active',
        ]);

        $response->assertSessionHasErrors('price_per_hour');
    }

    public function test_admin_cannot_update_court_with_invalid_status()
    {
        $response = $this->actingAs($this->admin)->put("/courts/{$this->court->id}", [
            'name' => 'Court 1',
            'type' => 'Indoor',
            'price_per_hour' => 100,
            'status' => 'Pending',
        ]);

        $response->assertSessionHasErrors('status');
    }
}
```

### `tests/Feature/NegativeBookingTest.php`

```php
<?php

namespace Tests\Feature;

use App\Models\Booking;
use App\Models\Court;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class NegativeBookingTest extends TestCase
{
    use RefreshDatabase;

    private User $customer;
    private Court $court;

    protected function setUp(): void
    {
        parent::setUp();

        $this->customer = User::factory()->create(['role' => User::ROLE_CUSTOMER]);
        $this->court = Court::factory()->create([
            'status' => Court::STATUS_ACTIVE,
            'price_per_hour' => 100,
        ]);
    }

    public function test_unauthenticated_user_cannot_create_booking()
    {
        $response = $this->post('/book', [
            'court_id' => $this->court->id,
            'date' => now()->addDay()->format('Y-m-d'),
            'start_time' => '10:00',
            'end_time' => '12:00',
        ]);

        $response->assertRedirect('/login');
    }

    public function test_create_booking_missing_court_id()
    {
        $response = $this->actingAs($this->customer)->post('/book', [
            'date' => now()->addDay()->format('Y-m-d'),
            'start_time' => '10:00',
            'end_time' => '12:00',
        ]);

        $response->assertSessionHasErrors('court_id');
    }

    public function test_create_booking_with_invalid_court_id()
    {
        $response = $this->actingAs($this->customer)->post('/book', [
            'court_id' => 9999,
            'date' => now()->addDay()->format('Y-m-d'),
            'start_time' => '10:00',
            'end_time' => '12:00',
        ]);

        $response->assertSessionHasErrors('court_id');
    }

    public function test_create_booking_with_past_date()
    {
        $response = $this->actingAs($this->customer)->post('/book', [
            'court_id' => $this->court->id,
            'date' => now()->subDay()->format('Y-m-d'),
            'start_time' => '10:00',
            'end_time' => '12:00',
        ]);

        $response->assertSessionHasErrors('date');
    }

    public function test_create_booking_with_invalid_time_format()
    {
        $response = $this->actingAs($this->customer)->post('/book', [
            'court_id' => $this->court->id,
            'date' => now()->addDay()->format('Y-m-d'),
            'start_time' => '10 AM',
            'end_time' => '12:00',
        ]);

        $response->assertSessionHasErrors('start_time');
    }

    public function test_create_booking_end_time_before_start_time()
    {
        $response = $this->actingAs($this->customer)->post('/book', [
            'court_id' => $this->court->id,
            'date' => now()->addDay()->format('Y-m-d'),
            'start_time' => '12:00',
            'end_time' => '10:00',
        ]);

        $response->assertSessionHasErrors('end_time');
    }

    public function test_create_booking_on_inactive_court()
    {
        $inactiveCourt = Court::factory()->create(['status' => Court::STATUS_INACTIVE]);

        $response = $this->actingAs($this->customer)->post('/book', [
            'court_id' => $inactiveCourt->id,
            'date' => now()->addDay()->format('Y-m-d'),
            'start_time' => '10:00',
            'end_time' => '12:00',
        ]);

        $response->assertSessionHasErrors(['court_id' => 'This court is currently not available.']);
    }

    public function test_create_booking_overlapping_time_slot()
    {
        $date = now()->addDay()->format('Y-m-d');
        
        Booking::factory()->create([
            'court_id' => $this->court->id,
            'user_id' => $this->customer->id,
            'date' => $date,
            'start_time' => '10:00',
            'end_time' => '12:00',
            'status' => Booking::STATUS_BOOKED,
        ]);

        $response = $this->actingAs($this->customer)->post('/book', [
            'court_id' => $this->court->id,
            'date' => $date,
            'start_time' => '11:00',
            'end_time' => '13:00',
        ]);

        $response->assertSessionHasErrors(['start_time' => 'This time slot overlaps an existing booking for this court.']);
    }
}
```

### `tests/Feature/NegativeHistoryTest.php`

```php
<?php

namespace Tests\Feature;

use App\Models\Booking;
use App\Models\Court;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class NegativeHistoryTest extends TestCase
{
    use RefreshDatabase;

    private User $customer;
    private User $otherCustomer;
    private User $admin;
    private Booking $booking;

    protected function setUp(): void
    {
        parent::setUp();

        $this->customer = User::factory()->create(['role' => User::ROLE_CUSTOMER]);
        $this->otherCustomer = User::factory()->create(['role' => User::ROLE_CUSTOMER]);
        $this->admin = User::factory()->create(['role' => User::ROLE_ADMIN]);
        
        $court = Court::factory()->create();
        
        $this->booking = Booking::factory()->create([
            'court_id' => $court->id,
            'user_id' => $this->otherCustomer->id, // belongs to other customer
            'status' => Booking::STATUS_BOOKED,
        ]);
    }

    public function test_customer_cannot_access_admin_bookings()
    {
        $response = $this->actingAs($this->customer)->get('/admin/bookings');

        $response->assertStatus(403);
    }

    public function test_customer_cannot_update_admin_booking_status()
    {
        $response = $this->actingAs($this->customer)->patch("/admin/bookings/{$this->booking->id}/status", [
            'status' => Booking::STATUS_COMPLETED,
        ]);

        $response->assertStatus(403);
    }

    public function test_customer_cannot_delete_booking()
    {
        $response = $this->actingAs($this->customer)->delete("/admin/bookings/{$this->booking->id}");

        $response->assertStatus(403);
    }

    public function test_customer_cannot_cancel_another_users_booking()
    {
        $response = $this->actingAs($this->customer)->patch("/my-bookings/{$this->booking->id}/cancel");

        $response->assertStatus(403);
    }

    public function test_admin_cannot_update_status_to_invalid_value()
    {
        $response = $this->actingAs($this->admin)->patch("/admin/bookings/{$this->booking->id}/status", [
            'status' => 'UnknownStatus',
        ]);

        $response->assertSessionHasErrors('status');
    }
}
```
