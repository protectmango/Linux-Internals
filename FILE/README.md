# Chapter 4 : "File I/O: The Universal I/O Model" 

This chapter lays the foundation for all I/O operations in Linux by introducing the **universal I/O model**. This model uses a consistent set of system calls—`open()`, `read()`, `write()`, and `close()`—to handle all types of files, including regular disk files, directories, pipes, terminals, and sockets.

-----

### 1\. File Descriptors

A **file descriptor** is a small, non-negative integer used by the kernel to refer to an open file. It is the key to the universal I/O model. Each process has its own table of open file descriptors.

By default, every new process inherits three standard file descriptors:

  * **0 (`STDIN_FILENO`):** Standard input
  * **1 (`STDOUT_FILENO`):** Standard output
  * **2 (`STDERR_FILENO`):** Standard error

These are used for interacting with the terminal or for redirection in shell scripts.

-----

### 2\. Core I/O System Calls

#### `open()`

The `open()` system call is used to create or open a file. It returns a file descriptor that you will use for all subsequent operations on that file.

**C Code Example:**

```c
#include <fcntl.h> // For open()
#include <stdio.h> // For perror()

int fd = open("myfile.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);

if (fd == -1) {
    perror("open"); // Prints a descriptive error message
    // Handle error...
}
```

  * `"myfile.txt"`: The path to the file.
  * `O_WRONLY | O_CREAT | O_TRUNC`: A combination of flags specifying the file's access mode.
      * `O_WRONLY`: Open the file for writing only.
      * `O_CREAT`: Create the file if it doesn't exist.
      * `O_TRUNC`: Truncate the file to a length of 0 if it already exists.
  * `0644`: The file permissions (octal) if a new file is created.

#### `write()`

The `write()` system call writes a specified number of bytes from a buffer to an open file.

**C Code Example:**

```c
#include <unistd.h> // For write()
#include <string.h> // For strlen()

char buffer[] = "Hello, world!";
ssize_t bytesWritten = write(fd, buffer, strlen(buffer));

if (bytesWritten == -1) {
    perror("write");
    // Handle error...
}
```

  * `fd`: The file descriptor returned by `open()`.
  * `buffer`: The buffer containing the data to be written.
  * `strlen(buffer)`: The number of bytes to write.

#### `read()`

The `read()` system call reads a specified number of bytes from a file into a buffer. It returns the number of bytes read, which may be less than the requested amount. A return value of **0** indicates the end-of-file (EOF).

**C Code Example:**

```c
#include <unistd.h> // For read()

char readBuffer[100];
ssize_t bytesRead = read(fd, readBuffer, sizeof(readBuffer));

if (bytesRead == -1) {
    perror("read");
    // Handle error...
} else if (bytesRead == 0) {
    // End of file reached
}
```

  * `fd`: The file descriptor.
  * `readBuffer`: The buffer to store the data.
  * `sizeof(readBuffer)`: The maximum number of bytes to read.

#### `close()`

The `close()` system call closes an open file descriptor, releasing it and the resources associated with the file.

**C Code Example:**

```c
#include <unistd.h> // For close()

if (close(fd) == -1) {
    perror("close");
    // Handle error...
}
```

-----

### 3\. Shell Commands & File Descriptors

The universal I/O model is also the basis for standard shell commands. Commands like `ls`, `cat`, and `grep` operate on the concept of standard input and output.

  * **Pipes (`|`)**: The pipe operator connects the standard output of one command to the standard input of another.

      * `cat file.txt | grep "pattern"`: The standard output of `cat` (file.txt content) becomes the standard input of `grep`.

  * **Redirection (`>`, `<`, `>>`)**: These redirect standard I/O streams to or from files.

      * `echo "Hello" > output.txt`: Redirects standard output (file descriptor 1) to `output.txt`.
      * `grep "error" < log.txt`: Redirects standard input (file descriptor 0) from `log.txt`.

-----

### 4\. Other Key System Calls

  * **`lseek()`**: This system call is used to change the **file offset**, which is the position in the file where the next read or write operation will begin. This allows for random access within a file.
  * **`ioctl()`**: This call provides a catch-all mechanism for performing a wide variety of device-specific I/O operations that are not part of the standard I/O model. For example, it can be used to control a terminal's behavior.

These are excellent follow-up exercises that build on the foundational concepts from Chapter 4. Let's break down the solutions for each one.

### Exercise: `tee` Command Implementation

This exercise asks you to implement a simplified version of the `tee` command, which copies its standard input to both standard output and a specified file. It should also support an `-a` option to append to the file instead of overwriting it.

#### Solution: `tee.c`

The program needs to handle two main scenarios:

1.  **Default behavior:** Open the file, overwriting it if it exists.
2.  **`–a` option:** Open the file in append mode.

The `getopt()` function is a standard way to parse command-line options in C, as suggested by the exercise.

```c
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define BUF_SIZE 1024

int main(int argc, char *argv[]) {
    int opt;
    int append = 0;
    int outputFd;
    ssize_t numRead;
    char buf[BUF_SIZE];

    // Parse command-line options
    while ((opt = getopt(argc, argv, "a")) != -1) {
        switch (opt) {
            case 'a':
                append = 1;
                break;
            default: /* '?' */
                fprintf(stderr, "Usage: %s [-a] <file>\n", argv[0]);
                exit(EXIT_FAILURE);
        }
    }

    if (optind >= argc) {
        fprintf(stderr, "Expected file argument after options\n");
        exit(EXIT_FAILURE);
    }

    // Determine open flags based on the 'append' flag
    int openFlags = O_WRONLY | O_CREAT;
    if (append) {
        openFlags |= O_APPEND;
    } else {
        openFlags |= O_TRUNC;
    }

    // Open the file
    outputFd = open(argv[optind], openFlags, 0644);
    if (outputFd == -1) {
        perror("open output file");
        exit(EXIT_FAILURE);
    }

    // Read from standard input and write to standard output and the file
    while ((numRead = read(STDIN_FILENO, buf, BUF_SIZE)) > 0) {
        // Write to standard output
        if (write(STDOUT_FILENO, buf, numRead) != numRead) {
            perror("write to stdout");
            close(outputFd);
            exit(EXIT_FAILURE);
        }

        // Write to the output file
        if (write(outputFd, buf, numRead) != numRead) {
            perror("write to file");
            close(outputFd);
            exit(EXIT_FAILURE);
        }
    }

    if (numRead == -1) {
        perror("read from stdin");
        close(outputFd);
        exit(EXIT_FAILURE);
    }

    // Close the file
    if (close(outputFd) == -1) {
        perror("close output file");
        exit(EXIT_FAILURE);
    }

    return 0;
}
```

### Exercise: `cp` Program with Holes

This exercise requires a more sophisticated version of the `cp` command that preserves "holes" in a sparse file. A hole is a sequence of null (`\0`) bytes that the file system does not store on disk, saving space. Simply reading and writing the file as a stream of bytes will fill these holes, increasing the file's size.

The solution is to detect sequences of null bytes and, instead of writing them, use `lseek()` to advance the file offset in the destination file, effectively creating a hole.

#### Solution: `cp_sparse.c`

```c
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define BUF_SIZE 1024
#define HOLE_THRESHOLD 1 // A simple threshold: if one or more null bytes are found

int main(int argc, char *argv[]) {
    int inputFd, outputFd;
    ssize_t numRead, numWritten;
    char buf[BUF_SIZE];
    int nullCount;

    if (argc != 3) {
        fprintf(stderr, "Usage: %s <source_file> <dest_file>\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    inputFd = open(argv[1], O_RDONLY);
    if (inputFd == -1) {
        perror("open source file");
        exit(EXIT_FAILURE);
    }

    outputFd = open(argv[2], O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (outputFd == -1) {
        perror("open destination file");
        close(inputFd);
        exit(EXIT_FAILURE);
    }

    while ((numRead = read(inputFd, buf, BUF_SIZE)) > 0) {
        nullCount = 0;
        for (int i = 0; i < numRead; i++) {
            if (buf[i] == '\0') {
                nullCount++;
            }
        }

        if (nullCount >= HOLE_THRESHOLD) {
            // Found a sequence of null bytes, use lseek to create a hole.
            if (lseek(outputFd, numRead, SEEK_CUR) == -1) {
                perror("lseek");
                close(inputFd);
                close(outputFd);
                exit(EXIT_FAILURE);
            }
        } else {
            // Write non-null data
            numWritten = write(outputFd, buf, numRead);
            if (numWritten != numRead) {
                perror("write");
                close(inputFd);
                close(outputFd);
                exit(EXIT_FAILURE);
            }
        }
    }

    if (numRead == -1) {
        perror("read");
        close(inputFd);
        close(outputFd);
        exit(EXIT_FAILURE);
    }

    if (close(inputFd) == -1) {
        perror("close input");
        exit(EXIT_FAILURE);
    }
    if (close(outputFd) == -1) {
        perror("close output");
        exit(EXIT_FAILURE);
    }

    return 0;
}
```

# File I/O: Further Details

-----

### Atomicity and Race Conditions

[cite\_start]The concept of **atomicity** is central to preventing a common programming bug known as a **race condition**[cite: 361, 362, 363, 364]. [cite\_start]An atomic operation is guaranteed by the kernel to complete in a single, uninterruptible step[cite: 363].

[cite\_start]A race condition occurs when the outcome of an operation depends on the unpredictable scheduling order of two or more processes or threads accessing a shared resource[cite: 366, 367].

The book provides two examples to illustrate this problem:

1.  **Exclusively Creating a File:**
    [cite\_start]An incorrect way to exclusively create a file is to first check if it exists with `open()` and then, if it doesn't, call `open()` again with `O_CREAT`[cite: 373, 375]. [cite\_start]This introduces a "window for failure"[cite: 386]. [cite\_start]If the process is interrupted between the two `open()` calls, another process could create the file, leading to both processes incorrectly thinking they exclusively created it[cite: 400, 401, 403]. The atomic solution is to use the `O_CREAT | [cite_start]O_EXCL` flags in a single `open()` call, which ensures the check and creation happen as one operation[cite: 448].

2.  **Appending to a File:**
    [cite\_start]Similarly, a race condition can occur when multiple processes append data to the same file[cite: 450]. [cite\_start]A naive approach would be to use `lseek(fd, 0, SEEK_END)` followed by a `write()` call[cite: 452, 454]. [cite\_start]If a process is interrupted between these two calls, another process could also set its file offset to the same end position, and when the first process is rescheduled, it would overwrite the data written by the second process[cite: 457]. [cite\_start]The atomic solution is to use the `O_APPEND` flag when opening the file, which guarantees that the seek and write operations are performed as a single unit[cite: 460].

-----

### The `fcntl()` System Call

[cite\_start]The `fcntl()` system call provides a way to perform various control operations on an open file descriptor[cite: 464]. [cite\_start]Its second argument, `cmd`, specifies the operation[cite: 471].

#### `F_GETFL` and `F_SETFL`

[cite\_start]One of the most common uses of `fcntl()` is to retrieve or modify the file status flags (the flags used in the `open()` call, such as `O_APPEND` or `O_NONBLOCK`)[cite: 476, 477].

  * [cite\_start]**Retrieving flags:** Use the `F_GETFL` command to get the current flags[cite: 477, 479].
  * [cite\_start]**Modifying flags:** Use the `F_SETFL` command to change a subset of the flags[cite: 493]. [cite\_start]To do this, you must first retrieve the current flags with `F_GETFL`, modify the flags you want to change, and then call `F_SETFL` with the new flags[cite: 504].

**C Code Example:**

```c
#include <fcntl.h>
#include <stdio.h>
#include "tlpi_hdr.h"

int main(int argc, char *argv[]) {
    // open file for writing and appending
    int fd = open("log.txt", O_WRONLY | O_CREAT, 0644);
    if (fd == -1) errExit("open");

    // Retrieve current flags
    int flags = fcntl(fd, F_GETFL);
    if (flags == -1) errExit("fcntl F_GETFL");

    // Check if the file is writable
    if ((flags & O_ACCMODE) == O_WRONLY) {
        printf("File is writable\n");
    }

    // Add O_APPEND to the flags
    flags |= O_APPEND;

    // Set the new flags
    if (fcntl(fd, F_SETFL, flags) == -1) errExit("fcntl F_SETFL");
    printf("O_APPEND flag has been set\n");

    return 0;
}
```

-----

### Kernel Data Structures

[cite\_start]To fully understand file I/O, you need to know about the three main data structures the kernel uses to represent open files[cite: 515]:

1.  [cite\_start]**Per-process File Descriptor Table:** A table unique to each process that maps file descriptors (integers like 0, 1, 2) to open file descriptions[cite: 519].
2.  [cite\_start]**System-wide Open File Table (or Open File Description):** A system-wide table that stores all information related to an open file, including its current **file offset**, status flags (`O_APPEND`, `O_NONBLOCK`), and a reference to the file's i-node[cite: 523, 524].
3.  [cite\_start]**File System I-node Table:** A system-wide table where each entry (**i-node**) holds a file's persistent attributes, such as its type, permissions, size, and location on disk[cite: 530, 532, 534].

The relationship is illustrated as follows: A file descriptor points to an open file description, which in turn points to an i-node. Multiple file descriptors can point to the same open file description, and multiple open file descriptions can point to the same i-node.

-----

### Duplicating File Descriptors

[cite\_start]The `dup()` and `dup2()` system calls are used to duplicate file descriptors[cite: 595]. [cite\_start]This is how shell redirection, like `2>&1`, works[cite: 591, 594].

  * [cite\_start]**`dup(oldfd)`:** Creates a new file descriptor that is the lowest available unused integer[cite: 604]. [cite\_start]The new descriptor points to the same open file description as `oldfd`[cite: 603].
  * [cite\_start]**`dup2(oldfd, newfd)`:** Duplicates `oldfd` to the specific number `newfd`[cite: 619]. [cite\_start]If `newfd` is already open, it is closed first[cite: 620]. [cite\_start]This call is atomic and a safer way to get a specific file descriptor number[cite: 622].

**C Code Example (`dup2()`):**

```c
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>
#include "tlpi_hdr.h"

int main(void) {
    int fd;
    // Open a file; it will be assigned the lowest available descriptor (e.g., 3)
    fd = open("redirected.log", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd == -1) errExit("open");

    // Redirect standard output (descriptor 1) to the new file
    if (dup2(fd, 1) == -1) errExit("dup2");

    // This printf() will now write to the file
    printf("This text is now in the file, not the terminal!\n");

    // Close the original descriptor
    close(fd);

    return 0;
}
```

-----

### Extended I/O System Calls

  * [cite\_start]**I/O at a Specific Offset (`pread()`, `pwrite()`):** These calls perform I/O at a specified location in the file without changing the file offset[cite: 649, 650]. [cite\_start]This is particularly useful in multithreaded applications where multiple threads share the same file descriptor and need to avoid race conditions[cite: 661, 663, 664].
  * [cite\_start]**Scatter-Gather I/O (`readv()`, `writev()`):** These calls read from or write to multiple non-contiguous memory buffers in a single system call[cite: 669, 670]. [cite\_start]This is more efficient than performing multiple `read()` or `write()` calls[cite: 200, 201].
  * **Truncating a File (`truncate()`, `ftruncate()`):** These calls are used to change a file's size. If the new length is shorter, the file is truncated. [cite\_start]If it's longer, the file is extended with null bytes or a hole[cite: 208, 209].

### Solutions to Chapter 5 Exercises

Here are the solutions for the requested exercises, based on the concepts and examples presented in the provided Chapter 5 content.

-----

### Exercise 5-1: Creating a Large File

This exercise requires you to create a program that can create a large file (greater than 2 GB) using `open()` and `lseek()`, verifying the use of the `off_t` data type. This is achieved by compiling the program with the `_FILE_OFFSET_BITS=64` macro, which ensures `off_t` is a 64-bit integer, capable of holding large file offsets.

**Solution: `large_file.c`**

```c
#define _FILE_OFFSET_BITS 64
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>

int main(int argc, char *argv[]) {
    int fd;
    off_t offset;

    if (argc != 2) {
        fprintf(stderr, "Usage: %s <filename>\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    // Open file for writing, creating it if it doesn't exist.
    fd = open(argv[1], O_WRONLY | O_CREAT, 0644);
    if (fd == -1) {
        perror("open");
        exit(EXIT_FAILURE);
    }

    // Use lseek to move the file offset far beyond the normal 32-bit limit.
    // The value 2LL * 1024 * 1024 * 1024 represents 2 gigabytes.
    offset = 2LL * 1024 * 1024 * 1024 + 10;
    if (lseek(fd, offset, SEEK_SET) == -1) {
        perror("lseek");
        close(fd);
        exit(EXIT_FAILURE);
    }

    // Write a single byte to "create" the file with a hole.
    if (write(fd, "a", 1) != 1) {
        perror("write");
        close(fd);
        exit(EXIT_FAILURE);
    }

    printf("Successfully created a large file with a size of %lld bytes\n", (long long) offset + 1);

    close(fd);
    return 0;
}
```

**Explanation:**
[cite\_start]The `lseek()` system call is used to explicitly set the file offset[cite: 82]. By specifying an offset greater than the maximum value of a 32-bit `off_t` type, and compiling with `_FILE_OFFSET_BITS=64`, we ensure that the program can handle large file sizes. The `write()` call at that large offset creates a "hole" in the file, effectively setting its size to that value without writing all the intermediate null bytes.

-----

### Exercise 5-2: `O_APPEND` and `lseek()`

This exercise investigates what happens when a program opens a file with the `O_APPEND` flag and then calls `lseek()` before a `write()` call.

**Solution and Explanation:**

[cite\_start]When a file is opened with the `O_APPEND` flag, all subsequent `write()` calls are guaranteed to be atomic[cite: 48, 49]. The kernel ensures that the file offset is automatically set to the end of the file **before** each write operation, regardless of any preceding `lseek()` call.

Therefore, the data will **always be appended to the end of the file**. The `lseek()` call to the beginning of the file is effectively ignored by the kernel for `write()` operations on a file opened with `O_APPEND`.

[cite\_start]This is a crucial feature that prevents a race condition, where two processes could perform `lseek()` to the end of a file and then both attempt to write to the same location, overwriting each other's data[cite: 46]. [cite\_start]The `O_APPEND` flag solves this by making the seek and write a single, uninterruptible operation[cite: 48].

**Sample Code:**

```c
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char *argv[]) {
    int fd;

    if (argc != 2) {
        fprintf(stderr, "Usage: %s <filename>\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    // Open file with O_APPEND
    fd = open(argv[1], O_WRONLY | O_CREAT | O_APPEND, 0644);
    if (fd == -1) {
        perror("open");
        exit(EXIT_FAILURE);
    }

    // The data written will still appear at the end of the file.
    if (lseek(fd, 0, SEEK_SET) == -1) {
        perror("lseek");
        close(fd);
        exit(EXIT_FAILURE);
    }

    // This data will be appended to the end of the file, not the beginning.
    char *data = "This will be appended.\n";
    if (write(fd, data, strlen(data)) != strlen(data)) {
        perror("write");
        close(fd);
        exit(EXIT_FAILURE);
    }

    close(fd);
    return 0;
}
```

-----

### Exercise 5-7: Implementing `readv()` and `writev()`

This exercise asks you to implement simplified versions of the `readv()` and `writev()` scatter-gather I/O system calls using `read()`, `write()`, and `malloc()`.

[cite\_start]**Background:** The `readv()` and `writev()` calls are designed to be more efficient than multiple `read()` or `write()` calls[cite: 200, 196]. [cite\_start]An implementation using a single buffer would copy data from the user's provided buffers into a single contiguous buffer in memory, and then perform a single `read()` or `write()` on that buffer[cite: 196, 198]. [cite\_start]This approach, however, loses the atomicity of the original `readv()` and `writev()` calls[cite: 174, 192].

**Solution: `my_readv_writev.c`**

```c
#include <sys/uio.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>

// My implementation of readv()
ssize_t my_readv(int fd, const struct iovec *iov, int iovcnt) {
    ssize_t total_bytes = 0;
    for (int i = 0; i < iovcnt; i++) {
        ssize_t bytes_read = read(fd, iov[i].iov_base, iov[i].iov_len);
        if (bytes_read == -1) {
            return -1;
        }
        total_bytes += bytes_read;
        if (bytes_read < iov[i].iov_len) {
            // Partial read, return what was read so far
            break;
        }
    }
    return total_bytes;
}

// My implementation of writev()
ssize_t my_writev(int fd, const struct iovec *iov, int iovcnt) {
    ssize_t total_bytes = 0;
    for (int i = 0; i < iovcnt; i++) {
        ssize_t bytes_written = write(fd, iov[i].iov_base, iov[i].iov_len);
        if (bytes_written == -1) {
            return -1;
        }
        total_bytes += bytes_written;
        if (bytes_written < iov[i].iov_len) {
            // Partial write, return what was written so far
            break;
        }
    }
    return total_bytes;
}

// Alternative (more complex) implementation of writev() using malloc
ssize_t my_writev_alt(int fd, const struct iovec *iov, int iovcnt) {
    size_t total_len = 0;
    for (int i = 0; i < iovcnt; i++) {
        total_len += iov[i].iov_len;
    }

    // Allocate a single large buffer
    char *combined_buf = (char *)malloc(total_len);
    if (combined_buf == NULL) {
        errno = ENOMEM;
        return -1;
    }

    // Copy data from iovec buffers to the combined buffer
    size_t current_offset = 0;
    for (int i = 0; i < iovcnt; i++) {
        memcpy(combined_buf + current_offset, iov[i].iov_base, iov[i].iov_len);
        current_offset += iov[i].iov_len;
    }

    // Perform a single write call
    ssize_t bytes_written = write(fd, combined_buf, total_len);

    free(combined_buf);
    return bytes_written;
}

int main(void) {
    // This main function is just a placeholder to show the function definitions.
    // To test, you would need to create a file, define iovec structures, and call the functions.
    printf("Functions my_readv and my_writev are implemented.\n");
    return 0;
}
```
