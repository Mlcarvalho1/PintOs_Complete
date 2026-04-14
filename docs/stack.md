# Argument Passing (Stack Setup) — PintOS Implementation Guide

This covers **Project 2, Problem 2-1**: passing command-line arguments to a user process by building the initial stack before the process starts executing.

---

## Background: What Exists Today

`process_execute()` receives a string like `"/bin/ls -l foo bar"` and passes it straight to `thread_create()` and `load()`. Neither strips the arguments from the filename. The initial stack (`setup_stack`) just points `esp` at `PHYS_BASE` with no arguments on it.

The `feat/stack` branch adds a skeleton `argument_stack()` and splits the filename in `start_process()`, but the skeleton has several bugs (listed in [Common Mistakes](#common-mistakes)).

---

## Goal

Given a command line like:

```
/bin/ls -l foo bar
```

The kernel must build the following layout on the user stack **before** jumping to `main()`:

```
Address      Content                          Type
---------    -------                          ----
0xbfffffed   "/bin/ls\0"                      char[]  argv[0] string
0xbfffffea   "-l\0"                           char[]  argv[1] string
0xbfffffe7   "foo\0"                          char[]  argv[2] string
0xbfffffe4   "bar\0"                          char[]  argv[3] string
0xbfffffe0   (0-3 padding bytes to align)     uint8_t
0xbfffffdc   0x00000000                       char*   argv[4] = NULL
0xbfffffd8   0xbfffffe4                       char*   argv[3] = &"bar"
0xbfffffd4   0xbfffffe7                       char*   argv[2] = &"foo"
0xbfffffd0   0xbfffffea                       char*   argv[1] = &"-l"
0xbfffffcc   0xbfffffed                       char*   argv[0] = &"/bin/ls"
0xbfffffc8   0xbfffffcc                       char**  argv  (= &argv[0])
0xbfffffc4   4                                int     argc
0xbfffffc0   0x00000000                       void*   fake return address
              ^
              esp points here
```

The C runtime expects to find `argc`, `argv`, and a fake return address in exactly this layout when `main(int argc, char *argv[])` is called.

---

## Files to Modify

| File | Changes |
|------|---------|
| `userprog/process.c` | Fix `process_execute()`, fix `start_process()`, implement `argument_stack()`, revert `setup_stack()` |

---

## Step-by-Step

### Step 1 — Fix `process_execute()`: use only filename as thread name

**File:** `userprog/process.c`, function `process_execute()`

`thread_create` receives the thread name. If you pass the whole command line, the thread name becomes `"/bin/ls -l foo bar"`. Use only the first token:

```c
tid_t
process_execute (const char *file_name)
{
  char *fn_copy;
  tid_t tid;

  fn_copy = palloc_get_page (0);
  if (fn_copy == NULL)
    return TID_ERROR;
  strlcpy (fn_copy, file_name, PGSIZE);

  /* Extract just the executable name for the thread name.
     Use a temporary buffer so we don't modify fn_copy. */
  char name_buf[NAME_MAX + 2];
  char *save_ptr;
  strlcpy (name_buf, file_name, sizeof name_buf);
  char *exec_name = strtok_r (name_buf, " ", &save_ptr);

  tid = thread_create (exec_name, PRI_DEFAULT, start_process, fn_copy);
  if (tid == TID_ERROR)
    palloc_free_page (fn_copy);
  return tid;
}
```

---

### Step 2 — Fix `start_process()`: correct pointer freed and args passed

**File:** `userprog/process.c`, function `start_process()`

The branch code calls `strtok_r(entry, " ", &args)` which puts the filename at `file_name` and the rest at `args`. But then it does `palloc_free_page(file_name)`, which frees the *interior* of the `fn_copy` page — the correct pointer to free is `entry` (the page start).

```c
static void
start_process (void *file_name_)
{
  char *entry = file_name_;        /* points to the palloc'd page */
  char *args;

  /* Split "prog arg1 arg2" -> file_name = "prog", args -> "arg1 arg2" */
  char *file_name = strtok_r (entry, " ", &args);

  struct intr_frame if_;
  bool success;

  memset (&if_, 0, sizeof if_);
  if_.gs = if_.fs = if_.es = if_.ds = if_.ss = SEL_UDSEG;
  if_.cs = SEL_UCSEG;
  if_.eflags = FLAG_IF | FLAG_MBS;
  success = load (file_name, &if_.eip, &if_.esp);

  /* Free the page using the original pointer (page start), not file_name. */
  palloc_free_page (entry);

  if (!success)
    thread_exit ();

  /* Build the argument stack. */
  argument_stack (args, &if_.esp);

  /* Uncomment to debug the stack layout: */
  /* hex_dump (if_.esp, if_.esp, PHYS_BASE - if_.esp, true); */

  asm volatile ("movl %0, %%esp; jmp intr_exit" : : "g" (&if_) : "memory");
  NOT_REACHED ();
}
```

---

### Step 3 — Revert `setup_stack()`: esp starts at PHYS_BASE

**File:** `userprog/process.c`, function `setup_stack()`

`argument_stack()` will decrement `esp` itself. `setup_stack` must leave `esp = PHYS_BASE` (pointing just past the top of the user address space):

```c
static bool
setup_stack (void **esp)
{
  uint8_t *kpage;
  bool success = false;

  kpage = palloc_get_page (PAL_USER | PAL_ZERO);
  if (kpage != NULL)
    {
      success = install_page (((uint8_t *) PHYS_BASE) - PGSIZE, kpage, true);
      if (success)
        *esp = PHYS_BASE;   /* argument_stack() will push from here downward */
      else
        palloc_free_page (kpage);
    }
  return success;
}
```

> The `PHYS_BASE - 12` offset that was added was wrong — `argument_stack` handles all stack construction.

---

### Step 4 — Implement `argument_stack()`

**File:** `userprog/process.c`

PintOS limits command lines to one page (4096 bytes) and arguments to 128. Use fixed-size arrays — never `realloc` in kernel code (no `malloc`/`free` for kernel data in PintOS).

```c
#define ARGS_MAX 128

void
argument_stack (const char *args, void **esp)
{
  char *argv[ARGS_MAX];
  int argc = 0;

  /* --- Phase 1: copy each argument string onto the stack --- */

  /* We need to tokenize args. Work on a local copy since strtok_r modifies
     the string. args points into the palloc'd page which was already freed —
     use a local buffer. Actually: args is the remainder after strtok_r in
     start_process, which is still valid at this point. But strtok_r will
     modify it. That is fine because the page is freed after argument_stack
     returns only if we hold it — but here entry was freed before this call.
     Solution: receive args as a separate copy, or work before freeing entry.
     (See note in start_process above — call argument_stack BEFORE
      palloc_free_page.) */

  char buf[PGSIZE];
  strlcpy (buf, args != NULL ? args : "", sizeof buf);

  char *token, *save_ptr;
  for (token = strtok_r (buf, " ", &save_ptr);
       token != NULL && argc < ARGS_MAX;
       token = strtok_r (NULL, " ", &save_ptr))
    {
      size_t len = strlen (token) + 1;   /* include '\0' */
      *esp -= len;
      memcpy (*esp, token, len);
      argv[argc++] = *esp;               /* save pointer to this string */
    }

  /* --- Phase 2: word-align esp to a 4-byte boundary --- */
  uintptr_t addr = (uintptr_t) *esp;
  uintptr_t aligned = addr & ~(uintptr_t) 3;
  if (aligned < addr)
    {
      *esp = (void *) aligned;
      /* zero the padding bytes */
      memset (*esp, 0, addr - aligned);
    }

  /* --- Phase 3: push argv[argc] = NULL sentinel --- */
  *esp -= sizeof (char *);
  *(char **) *esp = NULL;

  /* --- Phase 4: push argv[i] pointers, last to first --- */
  for (int i = argc - 1; i >= 0; i--)
    {
      *esp -= sizeof (char *);
      *(char **) *esp = argv[i];
    }

  /* --- Phase 5: push argv (pointer to argv[0]) --- */
  char **argv_ptr = *esp;              /* current esp = &argv[0] on stack */
  *esp -= sizeof (char **);
  *(char ***) *esp = argv_ptr;

  /* --- Phase 6: push argc --- */
  *esp -= sizeof (int);
  *(int *) *esp = argc;

  /* --- Phase 7: push fake return address --- */
  *esp -= sizeof (void *);
  *(void **) *esp = NULL;
}
```

> **Important:** call `argument_stack` **before** `palloc_free_page(entry)` so that `args` (which points into the page) is still valid when the function copies strings into the stack.

Revised ordering in `start_process()`:

```c
  success = load (file_name, &if_.eip, &if_.esp);

  if (!success)
    {
      palloc_free_page (entry);
      thread_exit ();
    }

  argument_stack (args, &if_.esp);   /* must be before palloc_free_page */
  palloc_free_page (entry);

  /* hex_dump (if_.esp, if_.esp, PHYS_BASE - if_.esp, true); */
```

---

## Signature of `argument_stack`

Place the prototype at the top of `process.c` (before `start_process`) or in `process.h` if needed elsewhere:

```c
static void argument_stack (const char *args, void **esp);
```

Making it `static` keeps it file-private (it is only called from `start_process`).

---

## Data Flow Summary

```
process_execute("/bin/ls -l foo bar")
  |
  |-- copy cmdline into palloc'd page (fn_copy)
  |-- extract "ls" as thread name (temp buffer)
  |-- thread_create("ls", ..., start_process, fn_copy)
  |
start_process(fn_copy)
  |
  |-- strtok_r(entry, " ", &args)
  |     file_name = "ls"  (or "/bin/ls")
  |     args      = "-l foo bar"
  |
  |-- load("ls", &eip, &esp)   <-- setup_stack sets esp = PHYS_BASE
  |
  |-- argument_stack("-l foo bar", &esp)
  |     push strings:    "bar\0", "foo\0", "-l\0"   (last to first is fine)
  |     align esp to 4 bytes
  |     push NULL        (argv[argc])
  |     push &"bar"      (argv[2])
  |     push &"foo"      (argv[1])
  |     push &"-l"       (argv[0])
  |     push argv        (= &argv[0])
  |     push argc (3)
  |     push 0           (fake return address)
  |
  |-- palloc_free_page(entry)
  |
  |-- jump to user code via intr_exit
        esp points at the fake return address
        main(argc=3, argv=...) is called correctly
```

---

## Common Mistakes

| Mistake | Consequence |
|---------|------------|
| `char token` instead of `char *token` | Stores a single char, not a pointer; compile error or corruption |
| `realloc` on a stack-allocated VLA (`char *arr[1]`) | `realloc` on a non-heap pointer is undefined behavior; crashes |
| `palloc_free_page(file_name)` instead of `palloc_free_page(entry)` | Frees the middle of the page, not the page start; heap corruption |
| Calling `argument_stack` after `palloc_free_page(entry)` | `args` points into the freed page; reads garbage |
| `*esp = PHYS_BASE - 12` in `setup_stack` | Stack starts 12 bytes off; argument layout is misaligned |
| Pushing argv pointers first-to-last instead of last-to-first | argv[0] ends up pointing at wrong string |
| Forgetting the NULL sentinel after argv pointers | C runtime's argv[argc] is not NULL; undefined behavior |
| Forgetting the fake return address | Stack frame is 4 bytes short; argc/argv are read from wrong offset |
| Not word-aligning before pushing pointers | Misaligned pointer reads on x86 cause subtle bugs (or traps on stricter arches) |
| Passing full cmdline to `thread_create` as name | Thread name becomes `"/bin/ls -l foo bar"` — cosmetic, but also truncated at 16 chars |

---

## Verifying with `hex_dump`

Uncomment the `hex_dump` line in `start_process()`:

```c
hex_dump (if_.esp, if_.esp, PHYS_BASE - if_.esp, true);
```

For `pintos -- run 'ls -l foo bar'`, a correct dump looks like (addresses approximate):

```
bfffffc0                          00 00 00 00 04 00 00 00 |        ........|
bfffffd0  cc ff ff bf ed ff ff bf ea ff ff bf e7 ff ff bf |................|
bfffffe0  e4 ff ff bf 00 00 00 00 62 61 72 00 66 6f 6f 00 |........bar.foo.|
bffffff0  2d 6c 00 6c 73 00 00 00                         |-l.ls...|
```

- `esp` = `0xbfffffc0`
- `0x00000000` at `esp+0` = fake return address
- `0x00000004` at `esp+4` = argc = 4
- Pointer at `esp+8` = argv
- etc.

---

## Tests to Run

```
args-none      - process with no arguments
args-single    - single argument
args-multiple  - multiple arguments
args-many      - stress test with many arguments
args-dbl-space - arguments separated by multiple spaces
```

Build and run from `userprog/`:

```bash
cd pintos/src/userprog
make
cd build
pintos -- run 'args-multiple'
```

Or run all tests:

```bash
make check
```

---

## Minimal Diff Summary

```
userprog/process.c

  process_execute():
  + extract exec name into temp buffer for thread_create

  start_process():
  - char *file_name = file_name_;
  + char *entry = file_name_;
  + char *file_name = strtok_r (entry, " ", &args);
  - palloc_free_page (file_name);         // wrong pointer
  + argument_stack (args, &if_.esp);      // before freeing
  + palloc_free_page (entry);             // correct pointer

  setup_stack():
  - *esp = PHYS_BASE - 12;
  + *esp = PHYS_BASE;

  argument_stack() — new function:
  + tokenize args with strtok_r
  + push strings onto stack, record pointers
  + align to 4 bytes
  + push NULL, argv[], argv, argc, fake return address
```
