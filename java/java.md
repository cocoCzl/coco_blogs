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
