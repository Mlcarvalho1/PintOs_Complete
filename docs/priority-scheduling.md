# Priority Scheduling — PintOS Implementation Guide

This covers **Project 1, Problems 1-2 and 1-3**:
- **1-2 Priority Scheduler**: always run the highest-priority ready thread.
- **1-3 Priority Donation**: prevent priority inversion when a high-priority thread is blocked on a lock held by a low-priority thread.

---

## Background: What Exists Today

The current scheduler in `threads/thread.c` uses a plain FIFO `ready_list`. `next_thread_to_run()` just pops the front:

```c
return list_entry (list_pop_front (&ready_list), struct thread, elem);
```

There is no priority ordering anywhere. Every thread has a `priority` field, but it is never consulted during scheduling.

---

## Part 1 — Priority Scheduler (Problem 1-2)

### Goal

- The ready list must always yield the **highest-priority** thread next.
- When a thread is unblocked (or newly created) with higher priority than the currently running thread, the running thread must **yield immediately**.
- `thread_set_priority()` must yield if a higher-priority thread is now ready.

### Rule

> At all times, the running thread must have the highest priority among all ready/running threads.

---

### Changes for Part 1

#### 1. Keep `ready_list` sorted — `thread_unblock()` and `thread_yield()`

**File:** `threads/thread.c`

Add a comparator (put it near the top of the file, before `thread_unblock`):

```c
/* Returns true if thread A has strictly higher priority than thread B. */
static bool
thread_priority_greater (const struct list_elem *a,
                         const struct list_elem *b,
                         void *aux UNUSED)
{
  return list_entry (a, struct thread, elem)->priority
       > list_entry (b, struct thread, elem)->priority;
}
```

Then in **`thread_unblock()`**, replace `list_push_back` with sorted insert:

```c
/* Before (FIFO — wrong): */
list_push_back (&ready_list, &t->elem);

/* After (priority-ordered): */
list_insert_ordered (&ready_list, &t->elem, thread_priority_greater, NULL);
```

Do the same in **`thread_yield()`**:

```c
/* Before: */
list_push_back (&ready_list, &cur->elem);

/* After: */
list_insert_ordered (&ready_list, &cur->elem, thread_priority_greater, NULL);
```

`next_thread_to_run()` already pops the front — no change needed there.

#### 2. Yield on unblock — `thread_unblock()`

When a thread with higher priority is unblocked, the current thread must yield. However, `thread_unblock()` can be called from interrupt context (e.g., from `sema_up` inside an interrupt handler), so you cannot call `thread_yield()` directly. Use `intr_yield_on_return()` in that case:

```c
void
thread_unblock (struct thread *t)
{
  enum intr_level old_level;

  ASSERT (is_thread (t));

  old_level = intr_disable ();
  ASSERT (t->status == THREAD_BLOCKED);
  list_insert_ordered (&ready_list, &t->elem, thread_priority_greater, NULL);
  t->status = THREAD_READY;
  intr_set_level (old_level);

  /* Yield if the newly unblocked thread has higher priority. */
  if (!intr_context () &&
      t->priority > thread_current ()->priority)
    thread_yield ();
}
```

#### 3. Yield on `thread_set_priority()`

**File:** `threads/thread.c`

```c
void
thread_set_priority (int new_priority)
{
  thread_current ()->priority = new_priority;

  /* If our priority dropped, a ready thread may now be higher. */
  thread_yield_if_not_highest ();
}
```

Add a helper (or inline it):

```c
/* Yields the CPU if a higher-priority thread is ready. */
void
thread_yield_if_not_highest (void)
{
  if (!list_empty (&ready_list))
    {
      struct thread *front = list_entry (list_front (&ready_list),
                                         struct thread, elem);
      if (front->priority > thread_current ()->priority)
        thread_yield ();
    }
}
```

#### 4. `sema_up()` — wake the highest-priority waiter

**File:** `threads/synch.c`

Currently `sema_up` pops the front of `waiters` (FIFO). Change the waiter list to be priority-ordered, or sort before picking:

```c
void
sema_up (struct semaphore *sema)
{
  enum intr_level old_level;

  ASSERT (sema != NULL);

  old_level = intr_disable ();
  if (!list_empty (&sema->waiters))
    {
      /* Sort in case priorities have changed since the thread blocked. */
      list_sort (&sema->waiters, thread_priority_greater, NULL);
      thread_unblock (list_entry (list_pop_front (&sema->waiters),
                                  struct thread, elem));
    }
  sema->value++;
  intr_set_level (old_level);
}
```

> `thread_priority_greater` needs to be visible here. Either move it to `thread.h` / `thread.c` and expose it, or duplicate the comparator in `synch.c`.

#### 5. `cond_signal()` — wake the highest-priority waiter

**File:** `threads/synch.c`

`cond->waiters` is a list of `semaphore_elem`, each wrapping a one-thread semaphore. To pick the highest-priority waiter, compare the thread inside each semaphore's waiter list:

```c
/* Comparator for cond_signal: compares the priority of the single
   thread waiting on each semaphore_elem. */
static bool
sema_elem_priority_greater (const struct list_elem *a,
                             const struct list_elem *b,
                             void *aux UNUSED)
{
  const struct semaphore_elem *sa = list_entry (a, struct semaphore_elem, elem);
  const struct semaphore_elem *sb = list_entry (b, struct semaphore_elem, elem);

  /* Each semaphore has exactly one waiter when used by cond_wait. */
  const struct thread *ta = list_entry (list_front (&sa->semaphore.waiters),
                                        struct thread, elem);
  const struct thread *tb = list_entry (list_front (&sb->semaphore.waiters),
                                        struct thread, elem);
  return ta->priority > tb->priority;
}

void
cond_signal (struct condition *cond, struct lock *lock UNUSED)
{
  ASSERT (cond != NULL);
  ASSERT (lock != NULL);
  ASSERT (!intr_context ());
  ASSERT (lock_held_by_current_thread (lock));

  if (!list_empty (&cond->waiters))
    {
      list_sort (&cond->waiters, sema_elem_priority_greater, NULL);
      sema_up (&list_entry (list_pop_front (&cond->waiters),
                            struct semaphore_elem, elem)->semaphore);
    }
}
```

---

## Part 2 — Priority Donation (Problem 1-3)

### The Problem: Priority Inversion

```
L (priority 10)  holds lock A
M (priority 20)  running
H (priority 30)  wants lock A  --> blocked, waiting for L
```

L is now the bottleneck for H, but L has lower priority than M. M will keep running while H waits. This is **priority inversion**.

**Fix:** when H blocks on a lock held by L, temporarily **donate** H's priority to L so L can finish and release the lock.

### Nested Donation

```
L holds lock A
M holds lock B, waiting on lock A
H waiting on lock B
```

H donates to M (direct). M is waiting on A held by L, so H's priority must also be donated to L (nested).

### Multiple Donations

A thread can hold several locks at once. Its effective priority is:

```
effective_priority = max(base_priority, max priority of all threads waiting on any lock it holds)
```

---

### New Fields in `struct thread`

**File:** `threads/thread.h`

```c
struct thread
  {
    /* ... existing fields ... */

    int priority;                       /* Effective (possibly donated) priority. */
    int base_priority;                  /* Original priority before any donation. */

    struct list donations;              /* Threads donating priority to this thread.
                                           Each entry is a donor thread's donate_elem. */
    struct list_elem donate_elem;       /* Element for donor's thread's donations list. */

    struct lock *waiting_on_lock;       /* Lock this thread is blocked waiting for,
                                           or NULL if not waiting. */

    /* ... existing magic ... */
  };
```

Initialize in `init_thread()`:

```c
t->base_priority    = priority;
t->waiting_on_lock  = NULL;
list_init (&t->donations);
```

---

### Helper: Recompute Effective Priority

This function is called whenever donations may have changed:

```c
/* Recalculates thread T's effective priority as
   max(base_priority, highest donor priority). */
static void
thread_refresh_priority (struct thread *t)
{
  int eff = t->base_priority;

  if (!list_empty (&t->donations))
    {
      list_sort (&t->donations, thread_priority_greater_donate, NULL);
      int donor_max = list_entry (list_front (&t->donations),
                                  struct thread, donate_elem)->priority;
      if (donor_max > eff)
        eff = donor_max;
    }

  t->priority = eff;
}
```

(You need a comparator using `donate_elem` instead of `elem` — same logic, different field.)

---

### Changes for Part 2

#### 1. `lock_acquire()` — donate before blocking

**File:** `threads/synch.c`

```c
void
lock_acquire (struct lock *lock)
{
  struct thread *cur = thread_current ();

  ASSERT (lock != NULL);
  ASSERT (!intr_context ());
  ASSERT (!lock_held_by_current_thread (lock));

  if (lock->holder != NULL)
    {
      /* Record what we are waiting for. */
      cur->waiting_on_lock = lock;

      /* Donate priority up the chain (handles nesting). */
      struct thread *t = lock->holder;
      while (t != NULL && t->priority < cur->priority)
        {
          t->priority = cur->priority;           /* donate */
          /* Add cur to t's donation list (avoid duplicates). */
          list_insert_ordered (&t->donations, &cur->donate_elem,
                               thread_priority_greater_donate, NULL);
          /* Walk up: if t is itself waiting on a lock, continue. */
          if (t->waiting_on_lock != NULL)
            t = t->waiting_on_lock->holder;
          else
            break;
        }
    }

  sema_down (&lock->semaphore);
  cur->waiting_on_lock = NULL;
  lock->holder = cur;
}
```

> **Depth limit:** in practice, PintOS limits donation nesting to 8 levels. A for-loop with a counter is safer than unbounded while.

#### 2. `lock_release()` — remove donations from this lock

**File:** `threads/synch.c`

```c
void
lock_release (struct lock *lock)
{
  struct thread *cur = thread_current ();

  ASSERT (lock != NULL);
  ASSERT (lock_held_by_current_thread (lock));

  /* Remove all donations that were waiting on this specific lock. */
  struct list_elem *e = list_begin (&cur->donations);
  while (e != list_end (&cur->donations))
    {
      struct thread *donor = list_entry (e, struct thread, donate_elem);
      struct list_elem *next = list_next (e);
      if (donor->waiting_on_lock == lock)
        list_remove (e);
      e = next;
    }

  /* Recompute effective priority now that some donations are gone. */
  thread_refresh_priority (cur);

  lock->holder = NULL;
  sema_up (&lock->semaphore);

  /* Yield if a higher-priority thread was just unblocked. */
  thread_yield_if_not_highest ();
}
```

#### 3. `thread_set_priority()` — respect donation

```c
void
thread_set_priority (int new_priority)
{
  struct thread *cur = thread_current ();
  cur->base_priority = new_priority;
  thread_refresh_priority (cur);          /* won't drop below donated priority */
  thread_yield_if_not_highest ();
}
```

---

## Data Flow: Lock Acquisition with Donation

```
H calls lock_acquire(A)
  |
  A->holder == L  (L holds the lock)
  |
  H->waiting_on_lock = A
  H donates to L: L->priority = max(L->priority, H->priority)
  add H to L->donations
  |
  Is L waiting on another lock B?
    yes: L->waiting_on_lock = B, B->holder = M
      M->priority = max(M->priority, H->priority)  [nested donation]
      ...continue up chain (max 8 levels)
  |
  sema_down(A->semaphore)  --> H blocks
  |
  ... L runs (at donated priority), finishes work ...
  |
L calls lock_release(A)
  |
  Remove H from L->donations (H was waiting on A)
  thread_refresh_priority(L):
    L->priority = max(L->base_priority, remaining donors)
  sema_up(A->semaphore)  --> H unblocked
  |
H resumes, lock_acquire returns
H->waiting_on_lock = NULL
A->holder = H
```

---

## Files Modified Summary

| File | Changes |
|------|---------|
| `threads/thread.h` | Add `base_priority`, `donations`, `donate_elem`, `waiting_on_lock` |
| `threads/thread.c` | `init_thread`, `thread_unblock`, `thread_yield`, `thread_set_priority`, add `thread_refresh_priority`, `thread_yield_if_not_highest`, `thread_priority_greater` |
| `threads/synch.c` | `sema_up` (pick highest waiter), `lock_acquire` (donate), `lock_release` (remove donation, refresh), `cond_signal` (pick highest waiter) |

---

## Common Mistakes

| Mistake | Consequence |
|---------|------------|
| Donating but forgetting to remove on release | Priority stays inflated forever |
| Only doing one level of donation (no chain traversal) | Nested inversion not resolved |
| Using `priority` as base when `thread_set_priority` is called | Setting priority while donated overwrites the donation |
| Not yielding after unblocking a higher-priority thread | Violates the preemption invariant; priority-related tests fail |
| Not sorting `sema->waiters` before popping in `sema_up` | Priorities may have changed since threads blocked; FIFO wakeup is wrong |
| Inserting into `ready_list` with `list_push_back` instead of sorted insert | Scheduler ignores priority entirely |

---

## Tests to Run

```
priority-preempt     - newly created high-priority thread runs first
priority-fifo        - equal-priority threads run in FIFO order
priority-donate-one  - basic single donation
priority-donate-multiple - one thread holds two locks, two donors
priority-donate-nest - nested donation (3 threads, 2 locks)
priority-donate-sema - donation through semaphore
priority-donate-lower - donor sets its own priority lower after donating
priority-condvar     - cond_signal wakes highest-priority waiter
priority-sema        - sema_up wakes highest-priority waiter
```

Run with:

```bash
cd pintos/src/threads/build
pintos -- run priority-donate-nest
```

Or all at once:

```bash
make check
```
