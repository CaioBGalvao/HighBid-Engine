<!-- markdownlint-disable MD024 -->
# HighBid Engine Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Implement the scalable, real-time auction house backend with asynchronous bidding, Soft Close (Anti-Sniping), and real-time updates (Reverb + Mercure).

**Architecture:** The core bidding engine uses a Redis Lua script as a gatekeeper to protect the DB. Approved bids are pushed to a Laravel Horizon queue for asynchronous processing, which saves them to PostgreSQL and broadcasts WebSocket updates (Reverb) to the auction room, and uses Mercure for global user notifications.

**Tech Stack:** Laravel 12, PHP 8.5, PostgreSQL, Redis, Laravel Horizon, Laravel Reverb (WebSockets), FrankenPHP native Mercure (SSE), Pest for testing.

---

## Task 1: Database Setup (Models & Migrations)

### Files

- Create: `app/Models/Auction.php`, `database/migrations/xxxx_create_auctions_table.php`
- Create: `app/Models/Bid.php`, `database/migrations/xxxx_create_bids_table.php`
- Create: `app/Models/Follower.php`, `database/migrations/xxxx_create_followers_table.php`
- Test: `tests/Feature/Models/AuctionTest.php`

### Step 1: Write the failing test

```php
test('an auction can be created', function () {
    $auction = Auction::factory()->create([
        'title' => 'Vintage Watch',
        'current_price' => 1000,
    ]);
    expect($auction->title)->toBe('Vintage Watch');
});
```

### Step 2: Run test to verify it fails

Run: `php artisan test --compact --filter=AuctionTest`
Expected: FAIL.

### Step 3: Write minimal implementation

- Create migrations for Auctions (title, current_price, increments, starts_at, ends_at).
- Create migrations for Bids (auction_id, user_id, amount, is_proxy, max_proxy_amount).
- Create Models and Factories.

### Step 4: Run test to verify it passes

Run: `php artisan test --compact --filter=AuctionTest`
Expected: PASS

### Step 5: Commit

```bash
git add app/Models/ database/migrations/ database/factories/ tests/Feature/
git commit -m "feat: setup core database models for auctions and bids"
```

---

## Task 2: Redis Gatekeeper & Bidding Job

### Files

- Create: `app/Services/Bidding/BidGatekeeper.php`
- Create: `app/Jobs/ProcessAuctionBid.php`
- Test: `tests/Feature/Bidding/GatekeeperTest.php`

### Step 1: Write the failing test

```php
test('gatekeeper rejects lower bids', function () {
    $gatekeeper = new BidGatekeeper();
    // Simulate setting current highest bid in Redis to 1000
    // Try to bid 900 -> expect false
    // Try to bid 1100 -> expect true
});
```

### Step 2: Run test to verify it fails

Run: `php artisan test --compact --filter=GatekeeperTest`
Expected: FAIL.

### Step 3: Write minimal implementation

- Implement Lua Script in `BidGatekeeper` to atomically check `GET auction:{id}:price` and update if the new bid is higher.
- If true, dispatch `ProcessAuctionBid::dispatch()`.

### Step 4: Run test to verify it passes

Run: `php artisan test --compact --filter=GatekeeperTest`
Expected: PASS

### Step 5: Commit

```bash
git add app/Services/Bidding/ app/Jobs/ tests/Feature/Bidding/
git commit -m "feat: implement redis lua gatekeeper for bids"
```

---

## Task 3: Soft Close (Anti-Sniping) & Job Persistence

### Files

- Modify: `app/Jobs/ProcessAuctionBid.php`
- Test: `tests/Feature/Bidding/ProcessAuctionBidTest.php`

### Step 1: Write the failing test

```php
test('bid in the last 3 minutes extends auction end time', function() {
   // Create auction ending in 1 minute.
   // Process a bid job.
   // Assert ends_at is extended by 3 minutes.
});
```

### Step 2: Run test to verify it fails

Run: `php artisan test --compact --filter=ProcessAuctionBidTest`
Expected: FAIL.

### Step 3: Write minimal implementation

- In `ProcessAuctionBid`, validate inside a DB Transaction: if bid is valid, save to Postgres.
- Check if `now()` is within 3 mins of `ends_at`. If so, `ends_at = now()->addMinutes(3)`.

### Step 4: Run test to verify it passes

Run: `php artisan test --compact --filter=ProcessAuctionBidTest`
Expected: PASS

### Step 5: Commit

```bash
git add app/Jobs/ tests/Feature/Bidding/
git commit -m "feat: implement async db persistence and soft close logic"
```

---

## Task 4: Real-Time Broadcasting (Reverb & Mercure)

### Files

- Create: `app/Events/AuctionBidPlaced.php` (ShouldBroadcast on Reverb)
- Create: `app/Events/AuctionOutbidded.php` (ShouldBroadcast on Mercure)

### Step 1: Write the failing test

```php
test('auction bid placed event is broadcasted', function () {
    Event::fake();
    // process job
    Event::assertDispatched(AuctionBidPlaced::class);
});
```

### Step 2: Run test to verify it fails

Run: `php artisan test --compact`
Expected: FAIL.

### Step 3: Write minimal implementation

- Install Reverb (`php artisan install:broadcasting`).
- Fire `AuctionBidPlaced` (private channel `auction.{id}`) inside `ProcessAuctionBid`.

### Step 4: Run test to verify it passes

Run: `php artisan test --compact`
Expected: PASS

### Step 5: Commit

```bash
git add app/Events/ config/broadcasting.php
git commit -m "feat: broadcast bid events for realtime updates"
```
