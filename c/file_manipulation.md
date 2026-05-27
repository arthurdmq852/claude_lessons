# File manipulation in C

---

## Chapter 01 — Introduction

### Why files?

In C, everything you store in variables disappears when the program ends. Files let you persist data on disk, read configuration, log events, or interact with system files like `/etc/hosts`.

All file operations in C go through the standard library `<stdio.h>`, using a special type called `FILE`.

### The FILE type

`FILE` is a struct defined in `<stdio.h>`. You never manipulate it directly — you always work with a **pointer** to it.

```c
#include <stdio.h>

FILE *fp; // a pointer to a FILE
```

This pointer is what every file function takes as its first argument. Think of it as a "handle" to the open file.

### The 3-step pattern

Every file operation in C follows the same pattern:

```c
// 1. Open
FILE *fp = fopen("file.txt", "r");

// 2. Read or write
// ... do things ...

// 3. Close
fclose(fp);
```

> ⚠️ Always close your files. Forgetting `fclose()` can cause data loss or resource leaks.

---

## Chapter 02 — Opening a file with fopen

### Syntax

```c
FILE *fopen(const char *filename, const char *mode);
```

`filename` is the path to the file. `mode` is a string that controls how the file is opened.

### Modes

| Mode  | Meaning                     | File must exist? |
|-------|-----------------------------|------------------|
| `"r"` | Read only                   | Yes              |
| `"w"` | Write (creates or erases)   | No               |
| `"a"` | Append (add to end)         | No               |
| `"r+"` | Read + write               | Yes              |
| `"w+"` | Read + write (erases)      | No               |
| `"a+"` | Read + append              | No               |

### Always check for NULL

`fopen` returns `NULL` if the file can't be opened (wrong path, no permission, etc.). Always check it.

```c
FILE *fp = fopen("/etc/hosts", "a");
if (fp == NULL) {
    perror("fopen failed");
    return 1;
}
```

`perror()` prints the system error message automatically (e.g. "Permission denied").

> ⚠️ To write to `/etc/hosts`, you need to run your program with `sudo`. Otherwise `fopen` returns `NULL`.

---

## Chapter 03 — Reading from a file

### fgetc — read one character

```c
int c;
while ((c = fgetc(fp)) != EOF) {
    putchar(c);
}
```

`fgetc` returns the next character, or `EOF` when the file ends. Note: the return type is `int`, not `char`, because `EOF` is `-1`.

### fgets — read one line

```c
char line[256];
while (fgets(line, sizeof(line), fp) != NULL) {
    printf("%s", line);
}
```

`fgets(buffer, size, fp)` reads one line at a time into `buffer`, up to `size - 1` chars. It keeps the `\n` at the end.

### fscanf — read formatted data

```c
int age;
char name[50];
fscanf(fp, "%s %d", name, &age);
```

Works like `scanf` but reads from a file instead of stdin. Useful for structured data.

> 💡 For reading lines of unknown length, prefer `fgets` over `fscanf` — it's safer.

---

## Chapter 04 — Writing to a file

### fputc — write one character

```c
fputc('A', fp);
```

### fputs — write a string

```c
fputs("127.0.0.1 example.com\n", fp);
```

Note: `fputs` does **not** add a newline automatically — you must include `\n` yourself.

### fprintf — write formatted output

```c
char *url = "facebook.com";
fprintf(fp, "127.0.0.1 %s\n", url);
```

Works exactly like `printf` but writes to a file. This is the most flexible option — great for your URL blocker project.

### "w" vs "a"

| Mode  | Effect on existing content          |
|-------|--------------------------------------|
| `"w"` | Erases everything and starts fresh   |
| `"a"` | Keeps existing content, adds at end  |

> ⚠️ For `/etc/hosts`, always use `"a"` — never `"w"`, or you'll erase all your existing DNS entries!

---

## Chapter 05 — Moving the cursor with fseek

### The file cursor

When you open a file, there's an invisible cursor that tracks your current position. Every read or write advances it automatically.

### fseek — move the cursor

```c
int fseek(FILE *fp, long offset, int origin);
```

| Origin     | Meaning                        |
|------------|--------------------------------|
| `SEEK_SET` | From the beginning of the file |
| `SEEK_CUR` | From the current position      |
| `SEEK_END` | From the end of the file       |

```c
fseek(fp, 0, SEEK_SET);  // go to the beginning
fseek(fp, 0, SEEK_END);  // go to the end
fseek(fp, 10, SEEK_SET); // go to byte 10
```

### ftell — get current position

```c
long pos = ftell(fp); // returns current byte offset
```

### rewind — go back to start

```c
rewind(fp); // equivalent to fseek(fp, 0, SEEK_SET)
```

---

## Chapter 06 — Binary files

### Text vs binary mode

By default, `fopen` opens files in text mode. For raw binary data (images, structs, etc.), add `b` to the mode:

```c
FILE *fp = fopen("data.bin", "rb"); // read binary
FILE *fp = fopen("data.bin", "wb"); // write binary
```

### fread and fwrite

```c
// write a struct to file
typedef struct { int x; float y; } Point;
Point p = {10, 3.14f};
fwrite(&p, sizeof(Point), 1, fp);

// read it back
Point p2;
fread(&p2, sizeof(Point), 1, fp);
```

The signature is: `fread/fwrite(pointer, size_of_one_element, count, file)`. Both return the number of elements actually read/written.

---

## Chapter 07 — Error handling

### feof — did we reach the end?

```c
if (feof(fp)) {
    printf("End of file reached\n");
}
```

> ⚠️ Don't use `feof()` as the loop condition — it only becomes true *after* a failed read, which leads to processing the last value twice. Use the return value of `fgets`/`fgetc` instead.

### ferror — did something go wrong?

```c
if (ferror(fp)) {
    perror("File error");
    fclose(fp);
    return 1;
}
```

### clearerr — reset error flags

```c
clearerr(fp); // clears both EOF and error flags
```

### Golden rules

1. Always check if `fopen` returned `NULL`.
2. Always call `fclose()` — even on error paths.
3. Check the return value of `fread`/`fwrite` for partial reads/writes.

---

## Chapter 08 — Full example: the URL blocker

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define MAX_URL 256

int main() {
    char *url = malloc(MAX_URL * sizeof(char));
    if (url == NULL) {
        fprintf(stderr, "malloc failed\n");
        return 1;
    }

    printf("URL to block: ");
    fgets(url, MAX_URL, stdin);

    // remove the trailing newline from fgets
    url[strcspn(url, "\n")] = '\0';

    FILE *fp = fopen("/etc/hosts", "a");
    if (fp == NULL) {
        perror("fopen");
        free(url);
        return 1;
    }

    fprintf(fp, "127.0.0.1 %s\n", url);
    fprintf(fp, "127.0.0.1 www.%s\n", url);

    fclose(fp);
    free(url);
    printf("%s blocked!\n", url);
    return 0;
}
```

Compile and run with:

```bash
gcc blocker.c -o blocker
sudo ./blocker
```

### What's used from this course

| Concept          | Where                                      |
|------------------|--------------------------------------------|
| `malloc`         | Dynamically allocate the URL buffer        |
| `fgets`          | Read the URL from stdin safely             |
| `fopen("a")`     | Append to /etc/hosts without erasing it    |
| `fprintf`        | Write the formatted blocking lines         |
| `NULL check`     | Handle permission errors gracefully        |
| `fclose + free`  | Clean up resources                         |
