# Benchmarks

该项目是一个主要针对 [Aeron](https://github.com/aeron-io/aeron) 项目的各种基准测试的集合。
这些基准测试可以分为两大类：


This project is a collection of the various benchmarks primarily targeting the [Aeron](https://github.com/aeron-io/aeron) project.
The benchmarks can be divided into two major categories:




- [Messaging (remote) benchmarks](#remote-benchmarks-multiple-machines).

    远程基准测试的核心由 [`LoadTestRig`](https://github.com/aeron-io/benchmarks/blob/master/benchmarks-api/src/main/java/uk/co/real_logic/benchmarks/remote/LoadTestRig.java) 类实现，它是一个基准测试工具，用于向远程节点发送消息并在接收响应时进行计时。
    在测试运行期间，`LoadTestRig` 会以指定的固定速率、负载大小和突发大小发送消息。
    最终，它会生成整个测试运行的延迟直方图。



    The core of the remote benchmarks is implemented by the [`LoadTestRig`](https://github.com/aeron-io/benchmarks/blob/master/benchmarks-api/src/main/java/uk/co/real_logic/benchmarks/remote/LoadTestRig.java)
    class which is a benchmarking harness that is sending messages to the remote node(s) and timing the responses as
    they are received. During a test run the `LoadTestRig` sends messages at the specified fixed rate with the specified
    payload size and the burst size. In the end it produces a latency histogram for an entire test run.

    `LoadTestRig` 依赖于 [`MessageTransceiver`](https://github.com/aeron-io/benchmarks/blob/master/benchmarks-api/src/main/java/uk/co/real_logic/benchmarks/remote/MessageTransceiver.java) 抽象类的实现，该类负责向远程节点发送消息并从远程节点接收消息。

    *注意：这些基准测试是用 Java 编写的，但只要有相应的 Java 客户端，它们也可以针对其他语言编写的系统进行测试。*



    The `LoadTestRig` relies on the implementation of the [`MessageTransceiver`](https://github.com/aeron-io/benchmarks/blob/master/benchmarks-api/src/main/java/uk/co/real_logic/benchmarks/remote/MessageTransceiver.java)
    abstract class which is responsible for sending and receiving messages to/from the remote node.

    *NB: These benchmarks are written in Java, but they can target systems in other languages provided there is a
    Java client for it.*



    `LoadTestRig` 依赖于 [`MessageTransceiver`](https://github.com/aeron-io/benchmarks/blob/master/benchmarks-api/src/main/java/uk/co/real_logic/benchmarks/remote/MessageTransceiver.java) 抽象类的实现，该类负责向远程节点发送消息并从远程节点接收消息。

    *注意：这些基准测试是用 Java 编写的，但只要有相应的 Java 客户端，它们也可以针对其他语言编写的系统进行测试。*



- [Other benchmarks](#other-benchmarks-single-machine).

   A collection of the benchmarks that run on a single machine (e.g. Agrona ring buffer, Aeron IPC, Aeorn C++
   benchmarks, JDK queues etc.).


    一组在单台机器上运行的基准测试（例如：Agrona 环形缓冲区、Aeron IPC、Aeron C++ 基准测试、JDK 队列等）。


## Systems under test

This section lists the systems under test which implement the remote benchmarks and the corresponding test scenarios.

本节列出了实现远程基准测试的被测系统及其对应的测试场景。


### Aeron

For [Aeron](https://aeron.io/) the following test scenarios were implemented:

1. Echo benchmark.

   An Aeron Transport benchmark which consist of a client process that sends messages over UDP using an exclusive 
   publication and zero-copy API (i.e. [`tryClaim`](https://github.com/aeron-io/aeron/blob/3f6c5e15bd30a83d46978bf39eff8d927f30fe5a/aeron-client/src/main/java/io/aeron/Publication.java#L556)).
   And the server process which echoes the received messages back using the same API.

    一个 Aeron 传输基准测试，由以下部分组成：

   * **客户端进程**：使用独占发布和零拷贝 API（即 [`tryClaim`](https://github.com/aeron-io/aeron/blob/3f6c5e15bd30a83d46978bf39eff8d927f30fe5a/aeron-client/src/main/java/io/aeron/Publication.java#L556)）通过 UDP 发送消息。
   * **服务端进程**：使用相同的 API 将接收到的消息回传给客户端。




2. Live replay from a remote Archive.**从远程 Archive 进行实时回放**

    The client publishes messages to the server using publication over UDP. The server pipes those messages into a local 
    IPC publication which records them into an Archive. Finally, the client subscribes to the replay from that Archive
    over UDP and receives persisted messages.

    客户端通过 UDP 发布，将消息发送到服务器。服务器将这些消息传入本地 IPC 发布中，并将其记录到 Archive 中。最后，客户端通过 UDP 订阅该 Archive 的回放，从而接收已持久化的消息。




3. Live recording to a local Archive.**实时录制到本地 Archive**

    The client publishes messages over UDP to the server. It also has a recording running on that publication using
    local Archive. The server simply pipes message back. Finally, the client performs a controlled poll on the 
    subscription from the server limited by the "recording progress" which it gets via the recording events.

    The biggest difference between scenario 2 and this scenario is that there is no replay of recorded messages and
    hence no reading from disc while still allowing consumption of only those messages that were successfully persisted.


    客户端通过 UDP 将消息发布到服务器。同时，它使用本地 Archive 对该发布进行录制。服务器则简单地将消息回传。最后，客户端在从服务器的订阅上执行受控轮询，并通过录制事件获取“录制进度”来限制轮询范围。

    此场景与场景 2 最大的区别在于：该场景没有对已录制消息进行回放，因此不需要从磁盘读取数据，但仍然只允许消费那些已成功持久化的消息。



4. **集群基准测试** Cluster benchmark.

   The client sends messages to the Aeron Cluster over UDP. The Cluster sequences the messages into a log, reaches the
   consensus on the received messages, processes them and then replies to the client over UDP.

    客户端通过 UDP 将消息发送到 Aeron 集群。集群将消息排序到日志中，对接收到的消息达成共识，处理这些消息，然后通过 UDP 回复客户端。



5. **Aeron Echo MDC 基准测试** Aeron Echo MDC benchmark. 

   An extension to Aeron Echo benchmark which uses an MDC (or a multicast) channel to send the same data to multiple
   receivers. Only one receiver at a time will respond to a given incoming message ensuring that the number of replies
   matches the number of messages sent.

    这是对 Aeron Echo 基准测试的扩展，使用 MDC（多播）通道将相同的数据发送给多个接收者。
    对于每条接收到的消息，一次只有一个接收者进行响应，确保回复的数量与发送的消息数量相匹配。



6. **Aeron Archive Replay MDC 基准测试** Aeron Archive Replay MDC benchmark.


这是一个支持多重回放的 Aeron Archive 基准测试。该基准测试至少包含三个节点：

* 发送数据的客户端节点
* 将数据流录制到磁盘的 Archive 节点
* 请求从 Archive 回放录制内容的回放节点

与 Aeron Echo MDC 基准测试类似，一次只有一个回放节点会向客户端节点发送响应消息，从而确保发送的消息数量与回放的数量一致。

   Aeron Archive benchmark that multiple replays. The benchmark consists of at least three nodes: 
   - the client node sending the data
   - the Archive node recording the data stream to disc
   - the replay nodes requesting replay of the recording from the Archive
   
   Similar to the Aeron Echo MDC benchmark only one replay node at a time will send a response message back to the
   client node thus ensuring that the number of messages sent and the number of replays match.


请参阅 `scripts/aeron` 目录中的文档以获取更多信息。

Please the documentation in the ``scripts/aeron`` directory for more information.

### gRPC

对于 [gRPC](https://grpc.io/)，只有一个回显基准测试，并且只有单一实现：

* **流式客户端**：客户端使用流式 API 来发送和接收消息。

请参阅 `scripts/grpc` 目录中的文档以获取更多信息。


For [gRPC](https://grpc.io/) there is only echo benchmark with a single implementation:
- Streaming client - client uses streaming API to send and receive messages.

Please read the documentation in the ``scripts/grpc`` directory for more information.

### Kafka

Unlike the gRPC that simply echoes messages the [Kafka](https://kafka.apache.org/) will persist them so the benchmark is
similar to the Aeron's replay from a remote Archive.

Please read the documentation in the `scripts/kafka` directory for more information.

与仅简单回显消息的 gRPC 不同，[Kafka](https://kafka.apache.org/) 会将消息持久化，因此该基准测试类似于 Aeron 从远程 Archive 的回放。

请参阅 `scripts/kafka` 目录中的文档以获取更多信息。


## Remote benchmarks (multiple machines)

`scripts` 目录包含用于运行**远程基准测试**的脚本，即涉及多台机器的基准测试，其中一台作为**客户端**（基准测试工具），其余的是**服务器节点**。

`remote.io.aeron.benchmarks.LoadTestRig` 类实现了基准测试工具，而 `remote.io.aeron.benchmarks.Configuration` 类为基准测试工具提供配置。

在执行基准测试之前，必须先进行构建。可以在项目根目录下运行以下命令：

```bash
./gradlew clean deployTar
```

完成后，将生成一个 `build/distributions/benchmarks.tar` 文件，该文件需要部署到远程机器上。



The `scripts` directory contains scripts to run the _remote benchmarks_, i.e. the benchmarks that involve multiple
machines where one is the _client_ (the benchmarking harness) and the rest are the _server nodes_.

The `remote.io.aeron.benchmarks.LoadTestRig` class implements the benchmarking harness. Whereas the
`remote.io.aeron.benchmarks.Configuration` class provides the configuration for the benchmarking harness.

Before the benchmarks can be executed they have to be built. This can be done by running the following command in the
root directory of this project:
```bash
./gradlew clean deployTar
```
Once complete it will create a `build/distributions/benchmarks.tar` file that should be deployed to the remote machines.

### Running benchmarks via SSH (i.e. automated way)

The easiest way to run the benchmarks is by using the `remote_*` wrapper scripts which invoke scripts remotely using
the SSH protocol. When the script finishes its execution it will download an archive with the results (histograms).

The following steps are required to run the benchmarks:
1. Build the tar file (see above).
2. Copy tar file to the destination machines and unpack it, i.e. `tar xf benchmarks.tar -C <destination_dir>`.
3. On the local machine create a wrapper script that sets all the necessary configuration parameters for the target
benchmark. See example below.
4. Run the wrapper script from step 3.
5. Once the execution is finished an archive file with the results will be downloaded to the local machine. By default,
it will be placed under the `scripts` directory in the project folder.

Here is an example of a wrapper script for the Aeron echo benchmarks.
_NB: All the values in angle brackets (`<...>`) will have to be replaced with the actual values._


运行基准测试最简单的方法是使用 `remote_*` 包装脚本，这些脚本通过 SSH 协议在远程机器上调用基准测试脚本。当脚本执行完成后，它会下载包含结果（直方图）的压缩包。

运行基准测试需要以下步骤：

1. 构建 tar 文件（见上文）。
2. 将 tar 文件复制到目标机器并解压，例如：

   ```bash
   tar xf benchmarks.tar -C <destination_dir>
   ```
3. 在本地机器上创建一个包装脚本，用于设置目标基准测试所需的所有配置参数（见下方示例）。
4. 运行第 3 步中创建的包装脚本。
5. 执行完成后，包含结果的压缩文件会被下载到本地机器，默认会放置在项目文件夹下的 `scripts` 目录中。

以下是一个 Aeron Echo 基准测试的包装脚本示例：
*注意：所有尖括号 (`<...>`) 中的值都需要替换为实际值。*



```bash
# SSH connection properties
export SSH_CLIENT_USER=<SSH client machine user>
export SSH_CLIENT_KEY_FILE=<private SSH key to connect to the client machine>
export SSH_CLIENT_NODE=<IP of the client machine>
export SSH_SERVER_USER=<SSH server machine user>
export SSH_SERVER_KEY_FILE=<private SSH key to connect to the server machine>
export SSH_SERVER_NODE=<IP of the server machine>

# Set of required configuration options
export CLIENT_BENCHMARKS_PATH=<directory containing the unpacked benchmarks.tar>
export CLIENT_JAVA_HOME=<path to JAVA_HOME (JDK 17+)>
export CLIENT_DRIVER_CONDUCTOR_CPU_CORE=<CPU core to pin the 'conductor' thread>
export CLIENT_DRIVER_SENDER_CPU_CORE=<CPU core to pin the 'sender' thread>
export CLIENT_DRIVER_RECEIVER_CPU_CORE=<CPU core to pin the 'receiver' thread>
export CLIENT_LOAD_TEST_RIG_MAIN_CPU_CORE=<CPU core to pin 'load-test-rig' thread>
export CLIENT_NON_ISOLATED_CPU_CORES=<a set of non-isolated CPU cores to run the auxilary/JVM client threads on>
export CLIENT_CPU_NODE=<CPU node (socket) to run the client processes on (both MD and the test rig)>
export CLIENT_AERON_DPDK_GATEWAY_IPV4_ADDRESS=
export CLIENT_AERON_DPDK_LOCAL_IPV4_ADDRESS=
export CLIENT_SOURCE_CHANNEL="aeron:udp?endpoint=<SOURCE_IP>:13100|interface=<SOURCE_IP>/24"
export CLIENT_DESTINATION_CHANNEL="aeron:udp?endpoint=<DESTINATION_IP>:13000|interface=<DESTINATION_IP>/24"
export SERVER_BENCHMARKS_PATH=<directory containing the unpacked benchmarks.tar>
export SERVER_JAVA_HOME=<path to JAVA_HOME (JDK 17+)>
export SERVER_DRIVER_CONDUCTOR_CPU_CORE=<CPU core to pin the 'conductor' thread>
export SERVER_DRIVER_SENDER_CPU_CORE=<CPU core to pin the 'sender' thread>
export SERVER_DRIVER_RECEIVER_CPU_CORE=<CPU core to pin the 'receiver' thread>
export SERVER_ECHO_CPU_CORE=<CPU core to pin 'echo' thread>
export SERVER_NON_ISOLATED_CPU_CORES=<a set of non-isolated CPU cores to run the auxilary/JVM server threads on>
export SERVER_CPU_NODE=<CPU node (socket) to run the server processes on (both MD and the echo node)>
export SERVER_AERON_DPDK_GATEWAY_IPV4_ADDRESS=
export SERVER_AERON_DPDK_LOCAL_IPV4_ADDRESS=
export SERVER_SOURCE_CHANNEL="${CLIENT_SOURCE_CHANNEL}"
export SERVER_DESTINATION_CHANNEL="${CLIENT_DESTINATION_CHANNEL}"

# (Optional) Overrides for the runner configuration options 
#export MESSAGE_LENGTH="288" # defaults to "32,288,1344"
#export MESSAGE_RATE="100K"  # defaults to "1M,500K,100K"

# Invoke the actual script and optionally configure specific parameters
"aeron/remote-echo-benchmarks" --client-drivers "java" --server-drivers "java" --mtu 8K --context "my-test"

## DPDK-specific configuration
#export CLIENT_AERON_DPDK_GATEWAY_IPV4_ADDRESS=<SOURCE_DPDK_GATEWAY_ADDRESS>
#export SERVER_AERON_DPDK_GATEWAY_IPV4_ADDRESS=<DESTINATION_DPDK_GATEWAY_ADDRESS>
#export CLIENT_AERON_DPDK_LOCAL_IPV4_ADDRESS=<SOURCE_DPDK_ADDRESS>
#export SERVER_AERON_DPDK_LOCAL_IPV4_ADDRESS=<DESTINATION_DPDK_ADDRESS>
#export CLIENT_SOURCE_CHANNEL="aeron:udp?endpoint=${CLIENT_AERON_DPDK_LOCAL_IPV4_ADDRESS}:13100"
#export CLIENT_DESTINATION_CHANNEL="aeron:udp?endpoint=${SERVER_AERON_DPDK_LOCAL_IPV4_ADDRESS}:13000"
#export SERVER_SOURCE_CHANNEL="${CLIENT_SOURCE_CHANNEL}"
#export SERVER_DESTINATION_CHANNEL="${CLIENT_DESTINATION_CHANNEL}"
#"aeron/remote-echo-benchmarks" --client-drivers "c-dpdk" --server-drivers "c-dpdk" --mtu 8K --context "my-test"
```

### 手动运行基准测试（单次执行） Running benchmarks manually (single shot execution)

The following steps are required to run the benchmarks:
1. Build the tar file (see above).
2. Copy tar file to the destination machines and unpack it, i.e. `tar xf benchmarks.tar -C <destination_dir>`.
3. Follow the documentation for a particular benchmark to know which scripts to run and in which order.
4. Run the `benchmark-runner` script specifying the _benchmark client script_ to execute.

Here is an example of running the Aeron echo benchmark using the embedded Java MediaDriver on two nodes:
server (`192.168.0.20`) and client (`192.168.0.10`).

运行基准测试需要以下步骤：

1. 构建 tar 文件（见上文）。
2. 将 tar 文件复制到目标机器并解压，例如：

   ```bash
   tar xf benchmarks.tar -C <destination_dir>
   ```
3. 根据特定基准测试的文档，了解需要运行哪些脚本以及运行顺序。
4. 运行 `benchmark-runner` 脚本，并指定要执行的**基准测试客户端脚本**。

以下是在两台节点上使用内嵌 Java MediaDriver 运行 Aeron Echo 基准测试的示例：

* 服务器节点：`192.168.0.20`
* 客户端节点：`192.168.0.10`



```bash
server:~/benchmarks/scripts$ JVM_OPTS="\
-Dio.aeron.benchmarks.aeron.embedded.media.driver=true \
-Dio.aeron.benchmarks.aeron.source.channel=aeron:udp?endpoint=192.168.0.10:13000 \
-Dio.aeron.benchmarks.aeron.destination.channel=aeron:udp?endpoint=192.168.0.20:13001" aeron/echo-server

client:~/benchmarks/scripts$ JVM_OPTS="\
-Dio.aeron.benchmarks.aeron.embedded.media.driver=true \
-Dio.aeron.benchmarks.aeron.source.channel=aeron:udp?endpoint=192.168.0.10:13000 \
-Dio.aeron.benchmarks.aeron.destination.channel=aeron:udp?endpoint=192.168.0.20:13001" \
./benchmark-runner --output-file "aeron-echo-test" --messages "100K" --message-length "288" --iterations 60 "aeron/echo-client"
```

***注意**：在单次运行结束后，服务器端进程（例如 `aeron/echo-server`）将会退出。也就是说，如果需要再次手动运行（使用不同参数等），必须重新启动服务器进程。另一种方法是[通过 SSH 运行基准测试](#running-benchmarks-via-ssh-ie-automated-way)，即采用自动化方式运行。*


_**Note**: At the end of a single run the server-side process (e.g. `aeron/echo-server`) will exit, i.e. in order to do
another manual run (with different parameters etc.) one has to start the server process again. Alternative is to run the
benchmarks [via the SSH](#running-benchmarks-via-ssh-ie-automated-way)._

### 聚合结果 Aggregating the results

To aggregate the results of the multiple runs into a single file use the `aggregate-results` script.

For example if the ``results`` directory contains the following files:


要将多次运行的结果聚合到一个文件中，可以使用 `aggregate-results` 脚本。

例如，如果 `results` 目录包含以下文件：



```bash
results
├── echo-test_rate=1000_batch=1_length=32-0.hdr
├── echo-test_rate=1000_batch=1_length=32-1.hdr
├── echo-test_rate=1000_batch=1_length=32-2.hdr
├── echo-test_rate=1000_batch=1_length=32-3.hdr
└── echo-test_rate=1000_batch=1_length=32-4.hdr
```   

Running:
```bash
./aggregate-results results
```

Will produce the following result:

```bash
results
├── echo-test_rate=1000_batch=1_length=32-0.hdr
├── echo-test_rate=1000_batch=1_length=32-1.hdr
├── echo-test_rate=1000_batch=1_length=32-2.hdr
├── echo-test_rate=1000_batch=1_length=32-3.hdr
├── echo-test_rate=1000_batch=1_length=32-4.hdr
├── echo-test_rate=1000_batch=1_length=32-combined.hdr
└── echo-test_rate=1000_batch=1_length=32-report.hgrm
```

其中，`echo-test_rate=1000_batch=1_length=32-combined.hdr` 是五次运行的聚合直方图，
而 `echo-test_rate=1000_batch=1_length=32-report.hgrm` 是该聚合直方图的导出文件，可以使用 [http://hdrhistogram.github.io/HdrHistogram/plotFiles.html](http://hdrhistogram.github.io/HdrHistogram/plotFiles.html) 进行绘图。


where `echo-test_rate=1000_batch=1_length=32-combined.hdr` is the
aggregated histogram of five runs and the `echo-test_rate=1000_batch=1_length=32-report.hgrm` is an export of the
aggregated histogram that can be plotted using http://hdrhistogram.github.io/HdrHistogram/plotFiles.html.

### 绘制结果图表 Plotting the results

Aggregated results can be plotted using the `results-plotter.py` script which uses [hdr-plot](https://github.com/BrunoBonacci/hdr-plot) in order to produce latency plots of the histograms (the library needs to be installed in order to use the script).

可以使用 `results-plotter.py` 脚本绘制聚合结果图表。该脚本使用了 [hdr-plot](https://github.com/BrunoBonacci/hdr-plot) 库来生成直方图的延迟图表（需要先安装该库才能使用此脚本）。


Running

```bash
./results-plotter.py results
```

该脚本默认会生成按测试场景分组的直方图图表。也可以生成不同聚合方式的图表，并对目录中的直方图应用过滤条件进行绘制。

运行 `./results-plotter.py`（不带参数）即可查看该绘图脚本的功能概览。


will produce plots in which the histograms are grouped by test scenario by default. It is possible to produce graphs with a different kind of aggregation and to apply filters on the histograms to plot within a directory. Run `./results-plotter.py` (without arguments) in order to get an overview of the capabilities of the plotting script.

## 在 Kubernetes 上运行 Running on Kubernetes

You will need the following Docker containers built & injected into a repository that you can use.

The tests currently support Aeron Echo testing with either Java or C-DPDK media drivers.

你需要构建以下 Docker 容器并将其推送到你可用的镜像仓库。

当前测试支持使用 Java 或 C-DPDK 媒体驱动的 Aeron Echo 测试。


### 组件与容器

**基准测试：**

这是本仓库中的代码，必须构建为 Docker 容器。

**可选：Aeron DPDK 媒体驱动**

高级功能。

如果在测试配置中需要或启用，请参见 [https://github.com/aeron-io/premium-extensions/，或联系](https://github.com/aeron-io/premium-extensions/，或联系) [Adaptive](https://weareadaptive.com/) 的技术支持。

该驱动预计以容器形式存在于一个可访问的镜像仓库中。

**可选：Aeron C 媒体驱动**

即将支持。


Components & Containers

**Benchmarks:**

This is the code in *this* repository. It must be built as a Docker container.


**Optional: Aeron DPDK Media driver**

Premium feature.

If required/activated in your test configuration - see https://github.com/aeron-io/premium-extensions/ or ask your support contact at [Adaptive](https://weareadaptive.com/)

This is expected to reside in a container called in an accessible repository.

**Optional: Aeron C Media driver:**

Support coming soon

### 运行测试 Running the tests 

1. 构建基准测试容器并推送到你的 Kubernetes 节点可拉取的镜像仓库：
   Build the benchmarks container and push it to a repo that your K8s nodes can pull from:

    ```
    docker build -t <your_repo>:aeron-benchmarks .
    docker push <your_repo>:aeron-benchmarks
    ```

2. 使用你的测试环境配置更新以下文件 — 不打算测试的场景配置可以跳过。
    
   Update the following files with configuration from your test environment - you can skip scenario config you don't plan to test


   * `scripts/k8s/base/settings.yml`
   * `scripts/k8s/base/aeron-echo-dpdk/settings.yml`
   * `scripts/k8s/base/aeron-echo-java/settings.yml`

3. 确保你的测试环境处于当前激活的 `kubecontext`。

4. 如果你打算运行 DPDK 测试，确保你有启用 DPDK 的 Pod 或主机。该设置不在本说明范围内，请参见 [https://github.com/AdaptiveConsulting/k8s-dpdk-mgr](https://github.com/AdaptiveConsulting/k8s-dpdk-mgr) 获取设置示例。

5. 确保你有权限向 Kubernetes 的某个命名空间写入，默认情况下工具会使用 `default` 命名空间。



3. Make sure your test environment is the active `kubecontext`

4. If you are attempting to run DPDK tests, make sure you have a DPDK enabled Pod/Host. Setting this up is outside the scope of this documentation, please see https://github.com/AdaptiveConsulting/k8s-dpdk-mgr for an example of how to do this.

5. Ensure you are permissioned to write to a K8s namespace, by default the tooling will use the `default` namespace.

6. Run:
   
   ```
   ./scripts/k8s-remote-testing.sh (-t aeron-echo-java | aeron-echo-dpdk ) ( -n my_namespace )
   ```

##  其他基准测试（单机） Other benchmarks (single machine)

一组延迟基准测试，用于测试线程或进程间（IPC）通过 FIFO 数据结构和消息系统的往返时间（RTT）。

Set of latency benchmarks testing round trip time (RTT) between threads or processes (IPC) via FIFO data structures and messaging systems.


### Java Benchmarks

要运行 Java 基准测试，请在项目根目录执行 Gradle 脚本：

```bash
$ ./gradlew runJavaIpcBenchmarks
```

或者仅运行 Aeron 基准测试：

```bash
$ ./gradlew runAeronJavaIpcBenchmarks
```


To run the Java benchmarks execute the Gradle script in the base directory.

    $ ./gradlew runJavaIpcBenchmarks

or just the Aeron benchmarks

    $ ./gradlew runAeronJavaIpcBenchmarks

### C++ Benchmarks

要生成基准测试，请在项目根目录执行 `cppbuild` 脚本：

```bash
$ cppbuild/cppbuild
```

要运行基准测试，请执行各个基准测试二进制文件：

```bash
$ cppbuild/Release/binaries/baseline
$ cppbuild/Release/binaries/aeronExclusiveIpcBenchmark
$ cppbuild/Release/binaries/aeronIpcBenchmark
$ cppbuild/Release/binaries/aeronExclusiveIpcNanomark
$ cppbuild/Release/binaries/aeronIpcNanomark
```

**注意**：在 MacOS 上，需要为 Aeron 驱动共享库设置 `DYLD_LIBRARY_PATH`。例如：

```bash
$ env DYLD_LIBRARY_PATH=cppbuild/Release/aeron-prefix/src/aeron-build/lib cppbuild/Release/binaries/aeronIpcBenchmark
```

名称中带有 **Benchmark** 的二进制文件使用 Google Benchmark，仅显示平均时间；
名称中带有 **Nanomark** 的二进制文件使用 Nanomark（包含在源码中），会显示完整的直方图。

如果想指定 Aeron 的特定标签（版本），在调用 `cppbuild` 脚本时传入 `--aeron-git-tag` 参数。
例如：

```bash
cppbuild/cppbuild --aeron-git-tag="1.42.0"
```

将使用 Aeron `1.42.0` 版本。



To generate the benchmarks, execute the `cppbuild` script from the base directory.

    $ cppbuild/cppbuild

To run the benchmarks, execute the individual benchmarks.

    $ cppbuild/Release/binaries/baseline
    $ cppbuild/Release/binaries/aeronExclusiveIpcBenchmark
    $ cppbuild/Release/binaries/aeronIpcBenchmark
    $ cppbuild/Release/binaries/aeronExclusiveIpcNanomark
    $ cppbuild/Release/binaries/aeronIpcNanomark

**Note**: On MacOS, it will be necessary to set `DYLD_LIBRARY_PATH` for the Aeron
driver shared library. For example:

    $ env DYLD_LIBRARY_PATH=cppbuild/Release/aeron-prefix/src/aeron-build/lib cppbuild/Release/binaries/aeronIpcBenchmark

The binaries with __Benchmark__ in the name use Google Benchmark and only displays average times.

While the binaries with __Nanomark__ in the name use Nanomark (included in the source) and displays full histograms.

To pick a specific tag for Aeron, specify `--aeron-git-tag` parameter when invoking `cppbuild` script.
For example:
```bash
cppbuild/cppbuild --aeron-git-tag="1.42.0"
```
will use Aeron `1.42.0` release.

License (See LICENSE file for full license)
-------------------------------------------
Copyright 2015-2025 Real Logic Limited.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
