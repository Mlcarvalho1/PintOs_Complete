# PintOS Architecture Overview

This document explains how PintOS is structured — what the major subsystems are, how they relate to each other, and where to look when you need to understand or modify something.

---

## Directory Structure

```
pintos/src/
├── threads/        Core kernel: threads, scheduler, synchronization, memory
├── devices/        Hardware drivers: timer, keyboard, serial, disk, VGA
├── lib/            C standard library (kernel + user variants)
│   └── kernel/     Kernel-only data structures: list, hash, bitmap
├── userprog/       User process support: syscalls, ELF loader, address space
├── vm/             Virtual memory (Project 3 — not implemented yet)
├── filesys/        File system (Project 4 — stub exists)
└── examples/       Sample user programs (cat, echo, shell, ...)
```

Everything in `threads/` and `devices/` is what you work with in **Project 1**.

---

## Boot Sequence

Entry point: `threads/init.c` → `pintos_init()`

```
loader.S (assembly)
  └── pintos_init()               threads/init.c
        ├── bss_init()            zero uninitialized data
        ├── thread_init()         set up threading system (no scheduling yet)
        ├── console_init()        enable console output
        ├── palloc_init()         page allocator ready
        ├── malloc_init()         heap allocator ready
        ├── paging_init()         virtual memory mapping
        ├── intr_init()           set up IDT (interrupt descriptor table)
        ├── timer_init()          register timer interrupt (IRQ0)
        ├── kbd_init()            register keyboard interrupt
        ├── thread_start()        create idle thread, enable interrupts
        ├── timer_calibrate()     measure loops/tick for busy_wait
        └── run_actions(argv)     run kernel command-line test/program
```

After `thread_start()` interrupts are on and the scheduler is live.

---

## Threads (`threads/thread.h`, `threads/thread.c`)

### The `struct thread` layout

Each thread lives in its **own 4 KB page**. The `struct thread` is at the **bottom** of the page and the kernel stack grows **downward** from the top:

```
  4 KB ┌─────────────────────┐
       │    kernel stack     │
       │         |           │
       │         v  (grows down)
       │                     │
       │   (unused space)    │
       │                     │
       ├─────────────────────┤  <-- magic (stack overflow sentinel)
       │     :  fields  :    │
       │       name          │
       │       status        │
  0 KB └─────────────────────┘  <-- struct thread starts here
```

`running_thread()` finds the current thread by rounding the stack pointer down to a page boundary.

### Thread States

```
THREAD_RUNNING   Currently on the CPU
THREAD_READY     In ready_list, waiting for CPU time
THREAD_BLOCKED   Waiting for an event (lock, semaphore, sleep, ...)
THREAD_DYING     thread_exit() called, will be freed by next scheduler run
```

State transitions:

```
         thread_yield()          thread_unblock()
RUNNING ─────────────> READY <──────────────────── BLOCKED
   ^                     │                              ^
   │    schedule()       │  schedule() picks it         │
   └─────────────────────┘                              │
                                  thread_block() ───────┘
```

### Key Functions

| Function | What it does |
|----------|-------------|
| `thread_create(name, priority, func, aux)` | Allocates a page, sets up stack frames, adds to ready list |
| `thread_block()` | Sets status=BLOCKED, calls `schedule()`. **Requires interrupts OFF.** |
| `thread_unblock(t)` | Sets t->status=READY, pushes to ready_list |
| `thread_yield()` | Puts current thread back on ready list, calls `schedule()` |
| `thread_tick()` | Called every timer interrupt; enforces TIME_SLICE preemption |
| `thread_exit()` | Marks thread DYING, removes from all_list, calls `schedule()` |

### The Scheduler

`schedule()` in `thread.c`:
1. Calls `next_thread_to_run()` — pops front of `ready_list` (or returns `idle_thread` if empty)
2. Calls `switch_threads(cur, next)` — assembly in `switch.S`, saves/restores registers
3. Calls `thread_schedule_tail()` — marks new thread RUNNING, frees dying threads

The **idle thread** runs when nothing else is ready. It executes `hlt` (halt until interrupt), burning no CPU cycles.

---

## Interrupt System (`threads/interrupt.h`, `threads/interrupt.c`)

### Two kinds of interrupts

| Kind | Examples | Can sleep? |
|------|---------|-----------|
| External (hardware) | Timer (IRQ0), keyboard (IRQ1) | No |
| Internal (software/CPU) | System calls (int 0x30), page fault, divide-by-zero | Yes (syscalls) |

### Interrupt Descriptor Table (IDT)

- `intr_init()` sets up the IDT with stub handlers for all 256 vectors.
- `intr_register_ext(vec, handler, name)` — register a hardware interrupt handler.
- `intr_register_int(vec, dpl, level, handler, name)` — register a software interrupt.

### The interrupt frame (`struct intr_frame`)

When an interrupt fires, the CPU and the stub in `intr-stubs.S` push all registers onto the stack, forming a `struct intr_frame`. The registered handler receives a pointer to this frame.

### `intr_context()`

Returns `true` if code is currently running inside an interrupt handler. Many functions (like `thread_block()`, `lock_acquire()`) assert `!intr_context()` because they cannot sleep in interrupt context.

### `intr_yield_on_return()`

Sets a flag that causes `schedule()` to be called when the interrupt handler returns. Used when a higher-priority thread is unblocked from within an interrupt handler (you can't call `thread_yield()` directly from interrupt context).

---

## Timer (`devices/timer.h`, `devices/timer.c`)

- Uses the **8254 PIT** (Programmable Interval Timer) chip, channel 0.
- `TIMER_FREQ = 100` — fires 100 interrupts per second (every 10 ms).
- `ticks` — global `int64_t` counting interrupts since boot.

### The interrupt handler

```c
timer_interrupt()        // runs 100x/sec, interrupts OFF
  ticks++
  thread_tick()          // enforce TIME_SLICE preemption
  [alarm clock] check sleep_list, unblock ready sleepers
```

### `timer_ticks()` vs `ticks`

`timer_ticks()` disables interrupts to safely read the `ticks` variable. Never read `ticks` directly from outside `timer.c`.

---

## Synchronization (`threads/synch.h`, `threads/synch.c`)

Three primitives are provided:

### Semaphore (`struct semaphore`)

- A non-negative integer counter plus a waiter list.
- `sema_down()` — if value == 0, block. Otherwise decrement.
- `sema_up()` — increment; if waiters exist, unblock one.
- Safe to call from interrupt handlers (`sema_up` only).

### Lock (`struct lock`)

- A binary semaphore (value 0 or 1) with an owner field.
- Same thread must acquire and release.
- **Cannot** be used from interrupt handlers.
- Used as the building block for priority donation (Project 1-3).

### Condition Variable (`struct condition`)

- Used with a lock: `cond_wait(cond, lock)` atomically releases the lock and sleeps.
- `cond_signal(cond, lock)` wakes one waiter.
- `cond_broadcast(cond, lock)` wakes all waiters.
- Internally, each waiter gets its own semaphore (stored as `struct semaphore_elem`).

### When to use what

| Need | Use |
|------|-----|
| Simple producer/consumer counting | Semaphore |
| Mutual exclusion over a critical section | Lock |
| Wait for a condition within a critical section | Condition variable + lock |

---

## Memory Management

### Page Allocator (`threads/palloc.c`)

- Manages physical memory in 4 KB pages.
- Two pools: **kernel pool** and **user pool**.
- `palloc_get_page(flags)` — allocate one page.
- `palloc_free_page(page)` — free one page.
- Each `struct thread` is allocated with `palloc_get_page(PAL_ZERO)`.

### Malloc (`threads/malloc.c`)

- A simple `malloc`/`free` built on top of the page allocator.
- Uses a first-fit block allocator within pages.
- Use for small, variable-size kernel allocations.

### Memory Map

```
PHYS_BASE (0xC0000000 = 3 GB)
  ┌────────────────────────────┐
  │     Kernel virtual space   │  mapped 1:1 to physical RAM
  │  (code, stack, heap, etc.) │
  └────────────────────────────┘
  │     User virtual space     │  (Project 2+, not active in Project 1)
  └────────────────────────────┘
0x00000000
```

In Project 1, all code runs in kernel space. User programs are Project 2.

---

## The `list` Library (`lib/kernel/list.h`)

PintOS uses an **intrusive doubly-linked list** — instead of allocating separate list nodes, each structure embeds a `struct list_elem`:

```c
struct thread {
    struct list_elem elem;      /* used for ready_list / semaphore waiters */
    struct list_elem allelem;   /* used for all_list */
    ...
};
```

Key operations:

| Function | Purpose |
|----------|---------|
| `list_init(list)` | Initialize an empty list |
| `list_push_back(list, elem)` | Append to end |
| `list_push_front(list, elem)` | Prepend to front |
| `list_pop_front(list)` | Remove and return front element |
| `list_front(list)` | Peek at front without removing |
| `list_remove(elem)` | Remove element from whatever list it's in |
| `list_insert_ordered(list, elem, less, aux)` | Insert maintaining sort order |
| `list_sort(list, less, aux)` | Sort the list in-place |
| `list_entry(elem, type, member)` | Get the enclosing struct from a `list_elem` |
| `list_empty(list)` | Check if list has no elements |

`list_entry` is a macro that does pointer arithmetic — it's the backbone of the list API:

```c
struct thread *t = list_entry (list_front (&ready_list), struct thread, elem);
```

---

## Project 1 Subsystem Interactions

```
                      ┌──────────────┐
    100x/sec IRQ0 ──> │ timer_intr   │ ──> thread_tick() ──> intr_yield_on_return()
                      │ (timer.c)    │ ──> check sleep_list ──> thread_unblock()
                      └──────────────┘
                                                    │
                                                    v
  thread_sleep() ──> thread_block() ──> schedule() ──> switch_threads()
                                                    │
                                                    v
                                          next_thread_to_run()
                                          (pops ready_list front)

  lock_acquire() ──> sema_down() ──> thread_block()  (if lock held)
  lock_release() ──> sema_up()   ──> thread_unblock() (wake highest waiter)
```

---

## Key Invariants

1. **Interrupts off when modifying shared state** — `ready_list`, `sleep_list`, `semaphore.waiters`, thread status fields.
2. **`thread_block()` requires interrupts OFF** — it is always called with `intr_disable()` in effect.
3. **No sleeping in interrupt context** — `intr_context()` is asserted false in `sema_down`, `lock_acquire`, `cond_wait`.
4. **Stack overflow detection** — `struct thread.magic` is checked by `thread_current()`. If a kernel function uses too much stack space, this fires first.
5. **`struct thread` must stay small** — the thread struct and kernel stack share one 4 KB page. Keep `struct thread` well under 1 KB.

---

## Navigating the Source for Project 1

| Task | Files to read |
|------|--------------|
| How threads are created | `thread_create()` in `thread.c` |
| How scheduling works | `schedule()`, `next_thread_to_run()` in `thread.c` |
| How the timer fires | `timer_interrupt()` in `timer.c`, `thread_tick()` in `thread.c` |
| How sleeping works (after alarm fix) | `timer_sleep()` in `timer.c` |
| How locks and semaphores work | `synch.c` entirely |
| How interrupts are enabled/disabled | `intr_enable/disable/set_level` in `interrupt.c` |
