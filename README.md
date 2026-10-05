<!---
{
  "id": "b63c3f2e-d7cb-4dd0-ad74-26eb91a7173f",
  "teaches": "Debugging: printf-Debugging with a DBG Macro and -DDEBUG",
  "depends_on": [
    "279a01d6-7696-46b7-9cb7-2c44773ad06b",
    "f5d53abd-0e66-4120-9045-4575182d18fe"
  ],
  "author": "Stephan Bökelmann",
  "first_used": "2026-10-05",
  "keywords": [
    "debugging",
    "printf",
    "stderr",
    "macro",
    "__FILE__",
    "__LINE__",
    "-DDEBUG",
    "variadic macro"
  ]
}
--->

# Debugging: printf-Debugging with a `DBG` Macro and `-DDEBUG`

> In this exercise you will build the simplest debugging tool there is: a `DBG` macro that prints file, line and a value to `stderr`, exists only in the debug build, and vanishes completely from the release binary. You will then find a real bug with it, and learn where this technique stops being useful. No `sudo` is required.

## Introduction

The first thing most programmers do when a value is wrong is to print it. That is not a bad instinct. A `printf` costs nothing to learn and answers the question "what is `x` here?" immediately.

Done naively, it has two problems. The output goes to `stdout` and mixes with the program's real output, and the lines stay in the code forever because nobody dares to delete them. Both are solved by a small macro: it writes to `stderr`, it prepends file and line so you know where the output came from, and it expands to *nothing* unless the program was compiled with `-DDEBUG`. The debug/release configuration from the `make` exercise provides exactly that switch.

### 1.1) Further Readings and Other Sources

* [GCC Manual: Variadic Macros](https://gcc.gnu.org/onlinedocs/cpp/Variadic-Macros.html)
* [GCC Manual: Standard Predefined Macros (`__FILE__`, `__LINE__`, `__func__`)](https://gcc.gnu.org/onlinedocs/cpp/Standard-Predefined-Macros.html)
* [stderr(3) man page](https://man7.org/linux/man-pages/man3/stderr.3.html)

## Tasks

### Task 1: A Program With a Hidden Bug

```bash
mkdir -p ~/dbg-printf && cd ~/dbg-printf
```

```c
// file: stats.c
#include <stdio.h>

double average(const int *v, int n) {
    int sum = 0;
    for (int i = 1; i < n; i++) {      /* bug: starts at 1 */
        sum += v[i];
    }
    return (double)sum / n;
}

int main(void) {
    int data[] = { 10, 20, 30, 40 };
    printf("average = %.2f\n", average(data, 4));
    return 0;
}
```

```bash
gcc -Wall -Wextra -O2 stats.c -o stats && ./stats
```

The correct answer is `25.00`. Do not fix the bug yet.

### Task 2: The `DBG` Macro

Create a header that every file in the project can include:

```c
// file: dbg.h
#ifndef DBG_H
#define DBG_H
#include <stdio.h>

#ifdef DEBUG
#define DBG(fmt, ...) \
    fprintf(stderr, "[%s:%d %s] " fmt "\n", __FILE__, __LINE__, __func__, __VA_ARGS__)
#else
#define DBG(fmt, ...) ((void)0)
#endif

#endif
```

`__VA_ARGS__` makes the macro accept any number of arguments, like `printf`. `((void)0)` is an expression that does nothing, so `DBG(...);` stays valid C even when it is switched off.

### Task 3: Instrument the Loop

```c
#include "dbg.h"
/* inside the loop in average(): */
        DBG("i=%d v[i]=%d sum=%d", i, v[i], sum);
```

```bash
gcc -Wall -Wextra -O0 -DDEBUG stats.c -o stats && ./stats
gcc -Wall -Wextra -O2 stats.c -o stats && ./stats
```

Read the debug lines. Which index is never visited? Fix the bug and rebuild both variants.

### Task 4: Separate the Streams

```bash
gcc -Wall -Wextra -O0 -DDEBUG stats.c -o stats
./stats > out.txt
./stats 2> debug.log
./stats 2>/dev/null
cat out.txt debug.log
```

Because the macro writes to `stderr`, the program's real output is untouched. This is the reason not to use `printf` for debugging output.

### Task 5: Prove the Release Binary Is Clean

```bash
gcc -Wall -Wextra -O2 stats.c -o stats_release
gcc -Wall -Wextra -O0 -DDEBUG stats.c -o stats_debug
strings stats_release | grep -c 'v\[i\]'
strings stats_debug   | grep -c 'v\[i\]'
ls -l stats_release stats_debug
```

The format string does not exist in the release build. The preprocessor removed the whole call before the compiler ever saw it.

### Task 6: Wire It Into the Makefile

Take the `Makefile` with `BUILD_TYPE` from the make exercise (it already adds `-DDEBUG` in the debug configuration) and build both:

```bash
make BUILD_TYPE=debug   && ./build/debug/stats   2>&1 | head -3
make BUILD_TYPE=release && ./build/release/stats
```

### Task 7: Where It Stops Working

Add a timing-sensitive piece of code:

```c
#include <time.h>
/* in main, before the printf: */
    struct timespec t0, t1;
    clock_gettime(CLOCK_MONOTONIC, &t0);
    for (int k = 0; k < 100000; k++) average(data, 4);
    clock_gettime(CLOCK_MONOTONIC, &t1);
    fprintf(stderr, "took %.3f ms\n",
            (t1.tv_sec - t0.tv_sec) * 1e3 + (t1.tv_nsec - t0.tv_nsec) / 1e6);
```

Build and run once with and once without `-DDEBUG`, each time with `2>/dev/null` after the first line so you only see the timing. How much slower is the instrumented loop? Now imagine the loop is an interrupt handler on a microcontroller.

## Questions

- Why does the macro write to `stderr` instead of `stdout`? Name two concrete advantages.
- What does `((void)0)` do, and what would break if the disabled macro expanded to nothing at all? Try `if (x) DBG("a", 1); else ...`.
- Why is `__func__` not a macro but a predefined identifier? Where does its value come from?
- In Task 5, why is the string gone from the release binary? At which stage of the toolchain was it removed?
- Task 7: by what factor did the instrumented loop slow down? What does that mean for debugging a program whose bug depends on timing?
- Name a situation where `DBG` is the *right* tool even though a debugger is available.

## Advice

`DBG` is the tool for "I know which variable and which line I care about". It needs no learning, works on every system that has `stderr`, and costs nothing in the release build. Use it freely, and leave the macro in the code; it is not clutter, it is instrumentation.

It is the wrong tool for "I do not know where the bug is", because every new question is a rebuild, and for anything where time matters, because a `printf` takes longer than most of the code around it. The next exercises are about those cases.
