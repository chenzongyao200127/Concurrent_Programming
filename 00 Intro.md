# Concurrent Programming
# 第 12 章：并发编程

正如我们在第 8 章学到的，如果逻辑控制流在时间上重叠，那么它们就是并发的（`concurrent`）。
这种常见的现象称为并发（concurrency），出现在计算机系统的许多不同层面上。
硬件异常处理程序、进程和 Linux 信号处理程序都是大家很熟悉的例子。

到目前为止，我们主要将并发看做是一种操作系统内核用来运行多个应用程序的机制。但是，并发不仅仅局限于内核。它也可以在应用程序中扮演重要角色。例如，我们已经看到 Linux 信号处理程序如何允许应用响应异步事件，例如用户键入 Ctrl+C，或者程序访问虚拟内存的一个未定义的区域。
应用级并发在其他情况下也是很有用的：

 - `访问慢速 I/O 设备`
 当一个应用正在等待来自慢速 I/O 设备（例如磁盘）的数据到达时，内核会运行其他进程，使 CPU 保持繁忙。
 每个应用都可以按照类似的方式，通过交替执行 I/O 请求和其他有用的工作来利用并发。

 - `与人交互`
 和计算机交互的人要求计算机有同时执行多个任务的能力。例如，他们在打印一个文档时，可能想要调整一个窗口的大小。
 现代视窗系统利用并发来提供这种能力。每次用户请求某种操作（比如通过单击鼠标）时，一个独立的并发逻辑流被创建来执行这个操作。

 - `通过推迟工作以降低延迟`
 有时，应用程序能够通过推迟其他操作和并发地执行它们，利用并发来降低某些操作的延迟。
 比如，一个动态内存分配器可以通过推迟合并，把它放到一个运行在较低优先级上的并发 “合并” 流中，在有空闲的 CPU 周期时充分利用这些空闲周期，从而降低单个 free 操作的延迟。

 - `服务多个网络客户端`
 我们在第 11 章中学习的迭代网络服务器是不现实的，因为它们一次只能为一个客户端提供服务。
 因此，一个慢速的客户端可能会导致服务器拒绝为所有其他客户端服务。对于一个真正的服务器来说，可能期望它每秒为成百上千的客户端提供服务，由于一个慢速客户端导致拒绝为其他客户端服务，这是不能接受的。一个更好的方法是创建一个并发服务器，它为每个客户端创建一个单独的逻辑流。
 这就允许服务器同时为多个客户端服务，并且也避免了慢速客户端独占服务器。

 - `在多核机器上进行并行计算`。
 许多现代系统都配备多核处理器，多核处理器中包含有多个 CPU。被划分成并发流的应用程序通常在多核机器上比在单处理器机器上运行得快，因为这些流会并行执行，而不是交错执行。

使用应用级并发的应用程序称为并发程序（concurrent program）。`现代操作系统`提供了三种基本的构造并发程序的方法：

1. *进程*
用这种方法，每个逻辑控制流都是一个进程，由内核来调度和维护。因为进程有独立的虚拟地址空间，想要和其他流通信，控制流必须使用某种显式的进程间通信（interprocesscommunication，IPC）机制。

2. *I/O 多路复用*。
在这种形式的并发编程中，应用程序在一个进程的上下文中显式地调度它们自己的逻辑流。逻辑流被模型化为状态机，数据到达文件描述符后，主程序显式地从一个状态转换到另一个状态。因为程序是一个单独的进程，所以所有的流都共享同一个地址空间。

----------------------------------------------------------------------------------------------------
这句话描述的是 **I/O多路复用**（Input/Output Multiplexing）在并发编程中的应用。要理解这个概念，我们可以分解这段描述的关键点：

1. **并发编程**: 并发编程是指允许多个任务在重叠的时间段内进行，而不是顺序执行。这通常用于提高程序的效率和响应性。

2. **在一个进程的上下文中显式地调度它们自己的逻辑流**: 这意味着在单个进程内部，程序会显式管理多个逻辑任务（或“逻辑流”）。这与多线程编程不同，在多线程编程中，操作系统通常会处理线程之间的调度。

3. **逻辑流被模型化为状态机**: 每个逻辑流被视为一个状态机，这意味着它们可以根据输入或其他条件从一个状态转换到另一个状态。状态机是一个模型，其中系统可以处于有限个状态之一。

4. **数据到达文件描述符后，主程序显式地从一个状态转换到另一个状态**: 当I/O操作（如读取网络数据或文件）完成并且数据可用时，程序将基于这些事件改变其状态。文件描述符是一个指向打开文件的引用，包括网络连接。

5. **因为程序是一个单独的进程，所以所有的流都共享同一个地址空间**: 在这个模型中，由于所有的逻辑流都在同一个进程中运行，它们共享相同的内存地址空间。这与进程间通信相对，后者涉及多个独立的内存空间。

总结来说，I/O多路复用是一种并发编程模式，它允许单个进程高效地处理多个I/O操作。
*这是通过在数据可用时显式地切换逻辑流的状态来实现的*，而不是依赖于多个线程或进程。
这种方法通常用于网络编程和事件驱动的应用程序，因为它可以提高效率，减少资源消耗，同时简化程序的复杂度。
----------------------------------------------------------------------------------------------------

3. *线程*。
线程是运行在一个单一进程上下文中的逻辑流，由内核进行调度。
你可以把线程看成是其他两种方式的混合体，像进程流一样由内核进行调度，而像 I/O多路复用流一样共享同一个虚拟地址空间。

本章研究这三种不同的并发编程技术。为了使我们的讨论比较具体，我们始终以同一个应用为例——11.4.9 节中的迭代 echo 服务器的并发版本。


----------------------------------------------------------------------------------------------------
提供一个使用Python中的`selectors`模块实现的简单的I/O多路复用服务器的例子。这个例子展示了一个非阻塞的Echo服务器，它读取客户端发送的数据并原样返回。

```python
import selectors
import socket

def accept_wrapper(sock):
    conn, addr = sock.accept()  # Should be ready to read
    print('Accepted connection from', addr)
    conn.setblocking(False)
    data = types.SimpleNamespace(addr=addr, inb=b'', outb=b'')
    events = selectors.EVENT_READ | selectors.EVENT_WRITE
    sel.register(conn, events, data=data)

def service_connection(key, mask):
    sock = key.fileobj
    data = key.data
    if mask & selectors.EVENT_READ:
        recv_data = sock.recv(1024)  # Should be ready to read
        if recv_data:
            data.outb += recv_data
        else:
            print('Closing connection to', data.addr)
            sel.unregister(sock)
            sock.close()
    if mask & selectors.EVENT_WRITE:
        if data.outb:
            print('Echoing', repr(data.outb), 'to', data.addr)
            sent = sock.recv(1024)  # Should be ready to write
            data.outb = data.outb[sent:]

# Create a selector object
sel = selectors.DefaultSelector()

# Set up the server socket
host, port = '0.0.0.0', 65432
lsock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
lsock.bind((host, port))
lsock.listen()
print('Listening on', (host, port))
lsock.setblocking(False)

# Register the socket for monitoring with sel.select()
sel.register(lsock, selectors.EVENT_READ, data=None)

try:
    while True:
        events = sel.select(timeout=None)
        for key, mask in events:
            if key.data is None:
                accept_wrapper(key.fileobj)
            else:
                service_connection(key, mask)
except KeyboardInterrupt:
    print("Caught keyboard interrupt, exiting")
finally:
    sel.close()
```

这个例子中：

1. 创建了一个非阻塞的服务器套接字，并在指定端口上监听。
2. 使用`selectors.DefaultSelector()`创建一个选择器，用于监控套接字。
3. 在无限循环中，`sel.select()`调用等待I/O事件。它返回一组`(key, mask)`元组，其中每个元组代表一个就绪的I/O事件。
4. 对于每个就绪事件，如果是新的连接，则调用`accept_wrapper`来接受连接。如果是已经存在的连接上的数据，则调用`service_connection`来处理数据。

这个服务器接受客户端连接，读取客户端发送的数据，并将其原样发送回去（"echo"）。
需要注意的是，这个例子是一个基础的示例，实际应用中的服务器通常更复杂，包含更多的错误处理和功能。
----------------------------------------------------------------------------------------------------

