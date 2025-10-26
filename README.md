### **Detailed Notes on Linux Programming Concepts**

>[!NOTE]  
> These are the extract of the class notes that was taught to me in **Vector India**.

#### **1. Static vs. Dynamic Linking**

**Key Concept:** Linking is the process of combining your program's code with library code to create an executable.

**Static Linking:**
*   **How it works:** The library code is copied **directly into the final executable file** at compile time.
*   **Command:** `cc -static main.c -o output_static`
*   **Characteristics:**
    *   **File Size:** Large. The `size` command shows large `text`, `data`, and `bss` sections because the library is included.
    *   **Dependencies:** None. The executable is self-contained.
    *   **Tools:** `nm output_static` shows all symbols, including `printf` and other library functions, within the executable.
*   **Pros:** Portable, no external dependencies at runtime.
*   **Cons:** Wastes disk and memory if multiple programs use the same library; bug fixes require recompiling every program.

**Dynamic Linking:**
*   **How it works:** The executable contains **references** to a shared library (`.so` file). The library is loaded into memory only when the program runs.
*   **Command:** `cc main.c -o output_dynamic`
*   **Characteristics:**
    *   **File Size:** Small. The `size` command shows a much smaller executable.
    *   **Dependencies:** Required. Use `ldd output_dynamic` to list them (e.g., `libc.so.6`).
    *   **Tools:** `nm output_dynamic` may show `U` (undefined) for library functions like `printf`.
*   **Pros:**
    *   **Saves Memory:** The library is loaded into RAM once and shared by all programs using it.
    *   **Easy Bug Fixes/Updates:** Update the shared library file, and all programs using it will benefit without needing to be recompiled.
*   **Cons:** Requires the correct library version to be present on the system at runtime.

---

#### **2. Creating and Using Your Own Libraries**

**A. Static Library (`.a` archive)**

1.  **Compile object files:** Create Position Independent Code (PIC) object files.
    ```bash
    cc -c -fPIC sum.c -o sum.o
    cc -c -fPIC mul.c -o mul.o
    ```
2.  **Create the library:** Use the `ar` (archive) command.
    ```bash
    ar rcs libcalc.a sum.o mul.o
    ```
3.  **Use the library:** Link it during compilation.
    ```bash
    cc main.c libcalc.a -o my_program
    # OR
    cc main.c -L. -lcalc -o my_program
    ```
4.  **Manage the library:**
    *   **View contents:** `ar -tv libcalc.a`
    *   **Delete an object:** `ar -d libcalc.a sum.o`
    *   **Add an object:** `ar -r libcalc.a new.o`

**B. Dynamic Library (Shared Object `.so`)**

1.  **Compile object files (with `-fPIC`):** This is crucial for shared libraries.
    ```bash
    cc -c -fPIC sum.c -o sum.o
    cc -c -fPIC mul.c -o mul.o
    ```
2.  **Create the shared library:** Use the `-shared` flag.
    ```bash
    cc -shared -o libcalc.so sum.o mul.o
    ```
3.  **Use the library:** Compile and tell the linker to use your shared library. You might also need to update the linker's path.
    ```bash
    cc main.c -L. -lcalc -o my_program
    export LD_LIBRARY_PATH=.:$LD_LIBRARY_PATH  # Tell the loader where to find the lib at runtime
    ./my_program
    ```

---

#### **3. Advanced Dynamic Linking: Runtime Linking (``dlopen`` API)**

**Key Concept:** Instead of linking at compile or load time, you can load a library, find functions, and use them *explicitly* during your program's execution.

**Use Case:** Plugin architectures, loading features on demand.

**The API (`#include <dlfcn.h>`, link with `-ldl`):**

1.  **`dlopen(const char *filename, int flag)`:** Opens a shared library.
    *   `filename`: Path to the `.so` file.
    *   `flag`: `RTLD_LAZY` (resolve symbols only as needed) or `RTLD_NOW` (resolve all symbols immediately).
2.  **`dlsym(void *handle, const char *symbol)`:** Returns the address of a symbol (e.g., a function) from the opened library.
3.  **`dlclose(void *handle)`:** Closes the library handle.
4.  **`dlerror(void)`:** Returns a human-readable error string.

**Example Workflow:**
```c
#include <dlfcn.h>
#include <stdio.h>

int main() {
    void *handle;
    int (*sum_func)(int, int); // Declare a function pointer
    char *error;

    // 1. Open the library
    handle = dlopen("./libcalc.so", RTLD_LAZY);
    if (!handle) {
        fputs(dlerror(), stderr);
        return 1;
    }

    // 2. Find the 'sum' function
    sum_func = (int (*)(int, int)) dlsym(handle, "sum");
    if ((error = dlerror()) != NULL) {
        fputs(error, stderr);
        return 1;
    }

    // 3. Use the function
    int result = (*sum_func)(10, 20);
    printf("Result: %d\n", result);

    // 4. Close the library
    dlclose(handle);
    return 0;
}
```
**Compile:** `cc main.c -ldl -o runtime_program`

**Advantage:** The library (`libcalc.so`) is only loaded into RAM when the `dlopen` call is made, saving resources if the feature is never used.

---

#### **4. Process Management Fundamentals**

**What is a Process?**
A program that is currently loaded into memory and being executed. It consists of code, data, and stack.

**Process States:**
*   **Running:** Executing on the CPU.
*   **Ready:** Ready to run, waiting for the CPU.
*   **Wait/Sleep:** Waiting for an event (e.g., I/O, timer).
*   **Suspended:** Paused by a signal (e.g., `SIGSTOP` / `kill -19`). Can be resumed (`SIGCONT` / `kill -18`).
*   **Zombie:** Finished execution but still has an entry in the process table.

**Process ID (PID):**
*   A unique number assigned to each process.
*   Retrieved in C using `getpid()` (own PID) and `getppid()` (parent's PID).
*   The first process is `init`/`systemd` with PID 1.
*   Maximum PID value is defined in `/proc/sys/kernel/pid_max`.

**Key Commands:**
*   `ps`: List processes in the current terminal.
*   `ps -e` or `ps -A`: List all processes on the system.
*   `&`: Run a process in the background (e.g., `./a.out &`). The shell assigns a **Job ID**.
*   `fg [%job_id]`: Bring a background job to the foreground.
*   `kill [signal] [PID]`: Send a signal to a process (e.g., `kill -9 1234` to force kill).

---

#### **5. Linux Boot Process**

1.  **BIOS (Basic Input/Output System):** Hardware check and initialization.
2.  **MBR (Master Boot Record):** A small 512-byte program at the disk's start that locates and runs the bootloader.
3.  **GRUB (GRand Unified Bootloader):** Presents a menu, loads the selected Linux kernel into memory.
4.  **Kernel:** Initializes the system, mounts a temporary root filesystem (`initramfs`).
5.  **Init (`/sbin/init`):** The first process (PID 1). It starts all other system services and daemons according to the **runlevel**.
6.  **Login Prompt:** The `getty` process provides a login prompt.

---

#### **6. Random Number Generation in C**

**Key Concept:** The standard C library provides a pseudo-random number generator.

**Functions:**
*   `int rand(void);` Returns a pseudo-random number between `0` and `RAND_MAX`.
*   `void srand(unsigned int seed);` Seeds the random number generator.

**Important:** Without seeding (`srand`), `rand()` will produce the **same sequence** every time the program runs.

**How to get a good, changing seed:**
*   Use the current time: `srand(time(0));`
*   Use the process ID: `srand(getpid());`

**Generating Numbers in a Specific Range:**
*   **`0 to N-1`:** `rand() % N`
*   **`Low to High`:** `(rand() % (High - Low + 1)) + Low`

**Examples:**
*   **-50 to +50:** `(rand() % 101) - 50`
*   **0.25 to 0.75:** `((rand() % 51) * 0.01) + 0.25` // 0 to 50 -> 0.00 to 0.50 -> 0.25 to 0.75

---

### **Summary of Important Concepts Extracted**

1.  **Linking Types:** Understand the trade-offs between static and dynamic linking.
2.  **Library Creation:** Know the steps to create both static (`.a`) and dynamic (`.so`) libraries.
3.  **`dlopen` API:** For advanced, on-demand runtime linking.
4.  **Process vs. Program:** A process is an active instance of a program.
5.  **PID & PPID:** How the OS identifies and manages process hierarchies.
6.  **Process States:** The lifecycle of a process (Ready, Running, Waiting, etc.).
7.  **Boot Process:** The sequence from power-on to a working Linux system (BIOS -> MBR -> GRUB -> Kernel -> Init).
8.  **Random Numbers:** Always seed with `srand()` for variability. Use modulo to constrain the range.

---
## Linux/Unix Programming

#### **1. Program vs. Process**

**Key Concept:** A **Program** is a passive file on disk, while a **Process** is an active instance of that program loaded into memory and executing.

**What is a Program File (e.g., `a.out`)?**
It is an **Executable and Linkable Format (ELF)** file that contains:
1.  **Binary Format Identifier:** Magic number identifying it as an ELF file (previously systems used COFF).
2.  **Machine Instructions:** The actual code (algorithm) of the program.
3.  **Entry Point Address:** The memory address of the first instruction to execute (`main`).
4.  **Data:** Initialized and uninitialized variables (`data` and `bss` sections).
5.  **Symbol Table:** Information for debugging and dynamic linking.
6.  **Shared Library Info:** A list of dynamic libraries (`libc.so.6`, etc.) required at runtime.

**What is a Process?**
An active entity created by the kernel to execute a program. It is represented by a **Process Control Block (PCB)** which contains:
*   **PID & PPID:** Process ID and Parent Process ID.
*   **Program Code:** The executable instructions.
*   **Data Sections:** Initialized data, uninitialized data (BSS), heap, and stack.
*   **Virtual Memory Table:** Maps process addresses to physical RAM.
*   **Signal Information:** Pending and handled signals.
*   **Resource Usage:** CPU time, memory usage, open files.
*   **CPU State:** Values of registers (Stack Pointer, Program Counter).

**Orphaned Process:** If a parent terminates before its child, the child is adopted by the `init` process (PID 1). Its `getppid()` will then return `1`.

---

#### **2. Non-Local Goto: `setjmp()` and `longjmp()`**

**Key Concept:** These functions allow you to perform a "goto" across different function scopes, bypassing the normal function call/return stack unwinding. This is useful for error handling deep in nested function calls.

**Functions:**
*   `#include <setjmp.h>`
*   **`int setjmp(jmp_buf env);`**
    *   Saves the current stack context (register states like SP, PC) into the `env` buffer.
    *   **Returns `0`** when called directly.
*   **`void longjmp(jmp_buf env, int val);`**
    *   Restores the stack context from `env`.
    *   The program resumes execution as if `setjmp()` had just returned.
    *   **It returns the value `val`.** If `val` is `0`, `setjmp()` returns `1` instead (to distinguish from the direct call).

**Example Workflow for Error Handling:**
```c
#include <stdio.h>
#include <setjmp.h>

jmp_buf env; // Global buffer to save the state

int div(int i, int j) {
    if (j == 0) {
        longjmp(env, 4); // Jump back to setjmp, returning 4
    }
    return (i / j);
}

main() {
    int a = 10, b = 0, result;

    // Set the jump point
    int error_code = setjmp(env);

    if (error_code == 0) {
        // First time setjmp returns 0: proceed with risky operation
        result = div(a, b);
        printf("Result: %d\n", result);
    } else {
        // If longjmp was called, setjmp returns the error code
        if (error_code == 4) {
            printf("Error: Division by zero!\n");
        }
    }
}
```
**Important:** The stack context saved by `setjmp()` becomes invalid if the function that called `setjmp()` returns.

---

#### **3. Context Switching**

**Key Concept:** In a multitasking OS, a single CPU core can run multiple processes concurrently by rapidly switching between them. This is managed by the **scheduler**.

**How it Works:**
1.  The CPU executes instructions from Process P1.
2.  After its **time slice** (quantum) expires or it waits for I/O, an interrupt occurs.
3.  The OS saves the current state of P1 (registers, PC, SP, etc.) into its **PCB**.
4.  The OS loads the saved state of the next process, P2, from its PCB into the CPU registers.
5.  The CPU now resumes executing P2 from where it left off.

This entire procedure of saving one process's state and loading another's is a **Context Switch**. It is essential for creating the illusion of simultaneous execution.

---

#### **4. The `system()` Function**

**Key Concept:** The `system()` library function allows a C program to execute a shell command as if it were typed in a terminal.

**Synopsis:** `#include <stdlib.h>`  
`int system(const char *command);`

**How it Works:**
*   It calls `/bin/sh -c "command"`.
*   The function **blocks** (waits) until the shell command has finished executing.
*   The parent process (your C program) waits because the shell (`bash`) is its child and it waits for the shell to terminate.

**Example:**
```c
#include <stdlib.h>
main() {
    system("ls -l"); // Executes "ls -l", program waits here for it to finish
    while(1); // This is executed only after "ls -l" completes
}
```
**Note:** The command's executable is loaded from the disk into RAM for execution and is removed from RAM afterward, but the original file on the disk remains untouched.

---

#### **5. Library Functions vs. System Calls**

| Feature | Library Function (API) | System Call (SCI) |
| :--- | :--- | :--- |
| **Definition** | A function provided by a compiler's library (e.g., `libc`). | A function provided by the operating system kernel. |
| **Other Names** | Application Programming Interface (API). | System Call Interface (SCI). |
| **Execution Speed** | Faster. Executes entirely in **user space**. | Slower. Requires a context switch to **kernel space**. |
| **Examples** | `printf()`, `malloc()`, `fopen()` | `read()`, `write()`, `fork()`, `getpid()` |
| **Relationship** | Library functions are often **wrappers** around system calls. `printf()` eventually calls the `write()` system call to perform the actual output. | The kernel's direct interface for user programs to request services. |

**Communication Flow:**
`Your Program` -> `Library Function (e.g., printf)` -> `System Call (e.g., write)` -> `Kernel` -> `Hardware`

---

#### **6. The `fork()` System Call**

**Key Concept:** `fork()` creates a new process by duplicating the calling process. The new process is called the **child**, and the original is the **parent**.

**Return Value:**
*   **On Success:**
    *   **In the Parent:** Returns the **PID of the child** (a positive number).
    *   **In the Child:** Returns `0`.
*   **On Failure:** Returns `-1` in the parent (no child is created).

**Key Behavior:** After `fork()`, both parent and child **execute concurrently** from the instruction *immediately following* the `fork()` call. They have separate copies of variables and memory.

**Basic Example:**
```c
#include <stdio.h>
#include <unistd.h>
main() {
    pid_t pid;
    printf("Hello (PID=%d)\n", getpid()); // Printed once

    pid = fork();

    if (pid == 0) {
        // Child process
        printf("I am the child (PID=%d, PPID=%d)\n", getpid(), getppid());
    } else {
        // Parent process
        printf("I am the parent (PID=%d). My child is %d.\n", getpid(), pid);
    }
    // Common code for both
    while(1);
}
```
**Output:**
```
Hello (PID=2102)
I am the parent (PID=2102). My child is 2103.
I am the child (PID=2103, PPID=2102).
```

---

#### **7. Advanced `fork()` Patterns**

**A. Multiple `fork()` Calls**
```c
#include <stdio.h>
main() {
    printf("Hello\n");
    fork(); // Creates 2 processes
    fork(); // Each of the 2 processes creates another, total 4
    fork(); // Each of the 4 creates another, total 8
    printf("Hai\n");
    while(1);
}
```
*   The number of "Hai" outputs is **2^n**, where `n` is the number of `fork()` calls after the initial `printf`.
*   Total processes created = 2^n. The total number of processes running is the original + the new ones = **2^n**.

**B. `fork()` in a Loop**
```c
#include <stdio.h>
main() {
    int i;
    for (i = 0; i < 3; i++) {
        if (fork() == 0) {
            // Child process executes this
            printf("Hello from child (i=%d)\n", i);
            // IMPORTANT: Child should break out of the loop to prevent creating its own children.
            break;
        } else {
            // Parent process continues the loop
        }
    }
    while(1);
}
```
This creates a chain or tree of processes, depending on the placement of the `fork()` and the use of `break` in the child.

**C. Using `fork()` with `system()`**
Allows two commands to run (seemingly) in parallel.
```c
#include <stdio.h>
main() {
    if (fork() == 0) {
        system("pwd"); // Child executes 'pwd'
    } else {
        system("ls");  // Parent executes 'ls'
    }
    while(1);
}
```

---

#### **8. Buffered I/O and `fork()`**

**Key Concept:** The `printf` function often uses a **buffer** in memory to store data before writing it to the screen. This can lead to unexpected output when combined with `fork()`.

**The Issue:** If data is in the `stdout` buffer at the time of the `fork()`, both the parent and child processes inherit a **separate copy** of this buffer. When each process flushes its buffer (e.g., at exit or on a newline), the buffered data is printed again.

**Example:**
```c
#include <stdio.h>
main() {
    printf("Hello---"); // Note: No newline '\n', so it's buffered
    fork();
    printf("Hai\n"); // Now the buffer is flushed for both processes
}
```
**Potential Output:**
```
Hello---Hai
Hello---Hai
```
The string "Hello---" was in the buffer, duplicated by `fork()`, and printed twice.

**Solution:**
*   Use `\n` in `printf` as it often flushes the buffer.
*   Use `fflush(stdout);` before forking to manually flush the buffer.
*   The buffer is also flushed on `scanf`, process termination, or when the buffer is full.


---

### **Linux Process Management (Zombies, Waiting, and Execution)**

#### **1. Orphan and Zombie Processes**

**Key Concept:** Understanding the lifecycle and parent-child relationship of processes is crucial. When a child process finishes, its exit status must be collected by its parent. Failure to do this correctly leads to Orphan or Zombie processes.

**A. Orphan Process**
*   **Definition:** A process whose **parent has terminated** before it has.
*   **Mechanism:** The kernel detects that a process has become an orphan. To prevent it from being unmanaged, the **`init` process (PID 1) adopts it**.
*   **Result:** The orphan's `getppid()` will return `1`.
*   **Example:**
    ```c
    #include <stdio.h>
    #include <unistd.h>
    main() {
        if (fork() == 0) {
            // Child
            printf("Child: PID=%d, PPID=%d\n", getpid(), getppid());
            sleep(10); // Parent terminates during this sleep
            printf("Child: PID=%d, PPID=%d (Now adopted by init)\n", getpid(), getppid());
        } else {
            // Parent
            printf("Parent: PID=%d\n", getpid());
            sleep(2); // Parent terminates quickly
        }
    }
    ```

**B. Zombie Process (Defunct Process)**
*   **Definition:** A process that has **finished execution** but still has an entry in the process table.
*   **Mechanism:** This happens when a child process terminates, but its **parent has not yet read its exit status** using `wait()`. The kernel keeps the entry until the status is collected to let the parent know how the child ended.
*   **Problem:** Zombies consume system resources (PID, process table entry). If many zombies accumulate, they can exhaust available PIDs.
*   **Example:**
    ```c
    #include <stdio.h>
    #include <unistd.h>
    main() {
        if (fork() == 0) {
            // Child exits quickly
            printf("Child exiting.\n");
            exit(0);
        } else {
            // Parent does not call wait(), and goes into an infinite loop
            printf("Parent running... (Child is now a zombie)\n");
            while(1);
        }
    }
    ```
*   **Solution:** The parent must call `wait()` or `waitpid()` to "reap" the child and remove the zombie.

---

#### **2. Process Termination and Exit Status**

**Key Concept:** Processes can terminate either *normally* (e.g., by calling `exit`) or *abnormally* (e.g., killed by a signal). The `exit()` function is used to end a process and return a status code to the parent.

**Normal Termination:**
*   `exit(0)` or `return 0` from `main()`: Indicates **SUCCESS**.
*   `exit(1)` or `exit(EXIT_FAILURE)`: Indicates **FAILURE**.

**The `exit()` vs. `_exit()` Functions:**
*   **`void exit(int status);` (Library Function)**
    *   Performs cleanup before termination: flushes I/O buffers, calls functions registered with `atexit()`.
    *   **Use this in most cases.**
*   **`void _exit(int status);` (System Call)**
    *   Terminates the process **immediately** without any cleanup.
    *   Use this only in the child after a `fork()` to avoid flushing duplicated buffers.

**The `atexit()` Function:**
*   Registers a function to be called automatically when `exit()` is called.
*   Functions are called in the **reverse order** of their registration.
*   **Example:**
    ```c
    #include <stdio.h>
    #include <stdlib.h>
    void cleanup1() { printf("Cleanup 1\n"); }
    void cleanup2() { printf("Cleanup 2\n"); }
    main() {
        atexit(cleanup1);
        atexit(cleanup2);
        printf("In main\n");
        exit(0); // This will call cleanup2, then cleanup1
    }
    ```
    **Output:**
    ```
    In main
    Cleanup 2
    Cleanup 1
    ```

---

#### **3. The `wait()` and `waitpid()` System Calls**

**Key Concept:** A parent process uses `wait()` or `waitpid()` to synchronize with its children and collect their exit status, thereby preventing zombies.

**A. The `wait()` System Call**
*   **`pid_t wait(int *status);`**
*   **Behavior:**
    *   **Blocks** the parent until **any** of its children terminates.
    *   Returns the **PID of the terminated child**.
    *   Stores the child's exit status in the `status` variable.
*   **Disadvantage:** It blocks the parent, reducing concurrency.
*   **Example:**
    ```c
    #include <stdio.h>
    #include <sys/wait.h>
    #include <unistd.h>
    main() {
        int status;
        pid_t pid = fork();
        if (pid == 0) {
            // Child
            printf("Child running...\n");
            sleep(2);
            exit(42);
        } else {
            // Parent
            printf("Parent waiting...\n");
            wait(&status); // Blocks here until child exits
            printf("Child exited with status: %d\n", WEXITSTATUS(status));
        }
    }
    ```

**B. The `waitpid()` System Call**
*   **`pid_t waitpid(pid_t pid, int *status, int options);`**
*   **More powerful and flexible than `wait()`.**
*   **Parameters:**
    *   `pid`: Specify which child to wait for (e.g., `-1` for any child).
    *   `status`: Pointer to store the status.
    *   `options`: Control the behavior (e.g., `WNOHANG`).

**Important `waitpid()` Options:**
*   **`WNOHANG`**: Return immediately if no child has exited yet. This allows for **non-blocking** polling.
    ```c
    pid = waitpid(-1, &status, WNOHANG);
    if (pid == 0) {
        printf("No child has exited yet. Parent can do other work.\n");
    }
    ```
*   **`WUNTRACED`**: Also return if a child has stopped (e.g., by `SIGSTOP`).
*   **`WCONTINUED`**: Return if a stopped child has been resumed (by `SIGCONT`).

---

#### **4. Analyzing the Exit Status**

**Key Concept:** The `status` integer from `wait()` is a bitmask. Macros are used to decode it safely.

**Common Macros:**
*   **`WIFEXITED(status)`**: Returns true if the child terminated **normally** (via `exit` or `return`).
*   **`WEXITSTATUS(status)`**: If `WIFEXITED` is true, this returns the **exit status** (the low-order 8 bits passed to `exit`).
*   **`WIFSIGNALED(status)`**: Returns true if the child was terminated by a **signal**.
*   **`WTERMSIG(status)`**: If `WIFSIGNALED` is true, this returns the **signal number** that killed the child.
*   **`WIFSTOPPED(status)`**: Returns true if the child is currently **stopped**.
*   **`WSTOPSIG(status)`**: Returns the signal that caused the child to stop.

**Example Usage:**
```c
#include <sys/wait.h>
// ... after fork() and wait(&status) in parent ...
if (WIFEXITED(status)) {
    printf("Child exited normally with status %d\n", WEXITSTATUS(status));
} else if (WIFSIGNALED(status)) {
    printf("Child was killed by signal %d\n", WTERMSIG(status));
}
```

---

#### **5. The `exec()` Family of Functions**

**Key Concept:** The `exec()` functions replace the current process image with a new program. The PID remains the same, but the code, data, heap, and stack are overwritten.

**Key Behavior:**
*   On success, `exec()` **does not return** because the original program is gone.
*   On failure, it returns `-1`.
*   It's common to use `fork()` followed by `exec()` in the child process to run a new program.

**Common `exec()` Functions:**
| Function | Path Search? | Argument List |
| :--- | :--- | :--- |
| `execl("/bin/ls", "ls", "-l", NULL)` | **No** | **List** of arguments |
| `execvp("ls", args_array)` | **Yes** | **Vector** (array) of arguments |
| `execv("/bin/ls", args_array)` | **No** | **Vector** (array) of arguments |
| `execlp("ls", "ls", "-l", NULL)` | **Yes** | **List** of arguments |

**Key Differences:**
*   **`l` vs. `v`**: `l` expects a comma-separated list of arguments (ending with `NULL`). `v` expects an array of string pointers (ending with `NULL`).
*   **`p` vs. no `p`**: Functions with `p` (e.g., `execlp`, `execvp`) **search the `PATH` environment variable** for the executable. Functions without `p` require the **full path** to the executable.

**Examples:**
```c
#include <unistd.h>
// Example 1: Using execl (requires full path, list of args)
execl("/bin/ls", "ls", "-l", "-a", NULL);

// Example 2: Using execvp (searches PATH, uses argument array)
char *args[] = {"ls", "-l", "-a", NULL};
execvp("ls", args);

// Example 3: Common fork + exec pattern
#include <stdio.h>
main() {
    if (fork() == 0) {
        // Child process
        printf("Hello from child. Now running ls...\n");
        execl("/bin/ls", "ls", NULL);
        // If exec fails:
        perror("exec failed");
        exit(1);
    } else {
        // Parent process
        wait(NULL); // Wait for the child to finish
        printf("Parent: ls is done.\n");
    }
}
```

---

#### **6. Environment Variables**

**Key Concept:** Environment variables are name-value pairs that define the operating environment for a process. They are inherited from the parent process.

*   **View all:** Use the `env` command in the terminal.
*   **View one:** Use `echo $VARIABLE_NAME` (e.g., `echo $PATH`).
*   **The `PATH` Variable:** A colon-separated list of directories where the shell looks for commands when you don't specify a full path. This is why `execlp` and `execvp` can find commands like `ls` without `/bin/ls`.

**Accessing in a C Program:**
*   The `main` function can have a third argument: `main(int argc, char *argv[], char *envp[])`.
*   `envp` is an array of strings, each of the form `"NAME=value"`, ending with `NULL`.
```c
#include <stdio.h>
main(int argc, char *argv[], char *envp[]) {
    for (int i = 0; envp[i] != NULL; i++) {
        printf("%s\n", envp[i]);
    }
}
```

---

### **Summary of Key Concepts**

1.  **Zombie vs. Orphan:** A Zombie is a terminated child waiting for its parent to read its status. An Orphan is a live child whose parent died; it gets adopted by `init`.
2.  **Reaping Children:** Always use `wait()` or `waitpid()` in the parent to prevent zombie processes.
3.  **Exit Status:** Use `exit(status)` to terminate. Parents use `WIFEXITED` and `WEXITSTATUS` to check the child's exit status.
4.  **Non-Blocking Wait:** Use `waitpid(..., WNOHANG)` to check for child termination without blocking the parent.
5.  **The `exec()` Family:** These functions load and run a new program in the current process. They only return on error.
6.  **Classic Pattern:** `fork()` followed by `exec()` in the child is the standard way to create a new process running a different program.



---

### **Linux System Programming (Scheduling & Signals)**

#### **1. CPU Scheduling**

**Key Concept:** The CPU Scheduler is a part of the OS kernel that decides which process runs next on the CPU, aiming to maximize CPU utilization and system responsiveness.

**When Scheduling Happens:**
*   Process switches from **Running** to **Waiting** (e.g., for I/O).
*   Process switches from **Running** to **Ready** (e.g., on an interrupt).
*   Process switches from **Waiting** to **Ready** (e.g., I/O completion).
*   Process terminates.

**Scheduling Queues:**
*   **Job Queue:** All processes in the system.
*   **Ready Queue:** Processes that are loaded in memory and ready to execute, waiting for the CPU.
*   **Device Queues:** Processes waiting for a specific I/O device.

**Schedulers:**
*   **Long-Term Scheduler (Job Scheduler):** Selects processes from the job pool and loads them into memory (into the ready queue). It controls the **degree of multiprogramming**.
*   **Short-Term Scheduler (CPU Scheduler):** Selects a process from the ready queue and allocates the CPU to it. This scheduler is invoked frequently (every few milliseconds).
*   **Medium-Term Scheduler:** Used for swapping processes in and out of memory.

---

#### **2. Scheduling Algorithms**

**A. Preemptive vs. Non-Preemptive**
*   **Preemptive Scheduling:** The OS can force a running process to release the CPU (e.g., after its time slice expires or when a higher-priority process becomes ready).
*   **Non-Preemptive (Cooperative) Scheduling:** A process keeps the CPU until it voluntarily releases it (by terminating or switching to waiting state).

**B. Common Scheduling Policies**

1.  **First-Come, First-Served (FCFS) / FIFO**
    *   **Non-preemptive.**
    *   Simple to implement.
    *   **Convoy Effect:** Short processes can get stuck behind long ones, leading to poor **average turnaround time**.
    *   Good for long, CPU-bound processes; bad for short, I/O-bound processes.

2.  **Round Robin (RR)**
    *   **Preemptive.**
    *   Each process gets a small unit of CPU time (a **time quantum** or **time slice**).
    *   Excellent for **response time**.
    *   **Trade-off:**
        *   Time slice too long → Degenerates to FCFS.
        *   Time slice too short → High overhead from frequent **context switches**, hurting **throughput**.

3.  **Shortest Job First (SJF)**
    *   Can be preemptive (Shortest Remaining Time First - SRTF) or non-preemptive.
    *   Selects the process with the smallest next CPU burst.
    *   Provably optimal for **minimum average waiting time**.
    *   **Starvation** is possible for long processes.
    *   Impractical, as the next CPU burst length cannot be known precisely.

4.  **Priority Scheduling**
    *   Each process is assigned a priority. The highest priority process runs next.
    *   Can be preemptive or non-preemptive.
    *   **Starvation** is a major problem: low-priority processes may never run.
    *   **Solution: Aging** – gradually increase the priority of processes that wait in the system for a long time.

5.  **Multilevel Feedback Queue**
    *   Uses multiple ready queues, each with a different priority and scheduling algorithm (often Round Robin).
    *   A process can move between queues.
    *   **Example Rules:**
        *   Jobs start in the highest priority queue.
        *   If a job uses its entire time slice, it's moved to a lower priority queue (it's a CPU-bound job, be less nice to it).
        *   If a job gives up the CPU before its time slice is done (e.g., for I/O), it stays in the same or moves to a higher priority queue (it's an I/O-bound job, be nice to it).
    *   This is one of the most complex and common schedulers, designed to favor I/O-bound and interactive processes.

---

#### **3. Process Priority and the `nice` Value**

**Key Concept:** In Linux, you can influence the scheduler's decisions by adjusting a process's **nice value**.

*   **Range:** `-20` (highest priority, most favorable) to `+19` (lowest priority, least favorable).
*   A lower nice value means a higher priority.
*   Regular users can only *increase* the nice value (make a process less favorable). The `root` user can set any value.

**Commands:**
*   **`nice`:** Start a program with a modified priority.
    *   `nice -n 5 ./a.out &` (Runs `a.out` with nice value 5)
*   **`renice`:** Change the priority of an already running process.
    *   `renice 10 1234` (Changes PID 1234's nice value to 10)

**Process Types:**
*   **CPU-Bound Process:** Spends most of its time doing computations (e.g., `while(1);`).
*   **I/O-Bound Process:** Spends most of its time waiting for I/O operations (e.g., `scanf()`, file reads). The scheduler often gives these higher dynamic priority for better responsiveness.

---

#### **4. Introduction to Signals**

**Key Concept:** Signals are software interrupts delivered to a process to notify it of an important event.

*   They are used for inter-process communication (IPC) and asynchronous event handling.
*   Examples: `SIGINT` (Ctrl+C), `SIGKILL` (forceful termination), `SIGSEGV` (segmentation fault).

**Common Signals:**

| Signal Number | Signal Name      | Default Action        | Trigger                            |
| :------------ | :--------------- | :-------------------- | :--------------------------------- |
| 1             | `SIGHUP`         | Terminate             | Hangup (terminal closed)           |
| 2             | `SIGINT`         | Terminate             | Interrupt (Ctrl+C)                 |
| 3             | `SIGQUIT`        | Core Dump             | Quit (Ctrl+\)                      |
| 8             | `SIGFPE`         | Core Dump             | Floating-point exception           |
| 9             | `SIGKILL`        | Terminate             | Kill (cannot be caught or ignored) |
| 11            | `SIGSEGV`        | Core Dump             | Segmentation fault                 |
| 14            | `SIGALRM`        | Terminate             | Timer signal from `alarm()`        |
| 15            | `SIGTERM`        | Terminate             | Polite termination request         |
| 17            | `SIGCHLD`        | Ignore                | Child process stopped or terminated|
| 18            | `SIGCONT`        | Continue              | Continue if stopped                |
| 19            | `SIGSTOP`        | Stop                  | Stop (cannot be caught or ignored) |

**Sending Signals:**
*   **`kill` command:** `kill -SIGKILL 1234` or `kill -9 1234`
*   **`kill()` system call:** `kill(pid, sig);`
*   **`raise()` function:** Sends a signal to the current process itself.

---

#### **5. Handling Signals**

**Key Concept:** You can change what happens when a signal is received, except for `SIGKILL` and `SIGSTOP`.

**The `signal()` Function (Simple but Less Portable)**
*   `void (*signal(int signum, void (*handler)(int)))(int);`
*   **Handler can be:**
    *   `SIG_DFL`: Default action.
    *   `SIG_IGN`: Ignore the signal.
    *   A function pointer: Your custom signal handler.

**Example using `signal()`:**
```c
#include <stdio.h>
#include <signal.h>
#include <unistd.h>

void my_handler(int sig) {
    printf("\nReceived signal %d. Exiting gracefully.\n", sig);
    exit(0);
}

int main() {
    signal(SIGINT, my_handler); // Catch Ctrl+C
    printf("PID is %d. Press Ctrl+C to test.\n", getpid());
    while(1) {
        printf("Running...\n");
        sleep(1);
    }
    return 0;
}
```

**The `sigaction()` Function (Robust and Recommended)**
*   `int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact);`
*   More control and defined behavior by POSIX standard.
*   Allows you to specify a **blocking mask** during handler execution and other flags.

**The `struct sigaction`:**
*   `sa_handler`: Pointer to the signal handler function.
*   `sa_mask`: Set of signals to be blocked during the execution of the handler.
*   `sa_flags`: Special flags to modify behavior.

**Example using `sigaction()`:**
```c
#include <stdio.h>
#include <signal.h>
#include <unistd.h>

void handler(int sig) {
    write(STDOUT_FILENO, "Signal caught!\n", 15);
}

int main() {
    struct sigaction sa;
    sa.sa_handler = handler;
    sigemptyset(&sa.sa_mask); // Block no other signals in handler
    sa.sa_flags = 0;

    sigaction(SIGINT, &sa, NULL);

    printf("PID is %d. Press Ctrl+C.\n", getpid());
    while(1) pause(); // Wait for a signal
    return 0;
}
```

---

#### **6. Key Signal-Related Functions**

**A. `pause()`**
*   `int pause(void);`
*   Suspends the calling process until *any* signal is delivered that either terminates the process or is caught and the handler returns.

**B. `alarm()`**
*   `unsigned int alarm(unsigned int seconds);`
*   Schedules a `SIGALRM` to be delivered to the calling process after the specified number of seconds.
*   **Returns:** The number of seconds remaining until any previously scheduled alarm.
*   **Only one alarm can be pending.** A new call cancels the previous alarm.
*   **It does not block.** The process continues execution.

**Example: Disabling Ctrl+C for 10 seconds**
```c
#include <stdio.h>
#include <signal.h>
#include <unistd.h>

void alarm_handler(int sig) {
    signal(SIGINT, SIG_DFL); // Restore Ctrl+C default action
    printf("\nCtrl+C re-enabled.\n");
}

int main() {
    signal(SIGINT, SIG_IGN); // Ignore Ctrl+C
    signal(SIGALRM, alarm_handler);
    alarm(10); // Set alarm for 10 seconds
    printf("Ctrl+C disabled for 10 seconds. PID: %d\n", getpid());
    pause(); // Wait for the alarm
    while(1); // Now Ctrl+C will work here
    return 0;
}
```

---

#### **7. Daemon Processes**

**Key Concept:** A daemon is a background process that runs independently of a controlling terminal, often providing system services.

**Characteristics:**
*   Long-lived.
*   Runs in the background.
*   Has no controlling terminal (immune to terminal-generated signals like `SIGHUP` by default).
*   Started at system boot and runs until shutdown.
*   Examples: `crond` (scheduler), `sshd` (remote login), `httpd` (web server).

---

### **Summary of Key Concepts**

1.  **Scheduling Algorithms:** Understand the trade-offs between FCFS, Round Robin, SJF, and Priority Scheduling. Know what Preemptive and Non-Preemptive mean.
2.  **`nice` Value:** A user-influencable value that affects a process's scheduling priority.
3.  **Signals:** Software interrupts for process notification and control.
4.  **Signal Handling:** Use `sigaction()` over `signal()` for robust programs. You can set a handler to `SIG_DFL`, `SIG_IGN`, or a custom function.
5.  **Critical Functions:**
    *   `kill() / raise()`: Send signals.
    *   `alarm()`: Schedule a `SIGALRM`.
    *   `pause()`: Wait for any signal.
    *   `sigaction()`: Examine and change a signal action.
6.  **Daemons:** Long-running background processes with no controlling terminal.


---

### **Linux System Programming (Advanced Signals, Resource Limits, and File Management)**

#### **1. Advanced Signal Handling with `sigaction`**

**Key Concept:** The `sigaction` system call provides more robust and detailed control over signal handling compared to the `signal` function.

**Key `sa_flags` in `struct sigaction`:**

**A. `SA_NOCLDSTOP`**
*   **Purpose:** Modifies the behavior of `SIGCHLD`.
*   **Normal Behavior:** A parent receives `SIGCHLD` when a child:
    *   Terminates
    *   Is **stopped** (e.g., by `SIGSTOP`)
    *   Is **resumed** (e.g., by `SIGCONT`)
*   **With `SA_NOCLDSTOP`:** The parent **only** receives `SIGCHLD` when a child **terminates**. It does not receive it for stops or resumes.

**B. `SA_NOCLDWAIT`**
*   **Purpose:** Prevents child processes from becoming **zombies** when they terminate.
*   **Effect:** When a child exits, it is immediately reaped by the system. The parent cannot use `wait()` to get the child's exit status. The child vanishes completely.

**C. `SA_NODEFER` / `SA_NOMASK`**
*   **Purpose:** By default, the signal being handled is automatically **blocked** during the execution of its handler to prevent recursive interrupts.
*   **With `SA_NODEFER`:** The signal is **not blocked** during its own handler. This means the handler can be interrupted by another instance of the same signal.

**D. `SA_RESETHAND`**
*   **Purpose:** Resets the signal's disposition to its **default action** (`SIG_DFL`) **immediately** before the handler is entered.
*   **Effect:** The custom handler is only called for the **first** occurrence of the signal. Subsequent occurrences will trigger the default action (usually termination).

**The `sa_mask` Field:**
*   **Purpose:** Specifies a set of **additional signals** to be **blocked** during the execution of the signal handler.
*   This is in addition to the signal being delivered, which is blocked by default (unless `SA_NODEFER` is used).
*   **Functions for `sigset_t` (signal sets):**
    *   `sigemptyset(&set)`: Initializes a signal set to be empty.
    *   `sigfillset(&set)`: Initializes a signal set to contain all signals.
    *   `sigaddset(&set, signum)`: Adds a specific signal to the set.
    *   `sigdelset(&set, signum)`: Removes a specific signal from the set.

**Example: Blocking `SIGINT` during `SIGQUIT` handler**
```c
#include <signal.h>
#include <stdio.h>
#include <unistd.h>

void handler(int sig) {
    printf("In handler for signal %d\n", sig);
    sleep(5); // During this sleep, SIGINT is blocked.
    printf("Handler finished.\n");
}

int main() {
    struct sigaction sa;
    sa.sa_handler = handler;
    sigemptyset(&sa.sa_mask);
    sigaddset(&sa.sa_mask, SIGINT); // Block SIGINT while handler runs
    sa.sa_flags = 0;

    sigaction(SIGQUIT, &sa, NULL);

    while(1) pause();
    return 0;
}
```

---

#### **2. Resource Limits (`getrlimit`, `setrlimit`)**

**Key Concept:** The `getrlimit` and `setrlimit` system calls allow a process to inspect and set limits on its consumption of system resources (e.g., CPU time, file size).

**The `struct rlimit`:**
```c
struct rlimit {
    rlim_t rlim_cur;  // Soft limit (current, enforced limit)
    rlim_t rlim_max;  // Hard limit (ceiling for the soft limit)
};
```
*   **Soft Limit:** The actual limit enforced by the kernel. A process can adjust it up to the hard limit.
*   **Hard Limit:** The maximum value the soft limit can be raised to. Only a superuser process can increase a hard limit.

**Common Resource Limits (`resource` parameter):**

*   **`RLIMIT_CORE`:** Maximum size (in bytes) of a core dump file.
    *   If set to 0, core files are not created.
    *   A core dump is a file containing the process's memory image at the time of termination, useful for debugging.

*   **`RLIMIT_CPU`:** Maximum amount of CPU time (in seconds) the process can consume.
    *   On reaching the soft limit, `SIGXCPU` is sent repeatedly. Finally, `SIGKILL` is sent upon reaching the hard limit.

*   **`RLIMIT_DATA`:** Maximum size of the process's data segment (heap + data section). Affects `brk()` and `sbrk()`. `malloc()` will fail with `ENOMEM` if this limit is hit.

*   **`RLIMIT_FSIZE`:** Maximum size of files that the process can create.
    *   Attempts to write beyond this limit will result in a `SIGXFSZ` signal and the file write failing.

*   **`RLIMIT_STACK`:** Maximum size of the process stack (in bytes).
    *   Exceeding this limit causes `SIGSEGV` (segmentation fault).

**Example: Setting File Size Limit**
```c
#include <stdio.h>
#include <sys/resource.h>
#include <signal.h>

int main() {
    struct rlimit lim;
    // Get current limits
    getrlimit(RLIMIT_FSIZE, &lim);
    printf("Current Soft: %lu, Hard: %lu\n", (unsigned long)lim.rlim_cur, (unsigned long)lim.rlim_max);

    // Set a new soft limit (e.g., 5 bytes)
    lim.rlim_cur = 5;
    setrlimit(RLIMIT_FSIZE, &lim);

    // Try to create a file larger than 5 bytes
    FILE *fp = fopen("test.txt", "w");
    fwrite("abcdefgh", 1, 8, fp); // This will fail and likely generate SIGXFSZ
    fclose(fp);
    return 0;
}
```

---

#### **3. File System and Inodes**

**Key Concept:** The Linux file system is built upon **inodes** (index nodes), which are data structures that store all metadata about a file (except its name).

**File System Layout:**
1.  **Boot Block:** Contains the bootloader.
2.  **Super Block:** Contains information about the entire filesystem (size, free blocks, inode count).
3.  **Inode Table:** A table containing all the inodes for the filesystem.
4.  **Data Blocks:** The actual contents of the files and directories.

**What's in an Inode?**
*   File type (regular, directory, socket, etc.)
*   Permissions (`rwx` for user, group, others)
*   Owner (User ID) and Group ID
*   File size
*   Timestamps (creation, modification, access)
*   Link count (number of hard links)
*   Pointers to the data blocks on the disk.

**The `stat` System Call:**
*   `int stat(const char *pathname, struct stat *statbuf);`
*   Retrieves the inode information for a file and populates a `struct stat`.

**Key `struct stat` Members:**
*   `st_ino`: Inode number
*   `st_mode`: File type and mode (permissions)
*   `st_uid`: User ID of owner
*   `st_gid`: Group ID of owner
*   `st_size`: Total size in bytes
*   `st_nlink`: Number of hard links

**Macros for `st_mode`:**
*   `S_ISREG(m)`: Is it a regular file?
*   `S_ISDIR(m)`: Is it a directory?
*   `S_ISCHR(m)`: Is it a character device?
*   `S_ISBLK(m)`: Is it a block device?
*   `S_ISFIFO(m)`: Is it a FIFO (named pipe)?
*   `S_ISLNK(m)`: Is it a symbolic link?
*   `S_ISSOCK(m)`: Is it a socket?

**Example: Getting File Information**
```c
#include <sys/stat.h>
#include <stdio.h>
int main(int argc, char **argv) {
    struct stat sb;
    if (argc != 2) { fprintf(stderr, "Usage: %s <file>\n", argv[0]); return 1; }
    if (stat(argv[1], &sb) == -1) { perror("stat"); return 1; }
    printf("File size: %ld bytes\n", (long)sb.st_size);
    printf("Inode: %lu\n", (unsigned long)sb.st_ino);
    if (S_ISREG(sb.st_mode)) printf("It's a regular file.\n");
    return 0;
}
```

---

#### **4. Hard Links vs. Soft Links (Symbolic Links)**

| Feature | Hard Link | Soft (Symbolic) Link |
| :--- | :--- | :--- |
| **Inode** | Shares the **same inode** as the original file. | Has its **own, different inode**. |
| **Essence** | Another directory entry pointing to the same data. | A special file containing a **path** to the target file. |
| **File System** | Must be on the **same filesystem**. | Can cross filesystem boundaries. |
| **Link Count** | Increases the link count. | Does not affect the original file's link count. |
| **If Original Deleted** | Data remains accessible via the hard link. Link count decrements. | Becomes a **dangling link** (broken). |
| **`ls -l`** | Looks like a normal file. | Shows `l` in permissions and points to the target. |
| **Size** | Same apparent size as original. | Size is the length of the pathname it contains. |

**Identifying Link Type Programmatically:**
1.  Use `stat` on both files.
2.  If `st_ino` (inode number) is the **same**, it's a **hard link**.
3.  If `st_ino` is different, use `lstat` on the potential link file. If `lstat` shows it's a symbolic link (`S_ISLNK` is true), then it's a **soft link**.

**Example:**
```c
#include <sys/stat.h>
#include <stdio.h>
int main(int argc, char **argv) {
    struct stat s1, s2;
    if (argc != 3) { printf("Usage: %s file1 file2\n", argv[0]); return 1; }
    stat(argv[1], &s1);
    stat(argv[2], &s2);
    if (s1.st_ino == s2.st_ino) {
        printf("Hard link or the same file.\n");
    } else {
        // Check if file2 is a symlink pointing to file1
        lstat(argv[2], &s2); // Use lstat on the potential link
        if (S_ISLNK(s2.st_mode)) {
            printf("Soft link.\n");
        } else {
            printf("No link.\n");
        }
    }
    return 0;
}
```

---

#### **5. Directory Handling (`opendir`, `readdir`)**

**Key Concept:** Directories are special files that act as tables mapping filenames to inode numbers. We use specific functions to read their contents.

**Key Functions:**
*   **`DIR *opendir(const char *name);`**: Opens a directory stream. Returns a directory stream pointer or `NULL` on error.
*   **`struct dirent *readdir(DIR *dirp);`**: Reads the next directory entry. Returns a pointer to a `struct dirent` or `NULL` on end-of-stream or error.
*   **`int closedir(DIR *dirp);`**: Closes the directory stream.

**The `struct dirent`:**
*   Contains at least `d_ino` (inode number) and `d_name` (filename).

**Example: Listing a Directory**
```c
#include <stdio.h>
#include <dirent.h>
int main(int argc, char **argv) {
    DIR *dp;
    struct dirent *ep;
    char *dir = argc == 2 ? argv[1] : "."; // Use current dir if none provided
    dp = opendir(dir);
    if (dp != NULL) {
        while ((ep = readdir(dp)) != NULL) {
            printf("%s\n", ep->d_name);
        }
        closedir(dp);
    } else {
        perror("Couldn't open the directory");
    }
    return 0;
}
```

---

### **Summary of Key Concepts**

1.  **`sigaction` Flags:** `SA_NOCLDSTOP`, `SA_NOCLDWAIT`, `SA_NODEFER`, `SA_RESETHAND` provide fine-grained control over signal behavior.
2.  **Resource Limits:** Use `getrlimit`/`setrlimit` to manage process resources like CPU time (`RLIMIT_CPU`), file size (`RLIMIT_FSIZE`), and core dump size (`RLIMIT_CORE`).
3.  **Inodes:** The fundamental data structure storing all file metadata. Use `stat` to retrieve it.
4.  **File Types:** Use macros like `S_ISREG()` on `st_mode` to determine if a file is regular, directory, link, etc.
5.  **Hard vs. Soft Links:** Hard links share an inode; soft links are separate files containing a path. Use `stat` and `lstat` to distinguish them.
6.  **Directory Reading:** Use `opendir()`, `readdir()`, and `closedir()` to programmatically list directory contents.


---

### **Linux System Programming (Directories, Time, File I/O, and Pipes)**

#### **1. Directory Reading (`opendir`, `readdir`)**

**Key Concept:** Directories are special files that map filenames to inode numbers. The `opendir()`, `readdir()`, and `closedir()` functions are used to read their contents programmatically.

**Key Functions & Structures:**
*   **`DIR *opendir(const char *name);`**: Opens a directory stream.
*   **`struct dirent *readdir(DIR *dirp);`**: Reads the next directory entry.
*   **`int closedir(DIR *dirp);`**: Closes the directory stream.

**The `struct dirent`:**
*   `d_ino`: The inode number.
*   `d_name`: The filename (null-terminated string).
*   `d_type` (if supported): The file type (e.g., `DT_REG` for regular file, `DT_DIR` for directory).

**Example: Implementing a Simple `ls` Command**
```c
#include <stdio.h>
#include <dirent.h>

int main(int argc, char **argv) {
    DIR *dp;
    struct dirent *entry;
    char *dirname = (argc == 2) ? argv[1] : "."; // Use current dir if none provided

    dp = opendir(dirname);
    if (dp == NULL) {
        perror("opendir");
        return 1;
    }

    while ((entry = readdir(dp)) != NULL) {
        // Skip hidden files (those starting with '.')
        if (entry->d_name[0] != '.') {
            printf("%s  ", entry->d_name);
        }
    }
    printf("\n");
    closedir(dp);
    return 0;
}
```

**Assignment Idea: Recursive File Search**
*   Write a program that takes a starting path and a filename as input.
*   It should recursively search all directories from that path onward and count how many times the given filename appears.
*   **Logic:**
    1.  Open the starting directory.
    2.  Loop through each entry.
    3.  If an entry is a directory (and not '.' or '..'), recursively call the search function on that subdirectory.
    4.  If an entry is a file and its name matches the target, increment the counter.

---

#### **2. File Timestamps**

**Key Concept:** The inode stores three critical timestamps for a file, which can be accessed via the `struct stat` from the `stat()` system call.

**The Three Timestamps:**
*   **`st_atime` (Access Time):** Last time the file's **data was accessed** (e.g., by `read`, `cat`).
*   **`st_mtime` (Modification Time):** Last time the file's **content (data) was modified** (e.g., by `write`). This is the most important for tracking changes.
*   **`st_ctime` (Change Time):** Last time the file's **inode metadata was changed** (e.g., permissions, owner, link count). Note: Changing the file's data also changes its `st_ctime` because the file size and last modification time in the inode are updated.

**Example: Printing File Timestamps**
```c
#include <stdio.h>
#include <sys/stat.h>
#include <unistd.h>
#include <time.h>

int main(int argc, char **argv) {
    struct stat sb;
    if (argc != 2) {
        fprintf(stderr, "Usage: %s <filename>\n", argv[0]);
        return 1;
    }
    if (stat(argv[1], &sb) == -1) {
        perror("stat");
        return 1;
    }
    printf("Last access:      %s", ctime(&sb.st_atime));
    printf("Last modification: %s", ctime(&sb.st_mtime));
    printf("Last status change: %s", ctime(&sb.st_ctime));
    return 0;
}
```
*   **`time_t`** is an arithmetic type (often `long`) representing seconds since the Epoch (00:00:00 UTC, January 1, 1970).
*   **`ctime(const time_t *timep)`** converts a `time_t` value to a human-readable string.

---

#### **3. Low-Level File I/O (System Calls)**

**Key Concept:** The `open`, `read`, `write`, and `close` system calls provide direct, unbuffered access to files, offering more control than the standard I/O library (`fopen`, `fprintf`).

**The `open()` System Call**
*   `int open(const char *pathname, int flags);`
*   `int open(const char *pathname, int flags, mode_t mode);` (when creating a file)
*   **`flags`** specify the access mode and optional behavior:
    *   **Mandatory Flags (use one):**
        *   `O_RDONLY`: Open for reading only.
        *   `O_WRONLY`: Open for writing only.
        *   `O_RDWR`: Open for reading and writing.
    *   **Optional Flags (combine with `|`):**
        *   `O_CREAT`: Create the file if it doesn't exist. **Requires the `mode` argument.**
        *   `O_TRUNC`: Truncate the file to length 0 if it exists.
        *   `O_APPEND`: Move the file offset to the end before each write.

**File Descriptor:** A small, non-negative integer returned by `open()`, which is a handle (index into the kernel's per-process file descriptor table) used for all subsequent I/O operations on that file.

**Equivalents between `fopen` and `open`:**
*   `fopen("file", "r")` → `open("file", O_RDONLY)`
*   `fopen("file", "w")` → `open("file", O_WRONLY | O_CREAT | O_TRUNC, 0644)`
*   `fopen("file", "a")` → `open("file", O_WRONLY | O_CREAT | O_APPEND, 0644)`

**The `read()` and `write()` System Calls**
*   **`ssize_t read(int fd, void *buf, size_t count);`**
    *   Attempts to read up to `count` bytes from `fd` into `buf`.
    *   **Returns:** Number of bytes read, 0 on EOF, -1 on error.
    *   **Does not null-terminate the buffer!**
*   **`ssize_t write(int fd, const void *buf, size_t count);`**
    *   Attempts to write `count` bytes from `buf` to `fd`.
    *   **Returns:** Number of bytes written, -1 on error.

**Important:** Always check the return values of `read` and `write` as they may read/write fewer bytes than requested.

**Initializing Buffers:**
*   **`bzero(void *s, size_t n);`**: Sets the first `n` bytes of the memory area to zero (`\0`).
*   **`memset(void *s, int c, size_t n);`**: Sets the first `n` bytes of the memory area to the value `c`.

**Example: Reading from a File**
```c
#include "header.h" // Assume this includes all necessary headers
int main() {
    int fd;
    char buf[20];
    ssize_t bytes_read;

    fd = open("data.txt", O_RDONLY);
    if (fd < 0) { perror("open"); return 1; }

    bzero(buf, sizeof(buf)); // Zero out the buffer
    bytes_read = read(fd, buf, 5); // Read 5 bytes
    if (bytes_read < 0) { perror("read"); return 1; }

    printf("Read %zd bytes: %s\n", bytes_read, buf);
    close(fd);
    return 0;
}
```

---

#### **4. I/O Redirection using `dup2`**

**Key Concept:** I/O redirection (like `ls > file.txt`) is achieved by manipulating file descriptors. The `dup2(int oldfd, int newfd)` system call is the key. It closes `newfd` and then makes `newfd` an alias for `oldfd`.

**How `printf` and `scanf` work:**
*   `printf` writes to **Standard Output (stdout)**, which is **file descriptor 1**.
*   `scanf` reads from **Standard Input (stdin)**, which is **file descriptor 0**.

**Redirecting Standard Output to a File:**
1.  `close(1);` // Close the standard output descriptor.
2.  `open("file", O_WRONLY|O_CREAT|O_TRUNC, 0644);` // The new file will get the lowest available FD, which is now 1.
3.  Now, any `printf` statement will write to "file" instead of the screen.

**Example: Redirecting `printf` to a File**
```c
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
int main() {
    int fd;
    close(1); // Close stdout
    fd = open("output.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    printf("This will be written to output.txt, not the screen.\n");
    printf("File descriptor for the opened file is: %d\n", fd); // This will print '1'
    close(fd);
    return 0;
}
```

---

#### **5. Pipes for Inter-Process Communication (IPC)**

**Key Concept:** A pipe is a kernel-managed, unidirectional communication channel. It has a **read end** and a **write end**. Data written to the write end can be read from the read end. It's ideal for communication between related processes (e.g., parent and child).

**Creating a Pipe:**
*   `int pipe(int pipefd[2]);`
*   On success, `pipefd[0]` is the **read end**, and `pipefd[1]` is the **write end**.

**Key Pipe Behaviors:**
1.  **Empty Pipe:** A `read()` on an empty pipe will **block** until data is written.
2.  **Full Pipe:** A `write()` to a full pipe will **block** until space is available.
3.  **EOF:** If all write ends are closed, a `read()` will return 0 (EOF).
4.  **`SIGPIPE`:** If all read ends are closed, a `write()` will fail and generate a `SIGPIPE` signal.

**Example: Parent writes, Child reads**
```c
#include <stdio.h>
#include <unistd.h>
#include <string.h>
int main() {
    int p[2];
    pid_t pid;
    char buf[256];

    if (pipe(p) == -1) { perror("pipe"); return 1; }

    pid = fork();
    if (pid == 0) {
        /* Child process: reads from pipe */
        close(p[1]); // Close unused write end
        read(p[0], buf, sizeof(buf));
        printf("Child received: %s\n", buf);
        close(p[0]);
    } else {
        /* Parent process: writes to pipe */
        close(p[0]); // Close unused read end
        char *msg = "Hello from parent!";
        write(p[1], msg, strlen(msg) + 1); // Include null terminator
        close(p[1]);
    }
    return 0;
}
```

**Bidirectional Communication:** Use **two pipes**, one for each direction (Parent->Child and Child->Parent).

**Non-Blocking I/O:**
*   Use `pipe2(p, O_NONBLOCK)` to create a pipe in non-blocking mode.
*   In this mode, `read()` and `write()` return `-1` with `errno` set to `EAGAIN` immediately instead of blocking.
*   This is useful for checking if data is available without getting stuck.

**Pipe Capacity:**
*   The capacity of a pipe is finite (traditionally 64KB on many systems).
*   A continuous write loop will eventually fill the pipe and block (or fail with `EAGAIN` in non-blocking mode).

---

### **Summary of Key Concepts**

1.  **Directory Reading:** Use `opendir()`, `readdir()`, and `closedir()` to programmatically list directory contents.
2.  **File Timestamps:** `st_atime` (access), `st_mtime` (content change), `st_ctime` (metadata change).
3.  **Low-Level File I/O:** `open()`, `read()`, `write()`, `close()` offer direct file access. Always check return values.
4.  **I/O Redirection:** Use `dup2()` or strategically `close()` and `open()` to redirect stdin (0), stdout (1), and stderr (2).
5.  **Pipes:** Created with `pipe()`. Unidirectional communication channel between related processes. Understand blocking behavior and pipe capacity. Use two pipes for bidirectional communication.



---

### **Linux System Programming (Pipes, FIFOs, Message Queues, and Shared Memory)**

#### **1. Unnamed Pipes (Review & Key Behaviors)**

**Key Concept:** Unnamed pipes provide a unidirectional communication channel between related processes (e.g., parent and child). They exist only in the kernel and have no filesystem presence.

**Critical Pipe Behaviors:**
*   **Reading from an Empty Pipe:** A `read()` call will **block** until data is available.
*   **Writing to a Full Pipe:** A `write()` call will **block** until there is sufficient space in the pipe buffer.
*   **End-of-File (EOF):** If all **write ends** of a pipe are closed, a `read()` will return `0` (indicating EOF).
*   **Broken Pipe (`SIGPIPE`):** If all **read ends** of a pipe are closed, a `write()` will cause the kernel to send a `SIGPIPE` signal to the writing process. If the signal is not handled, this will terminate the process. If handled, `write()` will return `-1` with `errno` set to `EPIPE`.

**Example: Sequential Reading from a Pipe**
This demonstrates that data in a pipe is a byte stream; multiple reads can consume the data written in a single write.
```c
#include "header.h"
int main() {
    int p[2];
    if (pipe(p) < 0) { perror("pipe"); return 1; }

    if (fork() == 0) {
        // Child process: Reads in two parts
        char s[20];
        printf("In child before read...\n");
        read(p[0], s, 5); // Read first 5 bytes
        s[5] = '\0'; // Null-terminate
        printf("Data1 = %s\n", s);
        sleep(5);
        bzero(s, sizeof(s));
        read(p[0], s, 10); // Read next 10 bytes
        printf("Data2 = %s\n", s);
    } else {
        // Parent process: Writes one string
        char a[20];
        printf("Enter a string: ");
        scanf("%s", a);
        write(p[1], a, strlen(a) + 1); // Write entire string
        wait(NULL); // Wait for child to finish
    }
    return 0;
}
```
**Input/Output Example:**
*   **Input:** "HelloWorld123"
*   **Output:**
    *   `Data1 = Hello`
    *   (5 second pause)
    *   `Data2 = World123`

---

#### **2. FIFOs (Named Pipes)**

**Key Concept:** A FIFO (First-In, First-Out), or **named pipe**, is similar to an unnamed pipe but has a presence in the filesystem. This allows **unrelated processes** to communicate.

**Key Characteristics:**
*   Appears as a special file type (`p`) when listed with `ls -l`.
*   The file **size is always 0**; data is passed through the kernel, not stored on disk.
*   Must be **opened by both a reader and a writer** for either end to proceed past the `open()` call. A lone `open()` for reading or writing will **block** until the other end is also opened.

**Creating a FIFO:**
1.  **Shell Command:** `mkfifo my_fifo`
2.  **System Call:** `int mkfifo(const char *pathname, mode_t mode);`

**Example: Basic FIFO Communication**

**Process 1 (Writer):**
```c
#include "header.h"
int main() {
    int fd;
    char a[20];
    mkfifo("my_fifo", 0644);
    printf("Writer: Opening FIFO...\n");
    fd = open("my_fifo", O_WRONLY); // Blocks until a reader opens
    printf("Writer: FIFO opened.\n");
    while(1) {
        printf("Enter data: ");
        scanf("%s", a);
        write(fd, a, strlen(a)+1);
    }
    return 0;
}
```

**Process 2 (Reader):**
```c
#include "header.h"
int main() {
    int fd;
    char a[20];
    mkfifo("my_fifo", 0644);
    printf("Reader: Opening FIFO...\n");
    fd = open("my_fifo", O_RDONLY); // Blocks until a writer opens
    printf("Reader: FIFO opened.\n");
    while(1) {
        read(fd, a, sizeof(a));
        printf("Data: %s\n", a);
    }
    return 0;
}
```

**Full-Duplex Communication:** To achieve two-way communication between two unrelated processes, use **two FIFOs** (e.g., `fifo1` for A->B, `fifo2` for B->A).

**Advantages of FIFOs over Pipes:** Communication between unrelated processes.
**Disadvantages of FIFOs:**
*   Requires synchronization (both reader and writer must be active).
*   No built-in addressing mechanism; all readers see all messages.

---

#### **3. System V Message Queues**

**Key Concept:** Message queues provide a structured way for processes to exchange data in the form of messages. Each message has a type, allowing readers to selectively receive messages based on type.

**Key Features:**
*   **Structured Communication:** Data is sent in predefined message structures.
*   **Message Typing:** Messages have a type, allowing for priority or selective reading.
*   **Kernel Persistence:** Exist until explicitly removed or the system reboots.
*   **Asynchronous Communication:** Writers and readers do not need to be active simultaneously.

**Key Functions:**
1.  **`msgget()`:** Creates or opens a message queue.
2.  **`msgsnd()`:** Sends a message to the queue.
3.  **`msgrcv()`:** Receives a message from the queue.
4.  **`msgctl()`:** Performs control operations (e.g., get status, remove queue).

**The Message Buffer Structure:**
```c
struct msgbuf {
    long mtype;       // Message type (must be > 0)
    char mtext[1];    // Message data (can be any size)
};
// In practice, you define your own structure:
struct my_msgbuf {
    long mtype;
    char data[256];
    int some_other_field;
};
```

**Example: Sender and Receiver**

**Sender Program (`sender.c`):**
```c
#include "header.h"
struct my_msgbuf { long mtype; char data[20]; };

int main(int argc, char **argv) {
    struct my_msgbuf v;
    int id;
    if (argc != 3) {
        printf("Usage: %s <message_type> <message_data>\n", argv[0]);
        return 1;
    }
    id = msgget(1234, IPC_CREAT | 0644); // Key 1234
    v.mtype = atoi(argv[1]);
    strcpy(v.data, argv[2]);
    msgsnd(id, &v, strlen(v.data)+1, 0);
    printf("Sent: Type=%ld, Data=%s\n", v.mtype, v.data);
    return 0;
}
```

**Receiver Program (`receiver.c`):**
```c
#include "header.h"
struct my_msgbuf { long mtype; char data[20]; };

int main(int argc, char **argv) {
    struct my_msgbuf v;
    int id;
    if (argc != 2) {
        printf("Usage: %s <message_type_to_read>\n", argv[0]);
        return 1;
    }
    id = msgget(1234, IPC_CREAT | 0644); // Same key as sender
    msgrcv(id, &v, sizeof(v.data), atoi(argv[1]), 0);
    printf("Received: Type=%ld, Data=%s\n", v.mtype, v.data);
    return 0;
}
```

**Key `msgrcv()` Behavior:**
*   `msgrcv(id, &buf, size, msgtype, flags)`
*   **`msgtype = 0`:** Read the first message in the queue (any type).
*   **`msgtype > 0`:** Read the first message of that specific type.
*   **`msgtype < 0`:** Read the first message with the lowest type that is less than or equal to the absolute value of `msgtype`.

**Useful Flags:**
*   **`IPC_NOWAIT`:** Makes `msgsnd`/`msgrcv` non-blocking. Returns immediately with `-1` and `errno=ENOMSG` if the operation cannot be performed.
*   **`IPC_RMID`:** Used with `msgctl()` to remove a message queue from the system.

**Managing Message Queues:**
*   **View:** `ipcs -q`
*   **Remove:** `ipcrm -q <msqid>` or `ipcrm -Q <key>`

**Advantages:** Structured, typed messages; processes don't need to be related.
**Disadvantage:** Slower than other IPC mechanisms due to kernel involvement.

---

#### **4. System V Shared Memory**

**Key Concept:** Shared memory allows two or more processes to access a common segment of memory. This is the **fastest** form of IPC because it involves no kernel data copying after the initial setup.

**Key Functions:**
1.  **`shmget()`:** Creates or opens a shared memory segment.
2.  **`shmat()`:** Attaches the shared memory segment to the process's address space.
3.  **`shmdt()`:** Detaches the shared memory segment from the process.
4.  **`shmctl()`:** Performs control operations (e.g., destroy the segment).

**Workflow:**
1.  A process creates a shared memory segment with `shmget()`.
2.  Processes attach to the segment using `shmat()`, which returns a pointer to the shared memory.
3.  Processes read and write directly to this memory using the pointer.
4.  When done, processes detach with `shmdt()`.
5.  The segment is destroyed with `shmctl(..., IPC_RMID, ...)`.

**Example: Writer and Reader**

**Writer Program (`shm_writer.c`):**
```c
#include "header.h"
#include <sys/shm.h>
int main() {
    int id;
    char *p;
    // Create a shared memory segment of 50 bytes with key 5678
    id = shmget(5678, 50, IPC_CREAT | 0644);
    if (id < 0) { perror("shmget"); return 1; }
    printf("SHM ID: %d\n", id);
    // Attach the segment; let OS choose the address (shmaddr=0)
    p = shmat(id, 0, 0);
    printf("Enter data to write to shared memory: ");
    scanf("%s", p); // Write directly to shared memory
    printf("Data written. Press Enter to detach.\n");
    getchar(); getchar();
    shmdt(p); // Detach
    return 0;
}
```

**Reader Program (`shm_reader.c`):**
```c
#include "header.h"
#include <sys/shm.h>
int main() {
    int id;
    char *p;
    // Get the existing shared memory segment with key 5678
    id = shmget(5678, 50, 0644);
    if (id < 0) { perror("shmget"); return 1; }
    p = shmat(id, 0, 0); // Attach
    printf("Data read from shared memory: %s\n", p); // Read directly
    shmdt(p); // Detach
    // Optional: Remove the segment (usually done by the creator)
    // shmctl(id, IPC_RMID, NULL);
    return 0;
}
```

**Important Notes on Shared Memory:**
*   **Synchronization:** Shared memory provides **no inherent synchronization**. If multiple processes are writing, you must use **semaphores** or other synchronization primitives to prevent race conditions.
*   **Persistence:** The shared memory segment persists in the kernel until explicitly removed or the system reboots.
*   **Viewing:** Use `ipcs -m` to list shared memory segments.
*   **Private Segments:** Using `IPC_PRIVATE` as the key creates a segment that is private, but its `shmid` can still be passed to child processes via `fork()`.

**Advantage:** Extremely fast, as data does not need to be copied between processes.
**Disadvantage:** Requires explicit synchronization to avoid data corruption.

---

### **Summary of Key Concepts**

1.  **Pipes vs. FIFOs:** Pipes are for related processes and are unnamed; FIFOs have a filesystem name and allow unrelated processes to communicate. Both block on empty reads/full writes.
2.  **Message Queues:** Provide structured, typed messaging. Useful for prioritized or selective communication. Persist until removed.
3.  **Shared Memory:** The fastest IPC. Processes map the same physical memory into their address spaces. **Critical:** Requires external synchronization (e.g., semaphores).
4.  **Persistence:**
    *   **Pipes/FIFOs:** Exist only while processes are using them.
    *   **Message Queues / Shared Memory:** Kernel-persistent; must be explicitly deleted.
5.  **Management Commands:**
    *   **`ipcs`:** List IPC facilities.
    *   **`ipcrm`:** Remove IPC facilities.



---



### **1. Shared Memory (`shmget`, `shmat`, `shmdt`)**

**Concept:** Shared memory allows two or more processes to share a common segment of memory. It is the fastest form of IPC because it avoids copying data between kernel and user space.

**Key Functions:**

*   `int shmget(key_t key, size_t size, int shmflg);`
    *   **Purpose:** Creates a new or gets an existing shared memory segment.
    *   **Parameters:**
        *   `key`: Unique identifier (like 5 in the notes). Use `IPC_PRIVATE` for a new segment guaranteed to be unique.
        *   `size`: Size of the segment. The kernel rounds this up to a multiple of the system's page size (`PAGE_SIZE`).
        *   `shmflg`: Permissions (e.g., `0644`) combined with flags like `IPC_CREAT` (create if it doesn't exist).
*   `void *shmat(int shmid, const void *shmaddr, int shmflg);`
    *   **Purpose:** Attaches the shared memory segment to the address space of the calling process.
    *   **Parameters:**
        *   `shmid`: ID returned by `shmget`.
        *   `shmaddr`: Suggested attachment address. **Usually set to `NULL`**, letting the kernel choose a suitable address.
        *   `shmflg`: Flags like `SHM_RDONLY` (attach for read-only access).
    *   **Return Value:** On success, returns the address of the attached shared memory. On error, returns `(void *) -1`.
*   `int shmdt(const void *shmaddr);`
    *   **Purpose:** Detaches the shared memory segment from the process's address space.
    *   **Parameters:**
        *   `shmaddr`: The address returned by `shmat`.
    *   **Return Value:** 0 on success, -1 on error.

**Corrected Code Snippet:**
```c
#include <sys/ipc.h>
#include <sys/shm.h>
#include <stdio.h>
#include <stdlib.h>

int main() {
    int shmid;
    char *data;

    // Create a 50-byte shared memory segment
    shmid = shmget(5, 50, IPC_CREAT | 0644);
    if (shmid < 0) {
        perror("shmget");
        exit(1);
    }

    printf("shmid = %d\n", shmid);

    // Attach the segment. Let the kernel choose the address.
    data = (char *)shmat(shmid, NULL, 0);
    if (data == (char *)-1) {
        perror("shmat");
        exit(1);
    }

    // Read from the shared memory (assuming another process wrote to it)
    printf("DATA: %s\n", data);

    // Detach the segment
    if (shmdt(data) == -1) {
        perror("shmdt");
    }

    // Optional: Use 'shmctl(shmid, IPC_RMID, NULL)' to destroy the segment
    return 0;
}
```

**Important Notes from Document:**
*   The second parameter to `shmat` (`shmaddr`) is often `NULL` because the process doesn't know the address in advance.
*   **Disadvantage:** Lack of built-in **synchronization**. If two processes write simultaneously, data can be corrupted. You must use semaphores or other mechanisms to synchronize access.

---

### **2. Critical Section & The Need for Synchronization**

**Concept:** The part of the code where a process accesses a shared resource (e.g., shared memory, a file) is called the **Critical Section**.

**The Problem:**
*   If processes P1 and P2 enter their critical sections at the same time, they can corrupt the shared data.
*   The notes describe a simple "lock" variable (`x1`). If `x1==0`, the resource is free. A process sets it to 1 before entering, and back to 0 after leaving.
*   **The Flaw:** This method is unreliable. Checking `x1` and then setting it is not an **atomic operation** (it can be interrupted between the check and the set).

**Deadlock:**
*   The notes correctly identify a **deadlock** (called "dead process") scenario: P1 holds lock `x1` and waits for `x2`, while P2 holds lock `x2` and waits for `x1`. Both processes are stuck forever.

**Solution:** Use proper synchronization primitives provided by the OS, like **Semaphores** or **File Locking**.

---

### **3. File Descriptor Duplication (`dup`, `dup2`, `dup3`, `fcntl`)**

**Concept:** These system calls create a copy of an existing file descriptor. The new descriptor points to the same open file description in the kernel, sharing the file offset and status flags.

**Key Functions:**

*   `int dup(int oldfd);`
    *   **Purpose:** Duplicates `oldfd`. The new descriptor is the lowest-numbered available descriptor.
*   `int dup2(int oldfd, int newfd);`
    *   **Purpose:** Duplicates `oldfd` to a specific `newfd`. If `newfd` is already open, it is closed first. This is the most commonly used function, especially for I/O redirection.
*   `int fcntl(int fd, int cmd, ... /* arg */ );`
    *   **Purpose:** A versatile function for manipulating file descriptors.
    *   **Relevant `cmd` values:**
        *   `F_DUPFD`: Similar to `dup`, but finds the lowest available descriptor >= the provided argument.
        *   `F_GETFD` / `F_SETFD`: Get/Set file descriptor flags.
        *   `F_GETFL` / `F_SETFL`: Get/Set file status flags (e.g., `O_NONBLOCK` for non-blocking I/O).

**Corrected Pipe Example (from Page 10):**
This example uses `fork`, `pipe`, `dup2`, and `execlp` to run the `ps -el | grep pts/0` command.
```c
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>

int main() {
    int p[2];
    pipe(p); // p[0] is read-end, p[1] is write-end

    if (fork() == 0) {
        // Child Process: Will run "grep"
        close(p[1]);       // Close unused write-end
        dup2(p[0], 0);     // Duplicate read-end to stdin (fd 0)
        close(p[0]);       // Close the original read-end
        execlp("grep", "grep", "pts/0", NULL);
        perror("execlp grep");
        exit(1);
    } else {
        // Parent Process: Will run "ps"
        close(p[0]);       // Close unused read-end
        dup2(p[1], 1);     // Duplicate write-end to stdout (fd 1)
        close(p[1]);       // Close the original write-end
        execlp("ps", "ps", "-el", NULL);
        perror("execlp ps");
        exit(1);
    }
    return 0;
}
```

---

### **4. Advisory File Locking (`fcntl`)**

**Concept:** A mechanism to lock a region of a file (or the whole file) to prevent race conditions. "Advisory" means the lock is only effective if all cooperating processes explicitly check for locks.

**Key `fcntl` Commands for Locking:**
*   `F_SETLK`: Set or clear a lock non-blocking. Returns immediately if the lock cannot be acquired.
*   `F_SETLKW`: Like `F_SETLK`, but it **waits (blocks)** until the lock can be acquired.
*   `F_GETLK`: Check if a lock is present.

**The `flock` structure:**
```c
struct flock {
    short l_type;   // Lock type: F_RDLCK, F_WRLCK, F_UNLCK
    short l_whence; // How to interpret l_start: SEEK_SET, SEEK_CUR, SEEK_END
    off_t l_start;  // Starting offset for the lock
    off_t l_len;    // Number of bytes to lock; 0 means until EOF
    pid_t l_pid;    // PID of process holding the lock (filled by F_GETLK)
};
```

**Corrected Code Logic for W1.c and W2.c:**
The notes show two processes trying to write to a file. Using `fcntl` locking ensures one writes after the other.
```c
// Example for W1.c
#include "header.h" // Assume this includes necessary headers
int main() {
    struct flock v;
    char a[] = "Savam ";
    int fd, i;

    fd = open("data", O_RDWR | O_APPEND | O_CREAT, 0644);
    if (fd < 0) { perror("open"); return 1; }

    // Set up the lock structure for a WRITE lock on the entire file
    v.l_type = F_WRLCK;
    v.l_whence = SEEK_SET;
    v.l_start = 0;
    v.l_len = 0; // Lock entire file

    printf("Before Lock...\n");
    // Wait until the write lock is acquired
    fcntl(fd, F_SETLKW, &v);
    printf("I am in Critical Section...\n");

    // Write to the file
    for (i = 0; a[i]; i++) {
        write(fd, &a[i], 1);
        sleep(1); // Makes the interleaving obvious
    }

    // Unlock the file
    v.l_type = F_UNLCK;
    fcntl(fd, F_SETLK, &v);
    printf("Done.\n");
    return 0;
}
```
*(W2.c would be identical but with its own data string, e.g., "Bhalodya ")*

**Output:** You will see one process write its entire string without interruption, followed by the other, preventing corruption.

---

### **5. Semaphores (`semget`, `semctl`, `semop`)**

**Concept:** Semaphores are synchronization variables used to control access to a shared resource by multiple processes. They are the correct solution to the critical section problem.

**Key Functions:**

*   `int semget(key_t key, int nsems, int semflg);`
    *   **Purpose:** Creates/accesses an array of semaphores.
    *   `nsems`: Number of semaphores in the set.
*   `int semctl(int semid, int semnum, int cmd, ...);`
    *   **Purpose:** Control operations on the semaphore set.
    *   **Common `cmd` values:**
        *   `SETVAL`: Set the value of a semaphore to a specific number.
        *   `GETVAL`: Get the current value of a semaphore.
        *   `IPC_RMID`: Remove the semaphore set.
*   `int semop(int semid, struct sembuf *sops, size_t nsops);`
    *   **Purpose:** Perform atomic operations on semaphores. This is the core function for locking and unlocking.

**The `sembuf` structure:**
```c
struct sembuf {
    unsigned short sem_num;  // Semaphore index in the set (0-based)
    short          sem_op;   // Operation: Positive, Negative, or Zero
    short          sem_flg;  // Flags, e.g., IPC_NOWAIT, SEM_UNDO
};
```

**Semaphore Operations (`sem_op`):**

1.  **If `sem_op` is negative:** The process wants to acquire resources.
    *   If `semval` (current value) >= |`sem_op`|, then `semval` is decremented by |`sem_op`|, and the process continues.
    *   If `semval` < |`sem_op`|, the process blocks (waits) until enough resources are available.
    *   *This is the **P** or **Wait** operation.*

2.  **If `sem_op` is positive:** The process releases resources.
    *   `semval` is incremented by `sem_op`.
    *   *This is the **V** or **Signal** operation.*

3.  **If `sem_op` is zero:** The process waits for the semaphore to become zero.

**Types of Semaphores:**
*   **Binary Semaphore:** Acts as a mutex lock. Value is only 0 (locked) or 1 (unlocked).
*   **Counting Semaphore:** Allows a pre-defined number of processes (N) into a critical section. Its value can range from 0 to N.

**Using a Semaphore to Protect a File Write:**
The logic from the `fcntl` example can be implemented more cleanly with semaphores. Process W1 and W2 would both:
1.  `semget` the same semaphore set.
2.  Perform a `semop` with `sem_op = -1` (wait to enter critical section).
3.  Write to the file.
4.  Perform a `semop` with `sem_op = +1` (signal leaving critical section).

---

### **Summary of Important Concepts for Revision**

1.  **Shared Memory:** Fastest IPC. Remember `shmget`, `shmat`, `shmdt`. **Major Drawback:** Requires explicit synchronization.
2.  **Synchronization is Crucial:** Without it, you get **race conditions** and **data corruption**.
3.  **Deadlock:** A situation where two or more processes are unable to proceed because each is waiting for the other to release a resource.
4.  **File Descriptor Duplication:** Key for I/O redirection. Master `dup2`. Understand how it's used with `pipe` and `fork`.
5.  **Advisory File Locking:** Use `fcntl` with `F_SETLKW` and the `flock` struct to lock files and prevent concurrent write corruption.
6.  **Semaphores:** The standard solution for process synchronization.
    *   **`semop` is atomic.** This is why it works where a simple flag variable fails.
    *   **Negative `sem_op`** = Acquire/Lock/Wait.
    *   **Positive `sem_op`** = Release/Unlock/Signal.
    *   **Binary vs. Counting:** Know the difference.



---

### **Detailed & Corrected Notes on Linux System Programming (Part 2)**

### **1. Semaphores (Continued) - Controlling Critical Sections**

The notes show several programs (`W1.c`, `W2.c`) where two processes use a semaphore to coordinate access to a file, ensuring that one process writes its entire string before the other.

**Core Concept:** A semaphore value of 1 acts as a binary lock. A process performs a `semop` with `sem_op = -1` to "acquire" the lock (enter the critical section) and `sem_op = +1` to "release" it (exit the critical section). If the lock is held (semaphore value is 0), the acquiring process will block until it becomes available.

**Corrected and Simplified Logic for W1.c and W2.c:**

```c
// W1.c - Writes "Savam"
#include "header.h" // Assume standard headers and semaphore definitions

int main() {
    int sem_id;
    struct sembuf op;
    char data[] = "Savam ";
    int fd, i;

    // Get the existing semaphore set
    sem_id = semget(5, 1, 0666);
    if (sem_id < 0) { perror("semget"); return 1; }

    fd = open("data", O_WRONLY | O_APPEND | O_CREAT, 0644);
    if (fd < 0) { perror("open"); return 1; }

    // Set up the operation to LOCK (wait for semaphore to become 1, then set to 0)
    op.sem_num = 0;  // Use the first semaphore in the set
    op.sem_op = -1;  // Decrement semaphore by 1 (LOCK)
    op.sem_flg = 0;

    printf("W1: Waiting for lock...\n");
    semop(sem_id, &op, 1); // This will block if semaphore is 0
    printf("W1: In Critical Section...\n");

    // Write to file
    for (i = 0; data[i]; i++) {
        write(fd, &data[i], 1);
        sleep(1);
    }

    // Set up the operation to UNLOCK (set semaphore back to 1)
    op.sem_num = 0;
    op.sem_op = 1;   // Increment semaphore by 1 (UNLOCK)
    op.sem_flg = 0;

    semop(sem_id, &op, 1);
    printf("W1: Done. Lock released.\n");
    close(fd);
    return 0;
}
```
*(W2.c would be identical but would write its own string, e.g., "Bhalodya ")*

**Key Points:**
*   Both processes access the same semaphore set (key `5`).
*   The `sem_op = -1` operation is atomic. If the semaphore is 1, it decrements and proceeds. If it's 0, it waits. This prevents race conditions.
*   Using `semctl` with `SETVAL` inside the critical section (as shown in the original messy notes) is incorrect and breaks the locking mechanism. The locking should be done solely with `semop`.

---

### **2. POSIX Threads (pthreads)**

**Concept:** Threads are lightweight processes within a single process. They share the same address space, making communication between them fast and efficient.

**Why Use Threads?**
1.  **Faster Context Switching:** Switching between threads is faster than between processes because the memory map remains the same.
2.  **Efficient Communication:** Shared global variables allow easy data sharing without complex IPC.
3.  **Concurrency:** Allows a program to perform multiple tasks concurrently, improving performance on multi-core systems.

**Shared vs. Non-Shared Attributes:**
*   **Shared:** Most attributes are shared, including code, data, heap, and open file descriptors.
*   **Not Shared:** Each thread has its own Thread ID, stack, stack pointer, program counter, condition code, and general-purpose registers.

**Key pthread Functions:**
*   `pthread_create(pthread_t *thread, const pthread_attr_t *attr, void *(*start_routine) (void *), void *arg);`
    *   Creates a new thread executing `start_routine`.
*   `pthread_join(pthread_t thread, void **retval);`
    *   Waits for the specified thread to terminate. Analogous to `wait()` for processes.
*   `pthread_exit(void *retval);`
    *   Terminates the calling thread, returning `retval` to any joining thread.
*   `pthread_self(void);`
    *   Returns the thread ID of the calling thread.

**Corrected Basic Thread Example (from Page 10-11):**
```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>

void *thread_func(void *arg) {
    // Cast the argument back to the correct type
    char *message = (char *)arg;
    printf("Thread ID: %lu. Message: %s\n", pthread_self(), message);
    pthread_exit("Goodbye from thread");
}

int main() {
    pthread_t t1;
    void *thread_return;

    // Create a thread, passing a string as an argument
    if (pthread_create(&t1, NULL, thread_func, "Hello from main") != 0) {
        perror("pthread_create");
        return 1;
    }

    printf("Main: Created thread %lu\n", t1);

    // Wait for the thread to finish and get its return value
    pthread_join(t1, &thread_return);
    printf("Main: Thread returned: %s\n", (char *)thread_return);

    return 0;
}
```

---

### **3. Thread Synchronization: Mutexes**

**Concept:** Since threads share memory, they can corrupt shared data. A **mutex** (Mutual Exclusion) is a locking mechanism used to protect critical sections in a multi-threaded environment.

**Key Functions:**
*   `pthread_mutex_init(pthread_mutex_t *mutex, const pthread_mutexattr_t *attr);`
*   `pthread_mutex_lock(pthread_mutex_t *mutex);`
*   `pthread_mutex_unlock(pthread_mutex_t *mutex);`
*   `pthread_mutex_destroy(pthread_mutex_t *mutex);`

**Corrected Example for Alternating Output (from Page 17-18):**
The goal is to print alternating lower-case and upper-case letters (a, A, b, B, ...). This requires careful use of two mutexes to hand off control between threads.

```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>

pthread_mutex_t mutex1 = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t mutex2 = PTHREAD_MUTEX_INITIALIZER;

void *thread_lower(void *arg) {
    char c;
    for (c = 'a'; c <= 'z'; c++) {
        pthread_mutex_lock(&mutex1);   // Wait for turn to print lower-case
        printf("%c\n", c);
        pthread_mutex_unlock(&mutex2); // Signal upper-case thread to run
    }
    return NULL;
}

void *thread_upper(void *arg) {
    char c;
    for (c = 'A'; c <= 'Z'; c++) {
        pthread_mutex_lock(&mutex2);   // Wait for turn to print upper-case
        printf("%c\n", c);
        pthread_mutex_unlock(&mutex1); // Signal lower-case thread to run
    }
    return NULL;
}

int main() {
    pthread_t t1, t2;

    // Lock mutex2 so the upper-case thread waits initially
    pthread_mutex_lock(&mutex2);

    pthread_create(&t1, NULL, thread_lower, NULL);
    pthread_create(&t2, NULL, thread_upper, NULL);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    pthread_mutex_destroy(&mutex1);
    pthread_mutex_destroy(&mutex2);
    return 0;
}
```

---

### **4. Memory Management**

**Concept:** How an operating system manages RAM to run multiple processes efficiently.

**Key Concepts:**
*   **Binding:** Mapping a program's logical address to a physical RAM address. This can happen at compile time, load time, or execution time (most modern OSes).
*   **MMU (Memory Management Unit):** A hardware unit that translates virtual addresses to physical addresses at runtime.
*   **Partitioning:**
    *   **Fixed Partitioning:** RAM is divided into fixed-size partitions. Leads to **internal fragmentation** (wasted space within a partition).
    *   **Dynamic Partitioning:** Partitions are created to fit the exact size of processes. Leads to **external fragmentation** (free memory is broken into small, non-contiguous blocks).
*   **Solution to External Fragmentation:** **Defragmentation** (moving processes to consolidate free memory), but this is costly.

**Paging:**
*   Divides physical memory into fixed-sized blocks called **frames**.
*   Divides logical memory into blocks of the same size called **pages**.
*   The **Page Table** is a data structure that maps page numbers to frame numbers.
*   **Advantages:** Eliminates external fragmentation; allows a process's memory to be non-contiguous.
*   **Page Fault:** Occurs when a process accesses a page not currently in memory. The OS then loads the required page from disk (**page in**). If no free frame is available, an old page must be evicted (**page out**). A **dirty page** (one that has been modified) must be written back to disk.

---

### **5. 8051 Microcontroller Stack Operations (Bonus/Off-Topic)**

The notes briefly touch on the 8051 stack, which is a different context from Linux system programming.

*   **Stack Pointer (SP):** An 8-bit register that points to the last used location in the stack (which grows upwards in memory).
*   **PUSH:** Stores the contents of a specified register or memory location onto the stack. `SP` is first incremented, then data is copied.
*   **POP:** Retrieves data from the stack into a specified register or memory location. Data is copied, then `SP` is decremented.
*   The stack is used for temporary data storage, function calls, and saving context during interrupts.

**Example from Notes (Corrected):**
```assembly
CSEG AT 0H       ; Code segment starts at address 0
MOV R0, #35H     ; Load R0 with value 35H
MOV R5, #15H     ; Load R5 with value 15H
PUSH 00H         ; Push value in R0 (address 00H) onto stack
PUSH 05H         ; Push value in R5 (address 05H) onto stack
POP 00H          ; Pop top of stack into R0 (swaps values)
POP 05H          ; Pop next value into R5 (swaps values)
END
```
*This code exchanges the contents of R0 and R5 using the stack.*

---

### **Summary of Important Concepts for Revision (Part 2)**

1.  **Semaphore Operations:** Master the use of `semop` with `sem_op = -1` (lock/wait) and `sem_op = +1` (unlock/signal) to protect critical sections between processes.
2.  **POSIX Threads (pthreads):**
    *   Understand the `pthread_create`, `pthread_join`, `pthread_exit` lifecycle.
    *   Know that threads share the same address space, making communication easy but synchronization necessary.
3.  **Mutexes:** The primary tool for synchronizing threads. Know how to use `pthread_mutex_lock` and `pthread_mutex_unlock` to protect shared data from concurrent access.
4.  **Memory Management Fundamentals:**
    *   **Fragmentation:** Internal vs. External.
    *   **Paging:** The concepts of Pages, Frames, Page Tables, and the MMU.
    *   **Page Faults:** Understand what they are and the process of handling them (page-in, page-out).

