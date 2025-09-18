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


### Exercise 4-1: `cp` Program

The goal of this exercise is to write a simplified version of the `cp` command. The program should copy a file specified as its first command-line argument to a new file specified as the second argument. It should use `open()`, `read()`, `write()`, and `close()` system calls.

#### Solution

The program will open the source file for reading and the destination file for writing, creating it if it doesn't exist and truncating it if it does. It will then loop, reading a chunk of data from the source file and writing it to the destination file until the end of the source file is reached.

**C Code:**

```c
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>

#define BUF_SIZE 1024

int main(int argc, char *argv[]) {
    int inputFd, outputFd;
    ssize_t numRead;
    char buf[BUF_SIZE];

    if (argc != 3) {
        fprintf(stderr, "Usage: %s <source_file> <dest_file>\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    // Open source file for reading
    inputFd = open(argv[1], O_RDONLY);
    if (inputFd == -1) {
        perror("open source file");
        exit(EXIT_FAILURE);
    }

    // Open destination file for writing, creating it if it doesn't exist
    // and setting permissions to 0644.
    outputFd = open(argv[2], O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (outputFd == -1) {
        perror("open destination file");
        close(inputFd); // Close the source file before exiting
        exit(EXIT_FAILURE);
    }

    // Read from the input file and write to the output file
    while ((numRead = read(inputFd, buf, BUF_SIZE)) > 0) {
        if (write(outputFd, buf, numRead) != numRead) {
            perror("write");
            close(inputFd);
            close(outputFd);
            exit(EXIT_FAILURE);
        }
    }

    if (numRead == -1) {
        perror("read");
        close(inputFd);
        close(outputFd);
        exit(EXIT_FAILURE);
    }

    // Close both files
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

-----

### Exercise 4-2: File Descriptors and `dup2()`

This exercise involves writing a program that demonstrates how file descriptors are assigned and how they can be manipulated. The program should `open()` a file and print its file descriptor, and then use the `dup2()` system call to duplicate a file descriptor and show how it affects I/O operations.

#### Solution

The program will first open a file and print the assigned file descriptor. It will then use `dup2()` to make file descriptor `1` (standard output) refer to the same open file description as the first file. Subsequent `printf()` calls will then write to the file instead of the terminal.

**C Code:**

```c
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int fd;

    // Open a file; the first available descriptor will be assigned.
    // This is typically 3, as 0, 1, and 2 are already in use.
    fd = open("test_output.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd == -1) {
        perror("open");
        exit(EXIT_FAILURE);
    }
    
    // Print the file descriptor of the newly opened file.
    // This output will go to the terminal (stdout).
    printf("File opened with descriptor: %d\n", fd);

    // Now, we will duplicate file descriptor 1 (stdout) so that
    // it points to the same open file description as 'fd'.
    // All subsequent output to stdout will go to the file.
    if (dup2(fd, 1) == -1) {
        perror("dup2");
        close(fd);
        exit(EXIT_FAILURE);
    }

    // This printf() will now write its output to the 'test_output.txt' file
    // because file descriptor 1 has been redirected.
    printf("This output is now being redirected to the file.\n");

    // Close the original file descriptor.
    close(fd);

    return 0;
}
```

These solutions illustrate the fundamental concepts of file descriptors, the universal I/O model, and system calls like `open()`, `read()`, `write()`, and `dup2()` as discussed in Chapter 4 of the book.
