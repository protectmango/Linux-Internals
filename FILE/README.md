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

