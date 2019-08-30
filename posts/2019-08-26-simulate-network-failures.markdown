---
title: Simulating network failures at syscall level
---

### Intro

A short story of when checking one small property leads you deep down into the rabbit hole. The goal was to check high-level fault-tolerance property and I ended up changing CPU register to get the exact failure I wanted and exactly when I wanted.

For those familiar with the topic of syscall interception it could still be entertaining, as it introduces a new tool for the job.

### Kafka producer idempotence

All of this started, when I wanted to setup Kafka client with a new feature introduced in librdkafka 1.0: idempotent producer. Here is a definition of this feature, as described in [Confluent blog](https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/):

>An idempotent operation is one which can be performed many times without causing a different effect than only being performed once. The producer send operation is now idempotent. In the event of an error that causes a producer retry, the same message — which is still sent by the producer multiple times — will only be written to the Kafka log on the broker once.

Setting up Kafka producer to use idempotence is pretty straightforward, just a [config entry](https://github.com/edenhill/librdkafka/blob/v1.0.0/CONFIGURATION.md) *enable.idempotence=true* needs to be provided. Those of us, who doubt that it works properly ("is it really all that needs to be set?" "did I include a correct dependency version?"), would probably want some additional verification.

Under normal operation, turning this feature on does not change much. The only observable effect is some additional bits in the serialized message:

>How does this feature work? Under the covers it works in a way similar to TCP; each batch of messages sent to Kafka will contain a **sequence number** which the broker will use to dedupe any duplicate send.

At this point, we could just sniff at network traffic and check that those sequence numbers are actually set. Based on that it could be deduced that producer idempotence works. Still, the dedupe mechanism has not been observed. Ideally, we would have a test with two scenarios: with and without idempotence enabled, where we could observe duplicated messages in one scenario and no duplicates in the other.

How can we check deduplication (ideally without changing our application's code)?

Here are the options I have found so far:

* Killing TCP connections (or the broker itself) - a large number of messages is sent. During sending, TCP connections are killed (eg using `tcpkill`) or the broker itself is killed. See [blog post](https://jack-vanlightly.com/blog/2018/10/25/testing-producer-deduplication-in-apache-kafka-and-apache-pulsar).
* Socket network emulation - either by using callbacks (requires modifying our app code) or by preloading a library for wrapping network calls, as can be done in [tests for librdkafka](https://github.com/edenhill/librdkafka/blob/73295a702cd1c85c11749ade500d713db7099cca/tests/sockem.c#L200).
* Hatrace - modifying those system calls, which we want to fail.

I chose Hatrace to verify Kafka producer idempotence.

### What is Hatrace?

Short answer:

>scriptable `strace`

At least that is, what its [GitHub page](https://github.com/nh2/hatrace) says. For those of you, who wonder what is `strace`, it's a Linux utility tool for tracing system calls. Going futher, a system call (in short syscall) is a way for your application to use the interface of the Linux kernel to perform actions such as: reading a file from hard disk, creating network sockets, sending data through sockets, process creation and management, memory management, in short any IO operation your application uses.

Hatrace is:

* an executable to trace those syscalls (similarly to `strace`)
* a library for processing syscalls in a programmatic way

So, any IO operation made by your application can be traced and optionally modified using Hatrace library. You don't even need to have the source code:

<img src="/images/fault-inject.png" />

By changing syscalls we can inject a failure "from below". From the application perspective there won't be any difference between real and injected system call failure. There is no need to change the code and it could be used for any application, any language or virtual machine.

Hatrace is a relatively young (still a "work in progress") project led by [Niklas](https://github.com/nh2) and [Kirill](https://github.com/qrilka). It is written in Haskell, which you always wanted to learn ;) and that's a good thing, cause contributing to Hatrace is way easier than I initially thought.

It should be noted that `strace` also enables modification of syscalls using `-e inject`, but its abilities are limited by its executable syntax. For example, we can change the result of syscalls, but there is no programmatic way to control which syscalls should be changed.

The advantage of Hatrace over the other approaches described above is that in its current form it enables precise introduction of failure into syscalls. There are other ways to achieve this precision in overriding syscall results: [1](http://samanbarghi.com/blog/2014/09/05/how-to-wrap-a-system-call-libc-function-in-linux/), [2](https://github.com/pmem/syscall_intercept), [3](https://blog.trailofbits.com/2019/01/17/how-to-write-a-rootkit-without-really-trying/). Those mainly involve using the `LD_PRELOAD` trick and wrapping `glibc` calls. This approach has its own limitations. For example, [Go uses syscalls](https://stackoverflow.com/questions/55735864/how-does-golang-make-system-calls) directly on Linux, without depending on `glibc`. The other drawback is a necessity to use a lower-level language, and even though [all programmers must learn c](https://www.deconstructconf.com/2017/joe-damato-all-programmers-must-learn-c-and-assembly), I still prefer to write my test cases in a higher level language.

### Hatrace in practice

This is the code used to execute test cases in two scenarios:

```Haskell
spec :: Spec
spec = around_ withKafka $ do
  describe "kafka producer without idempotence" $ do
    it "sends duplicate messages on timeouts" $ do
      let enableIdempotence = False
      msgs <- runProducerTestCase enableIdempotence
      msgs `shouldSatisfy` (\case
                               Right messages -> length messages == 7
                               Left _ -> False
                           )

  describe "kafka producer with idempotence enabled" $ do
    it "sends duplicates, but they are discarded on the broker" $ do
      let enableIdempotence = True
      msgs <- runProducerTestCase enableIdempotence
      msgs `shouldSatisfy` (\case
                               Right messages -> length messages == 5
                               Left _ -> False
                           )
```

Those tests use Kafka and ZooKeeper, which are deployed to Docker in `withKafka`. Here is the actual execution of each test case (some hacks will follow):

```Haskell
runProducerTestCase :: Bool -> IO (Either KafkaError [Maybe B.ByteString])
runProducerTestCase enableIdempotence = do
  execPath <- takeDirectory <$> getExecutablePath
  let cmd = execPath </> "../idempotent-producer-exe/idempotent-producer-exe"
  argv <- procToArgv cmd [brokerAddress, kafkaTopic, show messageCount, show enableIdempotence]
  counter <- newIORef (0 :: Int)
  void $ flip runReaderT counter $
    sourceTraceForkExecvFullPathWithSink argv $
      syscallExitDetailsOnlyConduit .|
      changeSendmsgSyscallResult .|
      CL.sinkNull
  msgs <- consumeMessages
  printMessages msgs
  pure msgs
```

So, what happens in here is that in lines:

```Haskell
  let cmd = execPath </> "../idempotent-producer-exe/idempotent-producer-exe"
  argv <- procToArgv cmd [brokerAddress, kafkaTopic, show messageCount, show enableIdempotence]
```

A command to be executed is build. Firstly, a path to executable is constructed relative to current test location - this is a hack to run main program directly from tests. An why do we need to start a separate process? Internally, Hatrace uses similar mechanism as `strace`, which is a `ptrace(2)` system call as described in its man page:

>The `ptrace()` system call provides a means by which **one process** (the "tracer") may observe and control the execution of **another process** (the "tracee"), and examine and change the tracee's memory and registers.  It is primarily used to implement breakpoint debugging and system call tracing.

Next, we provide the parameters for our Kafka producer and then we run it in:

```Haskell
  void $ flip runReaderT counter $
    sourceTraceForkExecvFullPathWithSink argv $
      syscallExitDetailsOnlyConduit .|
      changeSendmsgSyscallResult .|
      CL.sinkNull
```

This part above runs the executable and traces syscall events. With a Conduit pipeline we filter only exit-from-syscall events, as `ptrace` notifies us on syscall enter and exit. The "meat" of bug injection happens in `changeSendmsgSyscallResult`:

```Haskell
changeSendmsgSyscallResult :: (MonadIO m, MonadReader (IORef Int) m)
                           => ConduitT SyscallEvent SyscallEvent m ()
changeSendmsgSyscallResult = awaitForever $ \(pid, exitOrErrno) -> do
  yield (pid, exitOrErrno)
  case exitOrErrno of
    Left{} -> pure () -- ignore erroneous syscalls
    Right exit -> case exit of
      DetailedSyscallExit_sendmsg SyscallExitDetails_sendmsg
          { bytesSent } -> do
            when (("msg2" `B.isInfixOf` bytesSent)) $ do
              counterRef <- ask
              counter <- liftIO $ readIORef counterRef
              when (counter < 2) $ liftIO $ do
                let timedOutErrno = foreignErrnoToERRNO eTIMEDOUT
                setExitedSyscallResult pid (Left timedOutErrno)
                modifyIORef' counterRef (+1)
      _ -> pure ()
```

The above code processes `SyscallEvent`s and when `sendmsg(2)` syscall is encountered, then the bytes sent are investigated. Certain type of messages, those which contain "msg2", will be failed with a timeout error, but only a limited number of them.

To determine the actual value that needs to be set, let's first have a look at the `setExitedSyscallResult` function:

```Haskell
setExitedSyscallResult :: CPid -> Either ERRNO Word64 -> IO ()
setExitedSyscallResult cpid errorOrRetValue = do
  let newRetValue =
        case errorOrRetValue of
          Right num -> num
          Left (ERRNO errno) -> fromIntegral (-errno)
  regs <- annotatePtrace "setExitedSyscallResult: ptrace_getregs" $ ptrace_getregs cpid
  let newRegs =
        case regs of
          X86 r -> X86 r { eax = fromIntegral newRetValue }
          X86_64 r -> X86_64 r { rax = newRetValue }
  annotatePtrace "setExitedSyscallResult: ptrace_setregs" $ ptrace_setregs cpid newRegs
```

When we set the syscall result to some errno value, then we take a negative of it and set in `rax` register in case of 64-bit architecture. To understand what is going on here, it's good to see how syscalls are executed from assembly.

Firstly, an application needs to set general purpose registers. On x86_64 this will be: syscall number in `rax` register, syscall arguments in `rdi`, `rsi`, `rdx`, `r10`, `r8`, `r9` registers. After that, a `syscall` machine instruction is executed to pass control to the kernel. Upon completion, the `rax` register is filled with a return value. Here is an example taken from some [StackOverflow answer](https://stackoverflow.com/a/20326189/1738581):

```Assembly
        .global _start

        .text
_start:
        # write(1, message, 13)
        mov     $1, %rax                # system call 1 is write
        mov     $1, %rdi                # file handle 1 is stdout
        mov     $message, %rsi          # address of string to output
        mov     $13, %rdx               # number of bytes
        syscall

        # exit(0)
        mov     $60, %rax               # system call 60 is exit
        xor     %rdi, %rdi              # return code 0
        syscall
message:
        .ascii  "Hello, World\n"
```

Fortunately, all of this "register" work is usually done by syscall wrappers provided by the `glibc` library, so we can simply call a C API function instead of assembly. There is one caveat, though, as described in man for syscalls (`man 2 intro`):

>On error, most system calls return a negative error number (..). The C library wrapper hides this detail from the caller: when a system call returns a negative value, the wrapper copies the absolute value into the errno variable, and returns -1 as the return value of the wrapper.

This description is in fact, inaccurate, as not all negative return values are treated as an error. Some could be also valid, succesful results. For more details see: [Linux System Calls, Error Numbers, and In-Band Signaling](https://nullprogram.com/blog/2016/09/23/). In short, this is what happens to translate negative return value to errno in [libc library](https://git.musl-libc.org/cgit/musl/tree/src/internal/syscall_ret.c?h=v1.1.15):

```C
long __syscall_ret(unsigned long r)
{
    if (r > -4096UL) {
        errno = -r;
        return -1;
    }
    return r;
}
```

In other words, we could also set the error value like that:

```Haskell
setExitedSyscallResult pid (Right $ fromIntegral (-110 :: CULong))
```

Where does the 110 value come from? I got it from `errno.h` (in my case it was in `/usr/include/asm-generic/errno.h`):

<div class="sourceCode"><pre class="sourceCode"><code class="sourceCode">
...
#define ESHUTDOWN       108     /* Cannot send after transport endpoint shutdown */
#define ETOOMANYREFS    109     /* Too many references: cannot splice */
<b>#define ETIMEDOUT       110     /* Connection timed out */</b>
#define ECONNREFUSED    111     /* Connection refused */
#define EHOSTDOWN       112     /* Host is down */
#define EHOSTUNREACH    113     /* No route to host */
...
</code></pre></div>

When the above error value is used, running the test scenarios will produce following output:

```
$ stack build --test
idempotent-producer-0.1.0.0: test (suite: idempotent-producer-test)


IdempotentProducer
  kafka producer without idempotence
zookeeper started
kafka broker started
%3|1567199588.190|FAIL|rdkafka#producer-1| [thrd:127.0.0.1:9092/1]: 127.0.0.1:9092/1: Send failed: Connection timed out (after 179ms in state UP)
%3|1567199588.190|ERROR|rdkafka#producer-1| [thrd:127.0.0.1:9092/1]: 127.0.0.1:9092/1: Send failed: Connection timed out (after 179ms in state UP)
%3|1567199588.291|FAIL|rdkafka#producer-1| [thrd:127.0.0.1:9092/1]: 127.0.0.1:9092/1: Send failed: Connection timed out (after 87ms in state UP)
%3|1567199588.291|ERROR|rdkafka#producer-1| [thrd:127.0.0.1:9092/1]: 127.0.0.1:9092/1: Send failed: Connection timed out (after 87ms in state UP)
messages were sent to broker
consumed messages: ["msg1","msg2","msg2","msg2","msg3","msg4","msg5"]
kafka broker stopped
zookeeper stopped
    sends duplicate messages on timeouts
  kafka producer with idempotence enabled
zookeeper started
kafka broker started
%3|1567199633.280|FAIL|rdkafka#producer-1| [thrd:127.0.0.1:9092/1]: 127.0.0.1:9092/1: Send failed: Connection timed out (after 80ms in state UP)
%3|1567199633.281|ERROR|rdkafka#producer-1| [thrd:127.0.0.1:9092/1]: 127.0.0.1:9092/1: Send failed: Connection timed out (after 80ms in state UP)
%3|1567199633.385|FAIL|rdkafka#producer-1| [thrd:127.0.0.1:9092/1]: 127.0.0.1:9092/1: Send failed: Connection timed out (after 99ms in state UP)
%3|1567199633.386|ERROR|rdkafka#producer-1| [thrd:127.0.0.1:9092/1]: 127.0.0.1:9092/1: Send failed: Connection timed out (after 99ms in state UP)
messages were sent to broker
consumed messages: ["msg1","msg2","msg3","msg4","msg5"]
kafka broker stopped
zookeeper stopped
    sends duplicates, but they are discarded on the broker

Finished in 86.6692 seconds
2 examples, 0 failures

idempotent-producer-0.1.0.0: Test suite idempotent-producer-test passed
```

The `ETIMEDOUT` error was chosen as an example, from `man 7 tcp` page. The [error handling logic](https://github.com/edenhill/librdkafka/blob/8695b9d63ac0fe1b891b511d5b36302ffc84d4e2/src/rdkafka_broker.c#L794) in `librdkafka` handles most of the errors similarly: tearing down the connection and reconnecting.

Could this situation also happen in real life? According to *Unix Network Programming, Volume 1, Chapter 7* yes, when the `SO_KEEPALIVE` socket option is set:

Scenario   |Peer process crashes   |Peer host crashes   |Peer host is unreachable
-- | -- | -- | -- 
Our TCP is actively sending data |Peer TCP sends a FIN, which we can detect immediately using select for readability. If TCP sends another segment, peer TCP responds with an RST. If the application attempts to write to the socket after TCP has received an RST, our socket implementation sends us SIGPIPE. |Our TCP will time out and our socket's pending error will be set to **ETIMEDOUT** |Our TCP will time out and our socket's pending error will be set to EHOSTUNREACH.


`SO_KEEPALIVE` is configurable in `librdkafka` [config](https://github.com/edenhill/librdkafka/blob/v1.0.0/CONFIGURATION.md) with `socket.keepalive.enable`.

We could also try running the example with another error set. For example, using the already mentioned `EHOSTUNREACH`, which will produce following errors in the log:

```
%3|1567202450.663|ERROR|rdkafka#producer-1| [thrd:127.0.0.1:9092/1]: 127.0.0.1:9092/1: Send failed: No route to host (after 89ms in state UP)
```

## Summary

This was a simple example of using Hatrace library to test the behavior of our application under failure scenario, which is hard to reproduce otherwise. It is applicable for all applications using system calls, so the application itself can be written in any language, while our tests would need to be written in Haskell.

The main drawback of this approach is its performance. Attaching to another process with `ptrace` and reading its memory makes calling syscalls much slower, even though the overhead of Haskell's FFI is [quite low](https://github.com/dyu/ffi-overhead).

For more possible use cases of Hatrace have a look at: [https://github.com/nh2/hatrace#use-cases](https://github.com/nh2/hatrace#use-cases)
