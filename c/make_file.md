# Makefile Course — Compiling C Projects with Multiple Files and Folders

## Table of Contents

1. [Why a Makefile?](#1-why-a-makefile)
2. [Basic Concepts](#2-basic-concepts)
3. [Your First Makefile](#3-your-first-makefile)
4. [Project with Folders](#4-project-with-folders)
5. [Variables](#5-variables)
6. [Automatic Variables](#6-automatic-variables)
7. [Pattern Rules](#7-pattern-rules)
8. [A Professional Makefile](#8-a-professional-makefile)
9. [Useful Commands](#9-useful-commands)
10. [Common Errors](#10-common-errors)

---

## 1. Why a Makefile?

When your C project grows, you don't want to type this every time:

```bash
gcc main.c src/utils.c src/math.c src/input.c -o my_program
```

A **Makefile** lets you just type `make` and it handles everything automatically.

It also **only recompiles what changed** — if you edit one file, it doesn't recompile all the others. This saves a lot of time on big projects.

---

## 2. Basic Concepts

A Makefile is made of **rules**. Each rule has this structure:

```makefile
target: dependencies
	command
```

> ⚠️ **Critical:** The indentation before `command` MUST be a **TAB**, not spaces. This is one of the most common Makefile errors.

- **target** — the file you want to produce (or a task name like `clean`)
- **dependencies** — files the target depends on
- **command** — the shell command to run

**Example:**

```makefile
main: main.c
	gcc main.c -o main
```

This means: *"To build `main`, use `main.c`, and run `gcc main.c -o main`."*

---

## 3. Your First Makefile

**Project structure:**

```
project/
├── main.c
└── Makefile
```

**Makefile:**

```makefile
all: main

main: main.c
	gcc main.c -o main

clean:
	rm -f main
```

**Commands:**

```bash
make          # builds the project
make clean    # removes the compiled binary
```

`all` is the default target — it's what runs when you just type `make`.

---

## 4. Project with Folders

**Project structure:**

```
project/
├── main.c
├── Makefile
└── src/
    ├── utils.c
    ├── utils.h
    ├── math.c
    └── math.h
```

**main.c** includes the headers like this:

```c
#include "src/utils.h"
#include "src/math.h"
```

**Simple Makefile for this project:**

```makefile
all: main

main: main.c src/utils.c src/math.c
	gcc main.c src/utils.c src/math.c -o main

clean:
	rm -f main
```

This works, but it recompiles everything every time. Let's improve this.

---

## 5. Variables

Variables avoid repetition and make the Makefile easier to update.

```makefile
CC      = gcc
CFLAGS  = -Wall -Wextra
SRC     = main.c src/utils.c src/math.c
OUT     = main

all: $(OUT)

$(OUT): $(SRC)
	$(CC) $(CFLAGS) $(SRC) -o $(OUT)

clean:
	rm -f $(OUT)
```

**Common variables by convention:**

| Variable | Purpose | Example |
|---|---|---|
| `CC` | The compiler | `gcc` or `clang` |
| `CFLAGS` | Compiler flags | `-Wall -Wextra -g` |
| `SRC` | Source files | `main.c src/*.c` |
| `OUT` | Output binary name | `my_program` |
| `OBJ` | Object files | `main.o src/utils.o` |

---

## 6. Automatic Variables

These special variables are filled automatically by Make inside a rule:

| Variable | Meaning |
|---|---|
| `$@` | The target name |
| `$<` | The first dependency |
| `$^` | All dependencies |

**Example:**

```makefile
main: main.c src/utils.c
	gcc $^ -o $@
```

This is equivalent to:

```makefile
main: main.c src/utils.c
	gcc main.c src/utils.c -o main
```

---

## 7. Pattern Rules

Instead of compiling everything in one command, you can compile each `.c` file into a `.o` (object file) separately, then link them at the end. This is faster because Make only recompiles files that changed.

```makefile
CC      = gcc
CFLAGS  = -Wall -Wextra
SRC     = $(wildcard *.c src/*.c)
OBJ     = $(SRC:.c=.o)
OUT     = main

all: $(OUT)

# Link all object files into the final binary
$(OUT): $(OBJ)
	$(CC) $(OBJ) -o $@

# Compile each .c file into a .o file
%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	rm -f $(OBJ) $(OUT)
```

**What `$(wildcard *.c src/*.c)` does:** automatically finds all `.c` files in the root and `src/` folder. You don't need to list them manually.

**What `$(SRC:.c=.o)` does:** replaces `.c` with `.o` in the list, giving you the object file names.

---

## 8. A Professional Makefile

This is a complete, reusable Makefile that works for most C projects:

```makefile
# ── Compiler & Flags ──────────────────────────────────────────
CC      = gcc
CFLAGS  = -Wall -Wextra -g

# ── Files ─────────────────────────────────────────────────────
SRC     = $(wildcard *.c src/*.c)
OBJ     = $(SRC:.c=.o)
OUT     = my_program

# ── Rules ─────────────────────────────────────────────────────
all: $(OUT)

$(OUT): $(OBJ)
	$(CC) $(OBJ) -o $@
	@echo "✓ Build successful: $(OUT)"

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	rm -f $(OBJ) $(OUT)
	@echo "✓ Cleaned."

re: clean all

.PHONY: all clean re
```

**The `.PHONY` line** tells Make that `all`, `clean`, and `re` are not real files — they are task names. Without this, if a file named `clean` existed in the folder, `make clean` would do nothing.

**The `re` rule** cleans and rebuilds from scratch in one command:

```bash
make re
```

---

## 9. Useful Commands

```bash
make              # build the project (runs the 'all' rule)
make clean        # remove compiled files
make re           # clean + rebuild
make -n           # dry run: shows what would be executed without doing it
make --debug      # shows detailed Make decision process
make -j4          # compile with 4 parallel jobs (faster on big projects)
```

---

## 10. Common Errors

### ❌ "missing separator"

```
Makefile:5: *** missing separator. Stop.
```

**Cause:** You used spaces instead of a TAB before the command.
**Fix:** Replace the indentation with a real TAB key.

---

### ❌ "No rule to make target"

```
make: *** No rule to make target 'src/utils.c'. Stop.
```

**Cause:** The file doesn't exist or the path is wrong.
**Fix:** Check the file path and name in your `SRC` variable.

---

### ❌ Nothing recompiles after a change

**Cause:** Make checks timestamps. If the `.o` file is newer than the `.c` file, it skips it.
**Fix:** Run `make re` to force a full rebuild.

---

### ❌ Header changes are not detected

By default, Make doesn't know that a `.c` file depends on a `.h` file. If you change `utils.h`, the files that include it won't recompile automatically.

**Fix:** Add `-MMD -MP` to `CFLAGS` and include the generated dependency files:

```makefile
CFLAGS  = -Wall -Wextra -g -MMD -MP
-include $(OBJ:.o=.d)
```

This makes `gcc` generate `.d` files that track header dependencies automatically.

---

## Summary

| Step | What to do |
|---|---|
| 1 | Create a `Makefile` at the root of your project |
| 2 | Set `CC`, `CFLAGS`, `SRC`, `OUT` variables |
| 3 | Use `$(wildcard src/*.c)` to auto-detect source files |
| 4 | Use pattern rules `%.o: %.c` for incremental compilation |
| 5 | Add `clean`, `re`, and `.PHONY` |
| 6 | Run `make` to build, `make re` to rebuild from scratch |
