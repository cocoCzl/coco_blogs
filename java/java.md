## Process死锁问题
jdk中有这么一段描述
```
By default, the created subprocess does not have its own terminal or console. All its standard I/ O (i. e. stdin, stdout, stderr) operations will be redirected to the parent process,
where they can be accessed via the streams obtained using the methods getOutputStream(), getInputStream(), and getErrorStream().
The parent process uses these streams to feed input to and get output from the subprocess.
Because some native platforms only provide limited buffer size for standard input and output streams, 
failure to promptly write the input stream or read the output stream of the subprocess may cause the subprocess to block, or even deadlock.
```
意思是默认情况下:  
1. 创建的子进程没有自己的终端或控制台，它的标准输入输出（stdin, stdout, stderr）会被重定向到父进程，
父进程可以通过 getOutputStream()、getInputStream() 和 getErrorStream() 获取这些流来与子进程交互。  
2. 由于一些平台对标准流的缓冲区大小有限，如果不及时读取输出或写入输入流，可能会导致子进程阻塞，甚至死锁。

**原因**  
这个问题的根源在于操作系统对进程间通信和标准输入输出流的缓冲区的管理方式。大多数操作系统为每个进程分配标准输入输出流（stdin、stdout、stderr）的缓冲区，  
以便在进程间交换数据时进行缓冲处理。缓冲区的大小是有限的，这意味着如果进程没有及时读取或写入数据，这些缓冲区可能会填满，从而阻止进程继续执行。  
缓冲区的有限大小：
1. 缓冲区的有限大小，当你启动一个子进程并与其交互时，子进程的标准输出（stdout）或标准错误输出（stderr）会被写入一个缓冲区。
如果父进程不及时读取这些输出数据，缓冲区就会被填满。由于缓冲区已满，子进程就无法继续写入数据，导致它阻塞（挂起，等待父进程读取输出）。
2. 类似地，如果父进程通过标准输入（stdin）向子进程传送数据，但子进程没有及时从标准输入流中读取数据，父进程的输入缓冲区也可能填满，从而阻塞父进程。

阻塞和死锁：
* 阻塞：子进程或父进程会因为缓冲区已满，导致它们无法继续执行。比如，子进程试图向 stdout 输出数据，但因为父进程没有读取它，缓冲区已满，子进程就会被阻塞，等待父进程读取。
* 死锁：如果父进程和子进程互相等待对方，且都在等待彼此读取对方的流，就会发生死锁。例如，父进程等待子进程的输出，而子进程又等待父进程向其提供输入，导致两者都无法继续执行。
### 示例
假设我们有一个父进程与子进程进行交互，父进程通过ProcessBuilder启动子进程:  
```
ProcessBuilder builder = new ProcessBuilder("some-command");
Process process = builder.start();

// 从标准输出流读取
InputStream inputStream = process.getInputStream();
BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream));

// 将输出流中的数据打印出来
while ((line = reader.readLine()) != null) {
    System.out.println(line);
}
```
* 如果父进程没有及时读取inputStream中的数据，子进程的输出缓冲区会填满，进而阻塞子进程。
* 反之，如果父进程向子进程的输入流（stdin）写入数据，但子进程没有及时读取数据，父进程也可能会被阻塞。
### 如何避免阻塞和死锁
1. 及时读取输出流：
使用独立的线程或非阻塞方式及时读取子进程的标准输出流和错误输出流。例如，可以创建一个线程专门读取子进程的输出流，这样父进程就不会因为输出缓冲区满而被阻塞。
```
new Thread(() -> {
    try (BufferedReader reader = new BufferedReader(new InputStreamReader(process.getInputStream()))) {
        String line;
        while ((line = reader.readLine()) != null) {
            System.out.println(line);
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
}).start();
```
2. 使用线程读取标准输入：
如果父进程需要向子进程写入数据，也应当确保在另一个线程中及时写入标准输入流，防止父进程被阻塞。
3. 使用 ProcessBuilder 设置流的重定向：
如果你不需要捕获子进程的输出，可以通过 ProcessBuilder 设置流的重定向，直接将标准输出和错误输出重定向到文件或 /dev/null，避免缓冲区满的问题。
```
builder.redirectOutput(new File("output.txt"));
builder.redirectError(new File("error.txt"));
```
4. 注意缓冲区大小：
了解平台上缓冲区的大小限制，并通过适当的设计减少大规模的数据交互，例如限制单次读写的数据量，或分批次进行数据传输。
### 总结
当父进程没有及时读取子进程的输出流，或者没有及时向子进程提供输入时，操作系统的缓冲区会被填满，从而导致进程的阻塞，甚至死锁。为了避免这些问题，我们可以使用线程异步处理流数据、使用适当的缓冲区管理，并确保进程间的输入输出能顺畅进行。

## Kubernetes CPU 限额的默认测量周期和YGC的问题
在 Kubernetes（k8s）中，CPU 限额的默认测量周期（即指标收集和限制生效的时间窗口）是 100 毫秒（ms）。

### 问题：
容器设置的是4C，然后宿主机器是128C。因为某些原因导致<font style="color:rgba(0, 0, 0, 0.85);">Kubernetes 的 CPU 限制无法被 JVM 感知（例如使用了较旧的 JDK 版本，或未启用相关特性），JVM 将根据 </font>**宿主机的总 CPU 核心数**<font style="color:rgba(0, 0, 0, 0.85);"> 来计算 GC 线程数量，而非容器的 CPU 配额。</font>

### <font style="color:rgb(0, 0, 0);">Kubernetes CPU 限额详细说明：</font>
1. **CPU 限额的核心机制**  
   Kubernetes 对容器的 CPU 资源限制（`resources.limits.cpu`）是通过 CGroup（控制组）实现的。CGroup 的 CPU 限制会跟踪容器在单位时间内的 CPU 使用量，并确保不超过设定的限额。
2. **默认周期的作用**  
   100 毫秒是 CGroup v1 中默认的 CPU 时间片周期（`cpu.cfs_period_us`，单位为微秒，100ms = 100000us）。在每个周期内，容器最多可使用的 CPU 时间由 `cpu.cfs_quota_us` 定义（例如，1 核 CPU 的限额对应 `quota = 100000us`，即填满整个周期）。
3. **周期与限额的关系**
    - 若容器的 CPU 限额为 `0.5` 核（500m），则 `quota = 50000us`，意味着每个 100ms 周期内最多使用 50ms 的 CPU 时间。
    - 周期长度（100ms）是固定的，限额通过调整每个周期内的可用时间（quota）来实现。
4. **是否可配置？**  
   是的，周期（`cpu.cfs_period_us`）和限额（`cpu.cfs_quota_us`）均可通过 CGroup 配置调整，但 Kubernetes 并未直接暴露这两个参数的 API，默认使用 100ms 周期。实际使用中，用户通常只需通过 `limits.cpu` 设定限额，无需修改周期。

总结：k8s 中 CPU 限额的默认测量周期为 **100 毫秒**，这是由 CGroup 机制决定的基础时间窗口。

### GC线程数<font style="color:rgb(0, 0, 0);">详细说明：</font>
#### 1. 并行垃圾回收器(ParallelGC):
ParallelGCThreads:这个参数指定了用于垃圾回收的并行线程数。

计算公式:

当CPU核心数小于等于8时，ParallelGCThreads 等于CPU核心数。

当CPU核心数大于8时，ParallelGCThreads 的计算公式为：ParallelGCThreads = 8 + ((CPU核心数- 8) * 5/8)。

#### 2. 并发垃圾回收器(CMS):
ParallelGCThreads:

用于STW (Stop-The-World) 暂停时，进行并行垃圾回收的线程数。

ConcGCThreads:

用于并发标记和清理阶段，与应用程序交替执行的线程数。

#### 计算公式:
ParallelGCThreads 的计算方式与并行垃圾回收器类似，根据CPU核心数进行调整.

ConcGCThreads 的默认值通常是`(ParallelGCThreads + 3) / 4` 。

#### 示例:
假设一个系统有16个CPU核心：

对于并行GC，ParallelGCThreads 大约会是：8 + ((16 - 8) * 5/8) = 13。

对于CMS，ParallelGCThreads 也会是13，而 ConcGCThreads 大约会是：(13 + 3) / 4 = 4。

总结:

GC线程数的设置对垃圾回收的性能有重要影响。合理的线程数设置可以减少垃圾回收的停顿时间，提高应用程序的响应速度。在实际应用中，可以根据具体的硬件配置和应用场景，通过调整 ParallelGCThreads 和 ConcGCThreads 等参数来优化GC性能。

### 解决方法
```java
-XX:ParallelGCThreads=4  # 根据容器配额手动计算
```
