# Alarm Clock — PintOS Implementation Guide

## Overview

The alarm clock is **Project 1, Problem 1** in PintOS. The goal is to fix `timer_sleep()` so it does **not busy-wait** — instead the calling thread should block until its wakeup time arrives.

---

## The Problem: Busy-Wait

The current implementation in `devices/timer.c:90`:

```c
void
timer_sleep (int64_t ticks)
{
  int64_t start = timer_ticks ();

  ASSERT (intr_get_level () == INTR_ON);
  while (timer_elapsed (start) < ticks)
    thread_yield ();          /* <-- spins, wasting CPU */
}
```

`thread_yield()` puts the thread back on the ready list immediately. The scheduler will pick it again and again, burning CPU doing nothing useful until enough ticks have passed.

**Goal:** replace the spin loop with a proper block/wake mechanism.

---

## Key Concepts to Understand First

### Timer Interrupt (the heartbeat)

`timer_interrupt()` fires **100 times per second** (TIMER_FREQ = 100). It runs in interrupt context:

```c
static void
timer_interrupt (struct intr_frame *args UNUSED)
{
  ticks++;           /* global tick counter */
  thread_tick ();    /* scheduler bookkeeping */
}
```

This is the only place where time reliably advances. Any wakeup logic must hook in here.

### Thread States

```
THREAD_RUNNING  -> the currently executing thread
THREAD_READY    -> on ready_list, waiting for CPU
THREAD_BLOCKED  -> sleeping/waiting; not on ready_list
THREAD_DYING    -> exiting
```

`thread_block()` moves the current thread to BLOCKED and yields.
`thread_unblock(t)` moves thread `t` from BLOCKED back to READY.

### Interrupt Levels

`intr_disable()` / `intr_set_level()` — used to create atomic sections.
`thread_block()` **requires interrupts to be OFF** when called.
Inside `timer_interrupt` you are already in interrupt context (interrupts off).

---

## Implementation Plan

### Files to modify

| File | What changes |
|------|-------------|
| `threads/thread.h` | Add `wakeup_tick` field to `struct thread` |
| `threads/thread.c` | Initialize `wakeup_tick` in `init_thread()` |
| `devices/timer.c` | Add sleep list, rewrite `timer_sleep()`, update `timer_interrupt()` |

---

## Step-by-Step

### Step 1 — Add `wakeup_tick` to `struct thread`

**File:** `threads/thread.h`

Inside `struct thread`, after the existing fields and before the `magic` member:

```c
struct thread
  {
    /* Owned by thread.c. */
    tid_t tid;
    enum thread_status status;
    char name[16];
    uint8_t *stack;
    int priority;
    struct list_elem allelem;

    /* Shared between thread.c and synch.c. */
    struct list_elem elem;

    /* --- ADD THIS --- */
    int64_t wakeup_tick;   /* tick at which this thread should wake up */

#ifdef USERPROG
    uint32_t *pagedir;
#endif

    unsigned magic;
  };
```

> **Why `int64_t`?** It matches the type of the global `ticks` counter.
> **Why before `magic`?** The `magic` sentinel must stay last; it detects stack overflow.

---

### Step 2 — Initialize `wakeup_tick` in `init_thread()`

**File:** `threads/thread.c`, function `init_thread()` (~line 452)

After `memset(t, 0, sizeof *t)` the field is already zeroed, so no extra line is strictly required — but you can add it for clarity:

```c
t->wakeup_tick = 0;
```

---

### Step 3 — Add a sleep list in `timer.c`

**File:** `devices/timer.c`, at the top (after the existing static variables)

```c
/* List of sleeping threads, ordered by wakeup_tick ascending. */
static struct list sleep_list;
```

Then initialize it in `timer_init()`:

```c
void
timer_init (void)
{
  list_init (&sleep_list);          /* ADD THIS LINE */
  pit_configure_channel (0, 2, TIMER_FREQ);
  intr_register_ext (0x20, timer_interrupt, "8254 Timer");
}
```

---

### Step 4 — Rewrite `timer_sleep()`

**File:** `devices/timer.c`, replace the existing function body:

```c
void
timer_sleep (int64_t ticks)
{
  int64_t wakeup = timer_ticks () + ticks;
  struct thread *cur = thread_current ();
  enum intr_level old_level;

  ASSERT (intr_get_level () == INTR_ON);

  /* Disable interrupts to safely insert into sleep_list and block. */
  old_level = intr_disable ();

  cur->wakeup_tick = wakeup;

  /* Insert in sorted order (earliest wakeup first). */
  list_insert_ordered (&sleep_list, &cur->elem,
                       thread_wakeup_less, NULL);

  thread_block ();   /* releases CPU; re-enables interrupts inside schedule() */

  intr_set_level (old_level);
}
```

You also need the comparator function. Add it as a static function near the top of `timer.c`:

```c
/* list_less_func for sleep_list: compares threads by wakeup_tick. */
static bool
thread_wakeup_less (const struct list_elem *a,
                    const struct list_elem *b,
                    void *aux UNUSED)
{
  const struct thread *ta = list_entry (a, struct thread, elem);
  const struct thread *tb = list_entry (b, struct thread, elem);
  return ta->wakeup_tick < tb->wakeup_tick;
}
```

> **Why sorted insertion?** So that `timer_interrupt` only needs to check the front of the list and stop early once it sees a thread that isn't ready yet.

---

### Step 5 — Wake threads in `timer_interrupt()`

**File:** `devices/timer.c`, update the interrupt handler:

```c
static void
timer_interrupt (struct intr_frame *args UNUSED)
{
  ticks++;
  thread_tick ();

  /* Wake any threads whose wakeup_tick has been reached. */
  while (!list_empty (&sleep_list))
    {
      struct thread *t = list_entry (list_front (&sleep_list),
                                     struct thread, elem);
      if (t->wakeup_tick > ticks)
        break;                        /* list is sorted; no need to look further */

      list_pop_front (&sleep_list);
      thread_unblock (t);
    }
}
```

> **Why is it safe to call `thread_unblock` here?**
> `timer_interrupt` runs with interrupts disabled (it IS the interrupt handler). `thread_unblock` only needs interrupts off, which is already the case.

---

## Data Flow Summary

```
timer_sleep(N) called by thread T
  |
  |-- compute wakeup_tick = current_ticks + N
  |-- disable interrupts
  |-- insert T into sleep_list (sorted by wakeup_tick)
  |-- call thread_block()
  |     |-- T.status = THREAD_BLOCKED
  |     |-- schedule() -> picks next thread
  |
  ... time passes, other threads run ...
  |
timer_interrupt fires each tick
  |-- ticks++
  |-- peek sleep_list front
  |-- if front->wakeup_tick <= ticks:
  |     pop it, call thread_unblock(T)
  |     T.status = THREAD_READY, T added to ready_list
  |-- repeat for all expired entries
  |
T gets scheduled again, returns from thread_block()
timer_sleep() returns to caller
```

---

## Why This is Correct

### No busy-wait
The sleeping thread is `THREAD_BLOCKED`. It never appears in `ready_list`, so the scheduler never picks it. CPU is free for other threads.

### No race condition
The section between setting `wakeup_tick` and calling `thread_block()` runs with interrupts disabled. This prevents `timer_interrupt` from firing in the middle and potentially missing a wakeup for a thread that hasn't been fully added to `sleep_list` yet.

### Correct list usage
`struct thread` already has an `elem` field. When a thread is sleeping it is **not** on `ready_list`, so reusing `elem` for `sleep_list` is safe (the comment in `thread.h:77` explicitly says elem is dual-purpose between ready queue and semaphore waiters — our usage is the same pattern).

---

## Common Mistakes to Avoid

| Mistake | Why it breaks |
|---------|--------------|
| Calling `thread_block()` with interrupts ON | `thread_block()` asserts `INTR_OFF`; will panic |
| Not disabling interrupts before inserting into `sleep_list` | Timer interrupt could fire between the insert and the block, see the thread, call `thread_unblock` on a non-blocked thread — assertion failure |
| Using `thread_yield()` instead of `thread_block()` | Thread stays READY, still gets scheduled — back to busy-waiting |
| Forgetting to initialize `sleep_list` in `timer_init` | `list_empty()` on uninitialized list is undefined behavior / crash |
| Not using sorted insert | Must scan entire list every tick instead of early-exit |

---

## Verifying the Implementation

PintOS includes tests under `tests/threads/`. The relevant ones are:

```
alarm-single       - one thread sleeps for a fixed number of ticks
alarm-multiple     - several threads sleep for different durations
alarm-simultaneous - threads wake at the same tick
alarm-priority     - combined with priority scheduling (later project)
alarm-zero         - timer_sleep(0) should return immediately
alarm-negative     - timer_sleep with negative value (should not sleep)
```

Build and run from inside the `threads/` directory:

```bash
cd pintos/src/threads
make
cd build
pintos -- run alarm-multiple
```

Or run all alarm tests:

```bash
make check
```

A passing `alarm-multiple` output looks like:

```
(alarm-multiple) begin
(alarm-multiple) Creating 5 threads to sleep 7 times each.
(alarm-multiple) Thread 0 sleeps 10 ticks each time, ...
...
(alarm-multiple) end
```

---

## Minimal Diff Summary

```
threads/thread.h
  + int64_t wakeup_tick;           (in struct thread)

devices/timer.c
  + static struct list sleep_list;
  + static bool thread_wakeup_less (...);

  timer_init():
  + list_init (&sleep_list);

  timer_sleep():
  - while (timer_elapsed(start) < ticks) thread_yield();
  + cur->wakeup_tick = wakeup;
  + list_insert_ordered (&sleep_list, &cur->elem, thread_wakeup_less, NULL);
  + thread_block ();

  timer_interrupt():
  + while (!list_empty(&sleep_list)) {
  +   check front->wakeup_tick; if expired, pop and thread_unblock
  + }
```

Total change: ~25 lines of new code, ~3 lines removed.
