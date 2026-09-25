# solutions.md — Rescu assessment

Candidate: Chadayu Kerdsanthat
Environment: Windows, Flutter 3.27.0, Java 17, Android emulator (`sdk gphone16k x86 64`)

> Status legend: DONE = fixed and verified, TODO = not started

| Item | Status |
|---|---|
| RES-101 | DONE |
| RES-102 | DONE |
| RES-103 | TODO |
| RES-104 | TODO |
| RES-105 | TODO |
| RES-106 | TODO |
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

TODO

### RES-104 · Duplicate deals in the home feed

TODO

### RES-105 · Home feed is janky and memory keeps climbing

TODO (needs DevTools before/after evidence)

### RES-106 · Wrong pickup times; "Pickup today" filter misses deals

TODO

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

Tools used and for what:

- TODO

Examples where an AI suggestion was wrong or misleading (need at least two; write what really happened):

1. TODO — what the AI suggested, how I noticed it was wrong, what I did instead.
2. TODO

---

## Design questions

**Q1. GetxController lifecycle vs widget State lifecycle**

TODO (RES-102 is an example on the `State` side: a resource created in `initState` must be released in `dispose`. Pair it with a GetX-side bug from Part A.)

**Q2. When does a large `Obx` hurt?**

TODO

**Q3. How would I test for RES-106?**

TODO

---

## Time spent and next steps

- Time spent so far: TODO (be honest, include setup time)
- With one more day: TODO
