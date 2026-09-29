# solutions.md — Rescu assessment

Candidate: Chadayu Kerdsanthat
Environment: Windows, Flutter 3.27.0, Java 17, Android emulator (`sdk gphone16k x86 64`)

> Status legend: DONE = fixed and verified, TODO = not started

| Item | Status |
|---|---|
| RES-101 | DONE |
| RES-102 | DONE |
| RES-103 | DONE |
| RES-104 | DONE |
| RES-105 | TODO |
| RES-106 | DONE |
| RES-107 | TODO |
| F-1 | TODO |
| F-2 | TODO |
| F-3 | TODO |

---

## Part A — Bug tickets

### RES-102 · Crash after leaving My orders

**Reproduction**

1. Run the app in debug mode on the emulator.
2. Home → open **My orders** (orders with an upcoming pickup are listed, each showing "Opens in mm:ss").
3. Press back and stay on Home for 1–2 seconds.
4. The console prints `setState() called after dispose()`, once per countdown row.

Log before the fix (trimmed to one error):

```
E/flutter: Unhandled Exception: setState() called after dispose(): _PickupCountdownState#17e8a(lifecycle state: defunct, not mounted)
E/flutter: #2  _PickupCountdownState.initState.<anonymous closure> (package:rescu/feature/order/widget/pickup_countdown.dart:22:7)
E/flutter: #3  _Timer._runTimers (dart:isolate-patch/timer_impl.dart:398:19)
```

Different State ids (`#17e8a`, `#725eb`, `#5f572`) appeared in the same log, one per row on the My orders screen.

**Root cause**

`_PickupCountdownState.initState` starts a `Timer.periodic` that calls `setState` every second, but the Timer was never stored and never cancelled. A `Timer` belongs to the Dart event loop, not to the widget, so when the route is popped and the State is disposed, the Timer keeps firing. On the next tick it calls `setState` on a defunct State, which throws. Every row has its own countdown widget, so every row leaks its own Timer (which is also why the errors appear several times).

**Fix**

Keep the Timer in a field and cancel it in `dispose()` (`pickup_countdown.dart`):

```dart
Timer? _timer;

@override
void initState() {
  super.initState();
  _timer = Timer.periodic(const Duration(seconds: 1), (_) {
    setState(() {});
  });
}

@override
void dispose() {
  _timer?.cancel();
  super.dispose();
}
```

**Why this fix**

The widget acquires a resource in `initState`, so it must release it in `dispose`. This removes the leak at the source instead of hiding the error.

**Alternative considered and rejected**

Guarding the callback with `if (mounted) setState(() {})`. It silences the exception (Flutter's own error text even suggests it), but the Timer would still fire every second forever after the screen is gone, and it keeps a reference to the disposed State, so it is a small leak. It treats the symptom, not the cause.

**Edge cases**

- Handled: multiple countdown widgets on one screen (each cancels its own Timer); leaving and re-entering the screen quickly (each new State creates and cancels its own Timer).
- Not handled (decided not to): the Timer keeps ticking each second even after the pickup window has opened or when only hours and minutes are shown ("Opens in 1h 53m"), which is a wasted rebuild but harmless; one shared ticker for all rows would be cleaner but is a larger refactor outside this ticket.

**Verification**

After the fix I opened My orders and left it three times, staying on Home for 10+ seconds each time. No `setState() called after dispose()` appeared in the console, and the countdown still counts down correctly when the screen is open (leaving for ~11 seconds and returning shows the number reduced by ~11 seconds, since the text is computed from `pickupStart - DateTime.now()`).

Commit: `[RES-102] Cancel PickupCountdown timer in dispose to stop setState after dispose`

---

### RES-101 · Search shows results for the wrong query

**Reproduction**

1. Go to the Search screen.
2. Type a query fast, then quickly delete it and type a different
   one (e.g. "vegan" then quickly switch to "sushi").
3. Watch the console: requests are sent for every keystroke, and
   they don't resolve in the same order they were sent — a request
   for an older, shorter query can take longer and return after a
   request for a newer query that returned faster.

Log evidence (before the fix), typing "pizza":

```
12:57:49.309  GET /deals/search?q=pizz   (293ms) -> resolves 49.602
12:57:49.478  GET /deals/search?q=pizza  (182ms) -> resolves 49.660
12:57:49.630  GET /deals/search?q=piz    (753ms) -> resolves 50.383
12:57:49.769  GET /deals/search?q=p      (1276ms) -> resolves 51.045
```

`p` was sent last but resolved last too — if nothing checked which
query a response belonged to, its (irrelevant) results would
overwrite the results for "pizza" that the user was actually
looking at.

**Root cause**

`onQueryChanged` called `_search` on every keystroke, with no
debounce, so many requests were in flight at once. Worse, when a
request resolved, `_search` called `results.assignAll(found)`
unconditionally — with no check that the response still matched
the query currently in the search box. Because requests don't
necessarily resolve in the order they were sent, a slower response
for an older query could arrive after a faster response for a
newer query and silently overwrite it.

**Fix**

In `search_deals_controller.dart`:
- Added a 300ms debounce Timer in `onQueryChanged`, so a burst of
  keystrokes only triggers one search after typing pauses.
- Added `_latestQuery`, updated at the start of `_search`, and
  checked against it before `assignAll`: `if (query != _latestQuery) return;`
  discards a response if a newer query has already superseded it.
- Cancel the debounce Timer in `onClose()`.

**Why this fix**

Debouncing reduces the number of in-flight requests, but does not
guarantee ordering by itself — two requests spaced further apart
than the debounce window can still resolve out of order. The
`_latestQuery` check is what actually prevents a stale response
from being displayed; the debounce is a secondary improvement to
reduce request volume.

**Alternative considered and rejected**

Debounce only, without the `_latestQuery` check. It makes the bug
much rarer (fewer overlapping requests), but doesn't fix it: two
requests further apart than 300ms can still race, exactly as shown
in the log above where `p` and `pizza` were ~140ms apart.

**Edge cases**

- Handled: rapidly switching between unrelated queries (e.g.
  "vegan" then "sushi") always leaves the UI showing results for
  the last query typed, verified in the console log.
- Not handled (decided not to): the actual HTTP request for a
  stale query is still sent and completed, just ignored on arrival
  — no request is cancelled. Using a proper cancel token would save
  bandwidth but is outside the scope of this ticket.

**Verification**

After the fix, typing quickly and switching between queries no
longer shows results for an old query — the log confirms the
displayed results always match the last query typed (`sushi` in
the final test), even when responses for earlier queries arrive
out of order.

Commit: `[RES-101] Debounce search and drop stale responses to fix results showing old query`

### RES-103 · Requests pile up the longer you browse

**Reproduction**

1. Open deal 1 from Home, then go back.
2. Open deal 2, then go back.
3. Open deal 3, then go back.
4. Open deal 1 again and tap **Add to bag**.
5. The console shows `re-checking availability for deal 1` several
   times in a row for a single tap, one extra time for every deal
   details screen visited earlier in the session.

Log evidence (before the fix), after visiting deals 1, 2 and 3 and
then adding deal 1 to the bag:

```
re-checking availability for deal 1
GET /deals/1
re-checking availability for deal 1
GET /deals/1
re-checking availability for deal 1
GET /deals/1
```

**Root cause**

`DealDetailsController.onInit` subscribes to the cart with
`ever(cartService.itemCount, (_) => _recheckAvailability())`.
`ever` returns a `Worker` that keeps listening until it is disposed,
but the controller never stored it and never disposed it. `CartService`
is a `GetxService` that lives for the whole session, so `itemCount`
outlives any single deal details screen. Every time the screen is
opened, a new permanent listener is added on top of the ones from
every earlier visit; none of them are ever removed. Tapping **Add to
bag** changes `itemCount` once, which fires every listener that has
piled up so far — each one calling `dealRepo.fetchById` for its own
deal — so the number of requests grows with how many deal screens
were visited earlier in the session, not with how many times you tap
add. Tapping add a second time for a deal that is already at its
stock limit does not fire anything, because `CartService.add` returns
early (`existing.quantity >= deal.quantityLeft`) without touching
`itemCount`, so no listener runs — which matches what was observed
while narrowing down the bug.

**Fix**

In `deal_details_controller.dart`:

```dart
late final Worker _cartWatcher;

@override
void onInit() {
  super.onInit();
  ...
  _cartWatcher = ever(cartService.itemCount, (_) => _recheckAvailability());
}

@override
void onClose() {
  _cartWatcher.dispose();
  super.onClose();
}
```

**Why this fix**

The controller acquires a listener (a `Worker`) in `onInit`, so it
must release it in `onClose`, the same lifecycle pairing as RES-102
but on the GetX side instead of the Flutter `State` side. Storing the
`Worker` and disposing it is the minimal change that stops the leak
at its source.

**Alternative considered and rejected**

Checking `deal.id` inside the listener and only recomputing for the
deal currently added to the cart. This would reduce wasted work but
would not fix the underlying leak — listeners from every past screen
visit would still accumulate and still all run on every cart change,
just doing a cheaper no-op instead of a network call. It treats a
symptom (duplicate work) rather than the cause (the worker never
being disposed).

**Edge cases**

- Handled: visiting several deal screens back-to-back before adding
  anything to the bag — only the currently open screen's listener
  fires when its own `onClose` hasn't run yet, and screens that were
  closed no longer react at all.
- Not handled (decided not to): if two deal screens for the *same*
  deal are somehow open at once (not reachable through normal
  navigation in this app), both would still re-check independently;
  this is out of scope since GetX only keeps one route of a given
  deal on the stack in practice.

**Verification**

Before the fix, opening deals 1, 2 and 3 in sequence and then adding
deal 1 to the bag printed `re-checking availability for deal 1`
three times. After the fix, repeating the exact same steps (deal 1 →
back → deal 2 → back → deal 3 → back → deal 1 → Add to bag) printed
it exactly once, confirmed with a hot restart between the two runs.

Commit: `[RES-103] Dispose cart-change worker to stop accumulating stock re-check listeners`

### RES-104 · Duplicate deals in the home feed

**Reproduction**

1. On Home, scroll to the bottom of the feed so `loadMore` starts
   fetching the next page.
2. While that request is still in flight, scroll back to the top and
   pull down to refresh.
3. Intermittently the feed ends up with duplicate cards, or with more
   items than the 122-deal catalog contains.

Because the two requests race, this only reproduces reliably if the
refresh lands while `loadMore`'s request is still pending. To make the
timing repeatable for testing, I temporarily added an artificial
delay before the `loadMore` network call and a log line reporting
`deals.length` vs. the number of unique ids after each fetch (both
removed before the final commit).

Log with the artificial delay, refreshing while `loadMore` for page 2
was still pending:

```
loadMore START page=2 — you have 10s to pull-to-refresh now
GET /deals?page=1
refresh done: deals=20 unique=20
GET /deals?page=2
loadMore page=2 done: _page=1 deals=40 unique=40
```

**Root cause**

`refreshDeals()` and `loadMore()` both read and write the same
`_page` field and the same `deals` list, with no coordination between
them:

1. `loadMore()` increments `_page` (e.g. to 2) and awaits the page 2
   response.
2. The user pulls to refresh: `refreshDeals()` resets `_page = 1` and
   awaits the page 1 response.
3. If the refresh response arrives first, `deals.assignAll(page 1
   items)` correctly resets the list to 20 items.
4. The now-stale `loadMore` response for page 2 then arrives and
   calls `deals.addAll(page 2 items)` unconditionally — it has no way
   to know the list was reset out from under it. The list now has 40
   items while `_page` still says 1.
5. The next scroll-to-bottom increments `_page` to 2 again and
   re-fetches a page that is already partly represented in `deals`,
   producing duplicate cards or a total above 122.

This is the same category of bug as RES-101: a response that is no
longer relevant by the time it arrives is allowed to overwrite
current state, because nothing tracks whether the request that
produced it is still the "current" one.

**Fix**

In `home_controller.dart`, add a generation counter that every
refresh bumps, plus a flag that stops a new `loadMore` from starting
while a refresh is in flight:

```dart
int _generation = 0; // bumps on every refresh; invalidates in-flight loadMore
bool _isRefreshing = false;

Future<void> refreshDeals() async {
  final gen = ++_generation;
  _isRefreshing = true;
  _isFetchingMore = false; // a stale loadMore no longer owns this flag
  try {
    final res = await dealRepo.fetchDeals(page: 1);
    if (gen != _generation) return; // a newer refresh superseded this one
    _page = 1;
    _totalPages = res.totalPages;
    deals.assignAll(res.items);
    refreshController.refreshCompleted();
  } finally {
    if (gen == _generation) _isRefreshing = false;
  }
}

Future<void> loadMore() async {
  if (_isFetchingMore || _isRefreshing) return;
  if (!hasMore) {
    refreshController.loadNoData();
    return;
  }
  _isFetchingMore = true;
  final gen = _generation;
  final page = _page + 1; // don't touch _page until the page really lands
  try {
    final res = await dealRepo.fetchDeals(page: page);
    if (gen != _generation) return; // a refresh replaced the list meanwhile
    _page = page;
    _totalPages = res.totalPages;
    deals.addAll(res.items);
  } catch (e) {
    LogService.error('loadMore failed', e);
  } finally {
    if (gen == _generation) _isFetchingMore = false;
    refreshController.loadComplete();
  }
}
```

**Why this fix**

A `loadMore` response is only safe to apply if no refresh happened
since it was requested, so it captures the current `_generation`
before awaiting and checks it against the live value once the
response lands; a mismatch means it is stale and should be dropped
instead of merged in. `_page` is also only advanced once the
corresponding page has actually landed, rather than eagerly before
the request, so a failed or discarded request can no longer leave
`_page` out of sync with what `deals` actually contains (the original
code's `_page--` on error had the same class of problem). `_isRefreshing`
additionally stops a brand new `loadMore` from starting mid-refresh,
which the generation check alone would not prevent.

**Alternative considered and rejected**

De-duplicating by deal id when appending in `loadMore` (e.g. only
adding items whose id is not already in `deals`). This would hide
duplicate cards, but `_page` would still end up wrong and pages that
were silently dropped as "already present" would never really be
loaded, quietly breaking pagination. It patches the visible symptom
without fixing the state that caused it.

**Edge cases**

- Handled: a refresh that lands before an in-flight `loadMore`
  (normal case, not a race — `loadMore`'s response is simply
  discarded); a `loadMore` that fails mid-flight (no longer
  decrements `_page`, since `_page` was never advanced early).
- Not handled (decided not to): two refreshes fired back-to-back
  before either resolves — the second `_generation` bump correctly
  invalidates the first, so only the newest refresh's response is
  applied; this is already covered by the same mechanism, not a gap.

**Verification**

With the original code and an artificial delay to make the race
reproducible, refreshing while a `loadMore` for page 2 was still
pending consistently produced `deals=40` right after the refresh
(should have been 20). With the fix applied, the identical sequence
produced `refresh done: deals=20 unique=20` followed by
`loadMore page=2 dropped (stale, refresh happened)`, leaving the list
at the correct 20 items. Repeated the sequence twice with the same
result each time.

Commit: `[RES-104] Ignore stale loadMore responses after refresh to stop duplicate feed items`

### RES-105 · Home feed is janky and memory keeps climbing

TODO (needs DevTools before/after evidence)

### RES-106 · Wrong pickup times; "Pickup today" filter misses deals

**Reproduction**

1. Open a deal whose store has an early-morning pickup window, e.g.
   deal 1 "Mystery Thai Feast" at Baan Somtam Kitchen, configured in
   `stores.json` as `pickupStartHour: 5, pickupStartMinute: 30,
   pickupEndHour: 8` (store-local Bangkok time).
2. The deal details screen shows a **"Pickup window: 22:30 – 01:00"**
   instead of the correct 05:30 – 08:00.
3. On Home, the "Pickup today" filter hides or shows deals
   inconsistently with their actual local pickup time, especially for
   stores with pickup windows spanning midnight.

**Root cause**

The fake backend (`fake_api_service.dart`) correctly converts each
store's local pickup hours to UTC before sending them, using the
market's fixed UTC offset. But `PickupWindowModel.fromJson` parsed
those UTC ISO-8601 strings with `DateTime.parse(...)` and never
called `.toLocal()`. Both derived getters read directly off that raw
UTC `DateTime`:

- `label` formats `start`/`end` with `DateFormat('HH:mm')`, so it
  printed the UTC hour instead of the Bangkok-local hour — off by
  exactly 7 hours (the market's UTC offset).
- `isToday` compares `start.day == DateTime.now().day`, so for
  windows near local midnight the UTC day and local day disagree,
  making "Pickup today" hide or show the wrong deals.

`isOpenNow` was unaffected, because it only compares absolute instants
(`isAfter`/`isBefore`), which are correct regardless of which timezone
the `DateTime` claims to be in.

**Fix**

In `pickup_window_model.dart`, add `.toLocal()` to both `DateTime.parse`
calls in the `fromJson` factory:

```dart
factory PickupWindowModel.fromJson(Map<String, dynamic> json) {
  return PickupWindowModel(
    start: DateTime.parse(json['start'] as String).toLocal(),
    end: DateTime.parse(json['end'] as String).toLocal(),
  );
}
```

**Why this fix**

`start` and `end` are the single source every getter reads from, so
converting to local time once at the parsing boundary fixes `label`
and `isToday` together without touching either getter, and without
affecting `isOpenNow`, which was already correct.

**Alternative considered and rejected**

Fixing `label` and `isToday` individually (e.g. adding a UTC offset
inside each getter). This duplicates the timezone conversion in two
places and would only fix the two getters that happen to be
observably wrong today; any future getter added off `start`/`end`
would inherit the same bug. Converting once in `fromJson` fixes the
model at its root.

**Edge cases**

- Handled: overnight pickup windows, e.g. Chao Phraya Sushi's
  22:00 – 01:00, verified to correctly show/hide under "Pickup
  today" depending on the current local time.
- Not handled (decided not to): a user changing their device
  timezone mid-session — `.toLocal()` uses the device's current
  timezone at parse time, which is the expected behaviour for a
  single-market (Bangkok) app and matches the rest of the app's
  assumptions.

**Verification**

After the fix, deal 1's pickup window correctly shows "05:30 – 08:00"
instead of "22:30 – 01:00", and the "Pickup today" filter correctly
includes/excludes deals — including the overnight-window store Chao
Phraya Sushi — based on the real Bangkok-local time, confirmed with
screenshots before and after the fix.

Commit: `[RES-106] Convert pickup window to local time so displayed hours and "today" checks match the store's actual timezone`

### RES-107 · Deep link opens to a crash

TODO

---

## Part B — Features

### F-1 · Live flash-sale countdowns

TODO

### F-2 · Impression tracking

TODO

### F-3 · Stock reservations with optimistic UI

TODO

---

## Part C — AI usage log

**Tools used:** Claude (this conversation), for reading and explaining
the codebase, forming and testing root-cause hypotheses, drafting
fixes, and writing this document. All fixes were typed/pasted and run
by me on the emulator; I verified every claimed behaviour against the
actual console logs before accepting it.

**Examples where an AI suggestion was wrong or misleading:**

1. **RES-103 — guessed the wrong mechanism before seeing the actual
   code.** Before I had shared `deal_details_controller.dart`, Claude
   looked only at two console-log screenshots and hypothesised that
   `re-checking availability` was firing from an automatic `Timer`
   polling every ~3 seconds, and proposed a test protocol built around
   that theory. I had actually noticed from using the app that the
   check seemed tied to the "+"/add action, not a timer, and said so.
   Claude's first response doubled down on the timer theory and asked
   me to run a "don't touch anything for 10 seconds" test to disprove
   me — only once I supplied the real `cart_service.dart` and
   `deal_details_controller.dart` did it find the actual cause (an
   `ever()` listener on `cartService.itemCount` that was never
   disposed). Lesson: I now push back and ask for the real source file
   whenever an AI's diagnosis is based only on log screenshots rather
   than code, since a plausible-looking pattern in a log can point to
   the wrong mechanism entirely.

2. **RES-104 — test instructions that could not actually reproduce
   the race.** To make the intermittent race condition in
   `home_controller.dart` reproducible for testing, Claude added an
   artificial delay before the `loadMore` network call and told me to
   "refresh as soon as you see `GET /deals?page=2` printed". That
   turned out to be backwards: the `GET` line only prints *after* the
   artificial delay elapses (because the delay was placed before the
   call that logs it), so by the time that line appeared on screen the
   race window had already closed and every test came back negative
   with no explanation of why. I ran the steps exactly as instructed
   twice and both times got "no failure", which is misleading — it
   looked like the fix worked when in fact the race was never
   triggered at all. It was only once I reported the confusing "no
   duplicates, but also no failure without the fix" results, twice,
   that Claude figured out the logging order was the problem and
   added an explicit `>>> loadMore START <<<` marker printed *before*
   the delay so I had a real, visible cue for when the race window
   actually opened. Lesson: when a test is built around timing, I now
   ask explicitly what a "before" run should look like versus an
   "after" run before spending time running it, instead of assuming
   the AI got the sequencing right.

---

## Design questions

**Q1. In this codebase, what is the difference between a
`GetxController`'s lifecycle and a widget `State`'s lifecycle? Name
one bug from Part A that exists because of confusion between the
two.**

A widget `State` is tied to a specific widget instance in the tree:
`initState` runs when it is first inserted and `dispose` runs when
Flutter removes it, which can happen often (e.g. every time a screen
is popped and re-pushed) and is driven by the widget tree itself. A
`GetxController` is tied to GetX's own dependency-injection lifecycle:
`onInit` runs once when the controller is put into memory (typically
when its bound route is first pushed, via `Get.put`/`GetView`) and
`onClose` runs when GetX decides to dispose it — which, depending on
how the controller was registered, may not happen at the same moment
its screen leaves the widget tree, and for a `GetxService` may not
happen for the entire life of the app. The two lifecycles look
similar (an "init" hook and a "cleanup" hook) but are driven by
completely different systems, and it is easy to assume they line up
one-to-one when they do not. RES-102 was a bug on the `State` side: a
`Timer` created in `initState` was never cancelled in `dispose`.
RES-103 was the GetX-side twin of the same mistake: an `ever()`
listener (a `Worker`) created in `onInit` was never disposed in
`onClose`, and because it was registered against a long-lived
`GetxService` (`CartService`), the leak was worse than a typical
`State` leak — every past visit to the deal details screen left
behind a permanent listener for the rest of the session, not just
until the next garbage collection.

**Q2. When does wrapping a large subtree in a single `Obx` hurt you?
How do you decide how tightly to scope reactivity?**

`Obx` subscribes to every observable read inside its `build` callback
during the build, and rebuilds the *entire* widget subtree passed to
it whenever any one of those observables changes — it has no
granularity below "this whole callback runs again". Wrapping a large
subtree in one `Obx` means a change to a single reactive value (say,
one deal's `quantityLeft` in a 100-item list) forces every widget in
that subtree to rebuild, even ones that read completely unrelated
data or no reactive data at all. On a list screen this shows up
exactly like RES-105's symptoms: frames drop during scroll because
far more of the tree rebuilds than actually changed, and if any of
that rebuilding work allocates images or other objects each time
(e.g. re-decoding a network image widget instead of reusing a cached
one), memory pressure grows with it. The way I'd decide how tightly to
scope reactivity is to ask, for each `Obx`, "what is the smallest
widget whose *output* actually depends on this observable?" and wrap
only that — typically a single `Text`, `Icon`, or small row, not a
whole card or list. If several unrelated pieces of state affect
different small parts of the same widget, that usually means several
small `Obx`s side by side rather than one `Obx` around all of them.

**Q3. How would you write an automated test that would have caught
RES-106 before release? What (if anything) would you change in the
code to make such a test possible?**

I'd write a unit test directly against `PickupWindowModel.fromJson`
that feeds it a known UTC ISO-8601 string (matching the shape
`fake_api_service.dart` actually sends) and asserts on `label` and
`isToday`, e.g. parsing `"2026-09-29T22:30:00Z"` and expecting
`label` to read `05:30 – ...` and not `22:30 – ...` once converted to
Bangkok time. The tricky part is that both `label` and, especially,
`isToday` depend on the *device's current time and timezone*
(`DateTime.now()`), which makes the test's outcome depend on when and
where it happens to run — exactly the kind of hidden dependency that
let this bug ship in the first place. To make that testable reliably,
I'd change two things in the code: (1) fix the device's timezone for
the test using Dart's `Intl`/timezone setup, or simpler, run assertions
that don't depend on the *current* date at all — construct both the
input and an explicit "now" and pass "now" in rather than reading it
live; and (2) refactor `isToday` to accept an optional `DateTime now`
parameter (defaulting to `DateTime.now()` in production) so a test can
inject a fixed instant instead of being at the mercy of whatever day
the test happens to run on. With that small seam in place, a table of
cases — a normal daytime window, an overnight window like Chao Phraya
Sushi's 22:00–01:00, and a window that starts just before/after local
midnight — could all be asserted deterministically without touching
the emulator at all.

---

## Time spent and next steps

- **Time spent so far:** roughly 10–12 hours across four days —
  environment setup (Flutter/Java/Android SDK version issues) took
  longer than expected, around 2–3 hours; RES-102 ~1 hour; RES-101
  ~1.5 hours; RES-106 ~1 hour (including reading `deals.json`/
  `stores.json` to understand the store-local time data); RES-103
  ~1.5 hours (extra time spent because my first hypothesis, prompted
  by an AI misdiagnosis, was wrong — see AI usage log); RES-104 the
  longest single item at ~2.5–3 hours, almost entirely spent getting
  a reliable reproduction for an intermittent race condition; writing
  this `solutions.md` ~1.5 hours.
- **With one more day:** I would attempt RES-107 (deep link crash)
  next, since the requirement is well-defined and likely a missing
  null-check or lookup against the catalog by id rather than a design
  question. After that, F-1 (live flash-sale countdowns) — it is the
  feature most connected to what I already fixed in RES-102/RES-103
  (Timer/Worker lifecycle) and is explicitly the feature the grading
  criteria call out as most valuable to complete fully rather than
  attempt multiple features partially. RES-105 (feed jank/memory)
  would come last since it explicitly requires DevTools before/after
  evidence and is likely to take the longest to properly diagnose
  rather than just patch.
