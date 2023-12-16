# Concurrent Programming with Processes
# 基于进程的并发编程

构造并发程序最简单的方法就是用进程，使用那些大家都很熟悉的函数，像 `fork`、`exec` 和 `waitpid`。
例如，一个构造并发服务器的自然方法就是，在父进程中接受客户端连接请求，然后创建一个新的子进程来为每个新客户端提供服务。

~~~c
#include "csapp.h"

// Function prototype for echo
void echo(int connfd);

// Signal handler for SIGCHLD
void sigchld_handler(int sig)
{
    // Reap all dead processes
    // waitpid with -1 waits for any child process
    while (waitpid(-1, 0, WNOHANG) > 0)
        ; // Empty loop body; waitpid cleans up child processes
    return;
}

// Main function of the server
int main(int argc, char **argv)
{
    int listenfd, connfd; // File descriptors for listener and connector
    socklen_t clientlen; // Byte size of client's address
    struct sockaddr_storage clientaddr; // Client address

    // Check command line args, expecting the port number
    if (argc != 2) {
        fprintf(stderr, "usage: %s <port>\n", argv[0]);
        exit(0);
    }

    // Set up the SIGCHLD signal handler
    Signal(SIGCHLD, sigchld_handler);

    // Open a listening socket
    listenfd = Open_listenfd(argv[1]);

    // Infinite loop for server to accept incoming connections
    while (1) {
        clientlen = sizeof(struct sockaddr_storage);
        // Accept a connection request from a client
        connfd = Accept(listenfd, (SA *) &clientaddr, &clientlen);

        // Create a child process to handle the client
        if (Fork() == 0) {
            Close(listenfd); // Child closes its listening socket
            echo(connfd);    // Child services client
            Close(connfd);   // Child closes connection with client
            exit(0);         // Child exits
        }

        // Parent closes connected socket (this is important to avoid resource leak)
        Close(connfd);
    }
}
~~~


~~~c
void echo(int connfd) {
    size_t n;
    char buf[MAXLINE];
    rio_t rio;

    Rio_readinitb(&rio, connfd);
    while ((n = Rio_readlineb(&rio, buf, MAXLINE)) != 0) {
        if (strcmp(buf, "\r\n") == 0)
        break;
        Rio_writen(connfd, buf, n);
    }
}
~~~

图 12-5 基于进程的并发 echo 服务器。父进程派生一个子进程来处理每个新的连接请求

第 29 行调用的 echo 函数来自于图 11-21。关于这个服务器，有几点重要内容需要说明：
 - 首先，通常服务器会运行很长的时间，所以我们必须要包括一个 SIGCHLD 处理程序，来回收僵死（zombie）子进程的资源（第 4~9 行）。因为当 SIGCHLD 处理程序执行时，SIGCHLD 信号是阻塞的，而 Linux 信号是不排队的，所以 SIGCHLD 处理程序必须准备好回收多个僵死子进程的资源。
 - 其次，父子进程必须关闭它们各自的 connfd（分别为第 33 行和第 30 行）副本。就像我们已经提到过的，这对父进程而言尤为重要，它必须关闭它的已连接描述符，以避免内存泄漏。
 - 最后，因为套接字的文件表表项中的引用计数，直到父子进程的 connfd 都关闭了，到客户端的连接才会终止。



----------------------------------------------------------------------------------------------------

在这段代码中，服务器使用信号处理来清理僵尸进程的方式是通过处理 SIGCHLD 信号。
理解这一点需要了解操作系统中进程的概念，尤其是关于父子进程和僵尸进程的处理。

### 僵尸进程（Zombie Process）
- 当一个子进程结束（或终止）时，它不会立即从系统中完全消失。尽管它已经停止执行，操作系统仍然保留了它的某些信息（如退出状态），以便父进程查询。
- 这个已终止但尚未被父进程清理（读取其退出状态）的进程称为僵尸进程。
- 僵尸进程占用系统资源，尽管不多，但如果大量累积，可能会导致资源耗尽。

### SIGCHLD 信号
- 当子进程结束时，操作系统向其父进程发送 SIGCHLD 信号。
- 默认情况下，大多数程序会忽略此信号，但可以通过编写信号处理程序来利用它。

### `sigchld_handler` 函数
- 在这个服务器代码中，`sigchld_handler` 函数是为 SIGCHLD 信号编写的处理程序。
- 它使用 `waitpid` 系统调用来检查是否有已结束的子进程。
- `waitpid(-1, 0, WNOHANG)` 的调用表示检查任何子进程（`-1`），不挂起调用进程（`WNOHANG`），如果有已结束的子进程，则回收它，释放它占用的资源，从而避免产生僵尸进程。

### 整体流程
1. **设置信号处理器**：在 `main` 函数中，使用 `Signal(SIGCHLD, sigchld_handler)` 设置了 SIGCHLD 的处理程序为 `sigchld_handler`。
2. **子进程终止**：当通过 `fork` 创建的子进程（用于处理客户端请求）完成其任务后，它会结束。
3. **接收 SIGCHLD**：父进程（服务器）接收到 SIGCHLD 信号，提示有子进程已终止。
4. **调用 `sigchld_handler`**：父进程调用 `sigchld_handler` 函数处理这个信号。
5. **清理僵尸进程**：`sigchld_handler` 使用 `waitpid` 循环来清理所有已终止的子进程，防止它们成为僵尸进程。

通过这种方式，服务器能够有效地管理资源，防止僵尸进程的积累，确保稳定运行。

----------------------------------------------------------------------------------------------------

# 进程的优劣
对于在父、子进程间共享状态信息，进程有一个非常清晰的模型：共享文件表，但是不共享用户地址空间。进程有独立的地址空间既是优点也是缺点。这样一来，一个进程不可能不小心覆盖另一个进程的虚拟内存，这就消除了许多令人迷惑的错误一一这是一个明显的优点。

另一方面，独立的地址空间使得进程共享状态信息变得更加困难。为了共享信息，它们必须使用显式的 IPC（进程间通信）机制。（参见下面的旁注。）基于进程的设计的另一个缺点是，它们往往比较慢，因为进程控制和 IPC 的开销很高。

----------------------------------------------------------------------------------------------------
进程是操作系统中独立运行的实体，具有自己的地址空间和资源。

### 优点
1. **安全性与隔离性**：每个进程有独立的地址空间，这意味着一个进程无法直接访问或修改另一个进程的内存。这种隔离性提高了系统的稳定性和安全性，因为即使一个进程崩溃或存在安全漏洞，也不会直接影响其他进程。
2. **错误隔离**：由于内存隔离，一个进程的错误（如内存泄漏、访问违规等）不会直接影响其他进程。这减少了错误传播的风险，使得系统更加健壮。

### 劣势
1. **进程间通信复杂性**：由于进程间不共享内存，它们需要使用进程间通信（IPC）机制来交换信息。IPC 机制包括管道、消息队列、共享内存、信号量等，相比于线程间通信，IPC 通常更复杂、效率较低。
2. **资源开销**：创建和管理进程比线程消耗更多资源。进程的创建涉及复制父进程的地址空间，设置新的内存管理结构，而线程则可以共享同一进程的资源。此外，进程上下文切换的开销也比线程大。
3. **性能问题**：由于独立的地址空间和资源，进程间的数据交换和通信比线程间更慢。如果应用程序需要频繁的进程间通信，那么性能可能会受到影响。

### 综合考虑
进程提供了强大的隔离和安全保障，非常适合需要隔离和安全性的场景，如操作系统中的不同服务、安全敏感的应用等。但是，当需要频繁交换数据或追求高效率时，线程或其他轻量级并发模型可能是更好的选择。
在系统设计时，需要根据具体需求和场景平衡进程的这些优势和劣势，选择最合适的并发模型。例如，在需要高度隔离和安全的系统中，使用进程可能更为合适，而在对性能要求高的计算密集型应用中，可能更倾向于使用线程或其他更轻量级的并发机制。

----------------------------------------------------------------------------------------------------
进程间通信（IPC）机制允许在不同进程间交换数据和信号。每种机制有其特定的用途和性能特征。以下是一些常用的IPC机制：

1. **管道（Pipes）**
   - **类型**：半双工（通常只能单向通信）。
   - **使用**：允许一个进程与另一个进程进行数据传输，通常用于父子进程间通信。
   - **特点**：数据以字节流形式传输，无结构化数据格式。

2. **命名管道（Named Pipes）**
   - **类型**：也称为 FIFO，是管道的扩展，允许不相关的进程间通信。
   - **使用**：在文件系统中有一个名字，允许非父子进程间通信。
   - **特点**：比普通管道灵活，可以在任何两个需要通信的进程间创建。

3. **消息队列（Message Queues）**
   - **类型**：允许进程以消息块的形式发送和接收数据。
   - **使用**：消息队列由消息组成，每个消息都有特定的类型。
   - **特点**：支持复杂的通信模式，如非阻塞调用和优先级消息。

4. **共享内存（Shared Memory）**
   - **类型**：最快的IPC方法，允许两个或多个进程共享一个内存区域。
   - **使用**：通常与其他通信机制（如信号量）结合使用，以同步对共享内存的访问。
   - **特点**：数据不需要在进程间复制，可以直接访问共享区域。

5. **信号量（Semaphores）**
   - **类型**：主要用于同步，非直接的通信方式。
   - **使用**：常用于控制对共享资源（如共享内存）的访问。
   - **特点**：可以是二进制（互斥锁）或可以取多个值，用于解决竞争条件和死锁。

6. **套接字（Sockets）**
   - **类型**：支持网络级的IPC，允许不同机器上的进程通信。
   - **使用**：基于网络模型（如 TCP/IP），适用于分布式系统。
   - **特点**：支持多种通信方式，包括流式（TCP）和数据报（UDP）。

7. **信号（Signals）**
   - **类型**：用于通知接收进程某个事件已经发生。
   - **使用**：例如，用于通知进程终止（SIGKILL）、停止（SIGSTOP）等。
   - **特点**：简单但有限的信息传递能力，主要用于进程控制。

8. **远程过程调用（RPC）**
   - **类型**：允许一个进程调用另一个进程的函数或过程。
   - **使用**：常用于客户端-服务器模型中，适用于不同机器上的进程间通信。
   - **特点**：抽象层次较高，隐藏了底层通信细节。

9. **内存映射文件（Memory-Mapped Files）**
   - **类型**：将文件或设备映射到内存。
   - **使用**：可以被多个进程访问，用于创建能被多个进程共享的文件。
   - **特点**：提供了一种文件和内存操作的结合方式，适用于大量数据的共享。

每种IPC机制都有其适用场景。选择合适的IPC机制取决于特定应用的需求，如数据传输的大小、速度、复杂性以及是否跨网络等。



> 旁注 - Unix IPC
> 在本书中，你已经遇到好几个 IPC 的例子了。第 8 章中的 waitpid 函数和信号是基本的 IPC 机制，它们允许进程发送小消息到同一主机上的其他进程。
> 第 11 章的套接字接口是 IPC 的一种重要形式，它允许不同主机上的进程交换任意的字节流。
> 然而，术语 UnixIPC 通常指的是所有允许进程和同一台主机上其他进程进行通信的技术。其中包括管道、先进先出（FIFO）、系统 V 共享内存，以及系统 V 信号量（semaphore）。这些机制超出了我们的讨论范围。Kerrisk 的著作【62】是很好的参考资料。


使用 C 语言展示进程间通信（IPC）机制的例子是一个很好的实践。下面我将分别通过简单的示例来演示管道（Pipes）、消息队列（Message Queues）和共享内存（Shared Memory）这三种常见的 IPC 机制。

### 1. 管道（Pipes）

在 Unix 和类 Unix 系统中，管道是一种典型的 IPC 机制，它允许一个进程的输出直接作为另一个进程的输入。

```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

int main() {
    int pipefd[2];
    pid_t cpid;
    char buf;

    if (pipe(pipefd) == -1) {
        perror("pipe");
        exit(EXIT_FAILURE);
    }

    cpid = fork();
    if (cpid == -1) {
        perror("fork");
        exit(EXIT_FAILURE);
    }

    if (cpid == 0) {    /* 子进程 */
        close(pipefd[1]);          // 关闭写端
        while (read(pipefd[0], &buf, 1) > 0)
            write(STDOUT_FILENO, &buf, 1);
        close(pipefd[0]);
        _exit(EXIT_SUCCESS);
    } else {            /* 父进程 */
        close(pipefd[0]);          // 关闭读端
        write(pipefd[1], "Hello, world!\n", 14);
        close(pipefd[1]);
        wait(NULL);                // 等待子进程
        exit(EXIT_SUCCESS);
    }
}
```

### 2. 消息队列（Message Queues）

消息队列允许不同的进程以消息的形式进行通信。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/ipc.h>
#include <sys/msg.h>

struct message {
    long mtype;
    char mtext[100];
};

int main() {
    struct message msg;
    int msgid;

    // 创建消息队列
    msgid = msgget(IPC_PRIVATE, 0666 | IPC_CREAT);

    msg.mtype = 1;
    sprintf(msg.mtext, "Hello, world!");

    // 发送消息
    msgsnd(msgid, &msg, sizeof(msg.mtext), 0);

    // 接收消息
    msgrcv(msgid, &msg, sizeof(msg.mtext), 1, 0);
    printf("Received: %s\n", msg.mtext);

    // 销毁消息队列
    msgctl(msgid, IPC_RMID, NULL);

    return 0;
}
```

### 3. 共享内存（Shared Memory）

共享内存是一种高效的 IPC 机制，允许两个或多个进程共享一个内存区域。

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/shm.h>
#include <sys/stat.h>

int main() {
    int segment_id;
    char* shared_memory;
    const int size = 4096;

    // 分配共享内存
    segment_id = shmget(IPC_PRIVATE, size, S_IRUSR | S_IWUSR);

    // 连接共享内存
    shared_memory = (char*) shmat(segment_id, NULL, 0);

    // 写入共享内存
    sprintf(shared_memory, "Hello, world!");

    // 读取共享内存
    printf("*%s*\n", shared_memory);

    // 断开连接
    shmdt(shared_memory);

    // 释放共享内存
    shmctl(segment_id, IPC_RMID, NULL);

    return 0;
}
```

这些示例仅作为基本的介绍，实际应用中可能需要更复杂的错误处理和同步机制。
在使用 IPC 机制时，也应考虑操作系统对这些机制的支持和限制。