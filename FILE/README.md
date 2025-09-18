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

