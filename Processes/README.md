# Processes

-----

### 1\. The Core Concepts: Processes and Programs

At its heart, a **program** is a passive file on disk that contains a set of instructions. A **process**, on the other hand, is an active instance of a running program. A single program can be used to create multiple processes. The program file contains all the necessary information for the kernel to create a process, including:

  * **Binary format identification**: The kernel uses this to understand the executable file's layout, most commonly the **Executable and Linking Format (ELF)**.
  * **Machine-language instructions**: The actual code that tells the CPU what to do.
  * **Data**: Values used for variables and constants.
  * **Symbol and relocation tables**: Information for dynamic linking and debugging.

-----

### 2\. Process Identification and Hierarchy

Every process in a Linux system has a unique identifier called a **process ID (PID)**. This PID is a positive integer used by the kernel and other processes to refer to it.

  * `getpid()`: This system call returns the PID of the calling process.

Processes also have a **parent process ID (PPID)**, which is the PID of the process that created it. This forms a process tree, with the **`init` process (PID 1)** as the root of the tree.

  * `getppid()`: This system call returns the PPID of the calling process.

If a parent process terminates before its child, the `init` process "adopts" the orphaned child.

-----

### 3\. The Memory Layout of a Process

A process's memory is organized into several distinct segments. This structure exists in **virtual memory**, an abstract representation managed by the kernel.

1.  **Text Segment**: Contains the executable machine-language instructions. It's read-only to prevent accidental modification and is often shared among multiple processes running the same program to save memory.
2.  **Initialized Data Segment**: Stores global and static variables that have been explicitly initialized in the source code. The values are loaded from the program file.
3.  **Uninitialized Data Segment (BSS)**: Contains global and static variables that are not explicitly initialized. The kernel initializes all this memory to zero before the program starts. This separation saves disk space in the executable file, which only needs to store the size of the BSS segment.
4.  **Stack**: Used for automatic variables, function parameters, and the return address. It grows and shrinks dynamically with function calls. On most systems, the stack grows downwards in memory.
5.  **Heap**: This segment is used for dynamic memory allocation at runtime (e.g., using `malloc()`). It typically grows upwards toward the stack.

The virtual memory system maps these virtual addresses to physical memory (RAM). This provides **process isolation**, allowing multiple processes to run without interfering with each other's memory.

-----

### 4\. Command-Line Arguments and Environment Variables

The `main()` function is the entry point for a program and receives command-line arguments and environment variables from the kernel.

#### Command-Line Arguments (`argc`, `argv`)

The `main()` function signature is typically `int main(int argc, char *argv[])`.

  * `argc` (argument count): The number of command-line arguments.
  * `argv` (argument vector): An array of pointers to null-terminated strings, each representing an argument. `argv[0]` is conventionally the program's name.

**C Code Example: Accessing Command-Line Arguments**

```c
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char *argv[]) {
    printf("Number of arguments: %d\n", argc);

    for (int i = 0; i < argc; i++) {
        printf("argv[%d]: %s\n", i, argv[i]);
    }

    return 0;
}
```

#### Environment List

Every process has an **environment list**, which is an array of strings in the `name=value` format. A child process inherits a copy of its parent's environment. This is a crucial way to pass information to a program without using command-line arguments.

**C Code Example: Accessing Environment Variables**

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    char *path = getenv("PATH");
    if (path) {
        printf("The PATH environment variable is: %s\n", path);
    } else {
        printf("The PATH environment variable is not set.\n");
    }

    return 0;
}
```

-----


### Exercise 6-1: The BSS Segment and Executable Size

**Question:** Compile the program in Listing 6-1 (which contains an uninitialized array of 10 MB), and list its size using `ls -l`. Although the program contains a large array, the executable file is much smaller than this. Why is this?

**Solution:**

The reason the executable file is significantly smaller than the size of the uninitialized array is due to the **BSS (Block Started by Symbol) segment** of the program's memory.

  * **Uninitialized Data**: The C standard guarantees that global and static variables that are not explicitly initialized will be automatically set to zero. Instead of storing a full 10 MB of null bytes in the executable file, the compiler places these variables in the BSS segment.
  * **Space-Saving Mechanism**: The executable file only needs to store a record of the BSS segment's **start address** and **size**. The operating system's program loader then allocates the required memory and initializes it to zero at runtime. This clever technique saves a massive amount of disk space.

### Exercise 6-2: `longjmp()` into a Returned Function

**Question:** Write a program to see what happens if we try to `longjmp()` into a function that has already returned.

**Solution and Explanation:**

The behavior of a `longjmp()` into a function that has already returned is **undefined**. The stack frame for that function is no longer valid, as it has been unwound and its memory may have been reused for other purposes. Attempting to jump back to it can lead to a program crash (e.g., a segmentation fault) or other unpredictable behavior.

This demonstrates why `setjmp()` and `longjmp()` should only be used to jump to a stack frame that is still active and on the call stack.

**C Code Example:**

```c
#include <stdio.h>
#include <setjmp.h>
#include <stdlib.h>

static jmp_buf env;

static void second_func() {
    printf("In second_func, returning...\n");
}

static void first_func() {
    // Save the environment
    if (setjmp(env) == 0) {
        printf("In first_func, calling second_func()...\n");
        second_func();
        printf("second_func() has returned, now exiting first_func().\n");
    } else {
        printf("Jumped back to first_func()! This is bad, the stack has been corrupted.\n");
        // We shouldn't get here.
    }
}

int main(void) {
    first_func();
    printf("Exiting main(). The environment is now invalid.\n");
    
    // Now we try to jump back to an invalid stack frame. This is undefined behavior.
    printf("Attempting longjmp() to an invalid stack frame...\n");
    longjmp(env, 1);
    
    return 0;
}
```

### Exercise 6-3: Implementing `setenv()` and `unsetenv()`

**Question:** Implement `setenv()` and `unsetenv()` using `getenv()`, `putenv()`, and, where necessary, code that directly modifies `environ`. Your version of `unsetenv()` should remove all definitions of an environment variable.

**Solution and Explanation:**

This exercise requires careful manipulation of the `environ` global variable, which is a pointer to the environment list. `environ` is an array of strings terminated by a `NULL` pointer.

**C Code:**

```c
#include <stdlib.h>
#include <string.h>
#include <stdio.h>

extern char **environ;

// Implementation of setenv()
int my_setenv(const char *name, const char *value) {
    char *new_string;
    size_t name_len = strlen(name);
    size_t value_len = strlen(value);

    // Check if the variable exists and can be overwritten
    char *existing = getenv(name);
    if (existing) {
        // Overwrite existing variable with putenv()
        if (asprintf(&new_string, "%s=%s", name, value) == -1) return -1;
        return putenv(new_string);
    } else {
        // Variable does not exist, so putenv() adds it
        if (asprintf(&new_string, "%s=%s", name, value) == -1) return -1;
        return putenv(new_string);
    }
}

// Implementation of unsetenv()
int my_unsetenv(const char *name) {
    char **ep;
    int found = 0;

    // Check if the name is empty
    if (name == NULL || name[0] == '\0') {
        errno = EINVAL;
        return -1;
    }

    // Iterate through the environment list
    for (ep = environ; *ep != NULL;) {
        if (strncmp(*ep, name, strlen(name)) == 0 && (*ep)[strlen(name)] == '=') {
            // Found a match, remove it by shifting subsequent pointers
            found = 1;
            char **src = ep + 1;
            char **dest = ep;
            while ((*dest++ = *src++) != NULL) {} // Shift pointers down
        } else {
            // Not a match, move to the next pointer
            ep++;
        }
    }

    if (found) return 0; // Success
    return 0; // Did not find variable, but still success
}
```

**Note:** The `asprintf()` function is a GNU extension and may require linking with `libbsd` or similar libraries on other systems. It is used here for convenience in creating the `name=value` string. This solution for `unsetenv()` directly manipulates the `environ` array, which is why it can handle and remove multiple definitions of the same environment variable.
