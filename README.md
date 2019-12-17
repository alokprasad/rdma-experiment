## VM installation instructions

You will need the following tools:

* [VirtualBox](https://www.virtualbox.org/wiki/Downloads) - a desktop virtualization system.
* [Vagrant](https://www.vagrantup.com/downloads.html) - a scripted tool for managing virtual machines.

```
sudo apt-get install virtualbox
sudo wget https://releases.hashicorp.com/vagrant/2.2.2/vagrant_2.2.2_x86_64.deb
sudo dpkg -i vagrant_2.2.2_x86_64.deb
```

To create the virtual machines use:

    export VAGRANT_HOME=/some_other_location/ ( Optional)
    vagrant up

This will create a two virtual machine with the SoftRoCE RDMA driver enabled on its virtual network interface. To access
the machine, use:

    $ vagrant status
    Current machine states:

    client                    running (virtualbox)
    server                    running (virtualbox)
    
    $vagrant ssh client
    $vagrant ssh server
    

The repository files should be available inside the VM under the `/vagrant` directory.

RDMA functionality should be available on the VM's Ethernet interface, which vagrant normally configures with the IP address 10.0.2.15.


     code {white-space: pre;}  



Table of Contents
-----------------

1.  [​​background material](#background)
2.  [Part One - Performance Measurements](#part-a)
3.  [Part Two - RDMA Programming](#part-b)

​​background
============

Operating system core bypass
----------------------------

Operating systems usually take on the implementation of network layer (IP) and transport (TCP) communication protocols, and present to user programs an abstract communication interface (sockets). The operating system is the only one that has access to the network card, so its additional function is to classify the received packages and associate them with the appropriate user program.

This approach frees the programmer from the burden of implementing the various protocols, and by familiarizing themselves with the particular network card information installed on the computer. However, this approach has a number of drawbacks: sending and receiving information involves exchanging processor contacts, from the context of the user process, to the operating system, and back. Switching the connection delays the execution of the user program, during which part of the processor state is lost, which impairs system performance.

Because the operating system is the only one that communicates with the network card, information sent from the primary user program is copied to the kernel memory, and only then sent to the network. Similarly, information received from the network is placed in the core memory, and only after the operating system classifies it is copied into the user process memory. The copying of the information delays its receipt and sending, and uses a processor that could have made more useful calculations at the same time.

Newer communication cards are capable of working with user programs directly, without operating system involvement. Using a memory mapping system equivalent to the memory processor unit (MMU) of the main processor. Just as the main processor translates virtual memory addresses from the memory space of the current process to the physical memory space, so the network card also uses virtual memory and can separate different processes. As a result, the user program can give receipt or send instructions with addresses in its virtual space, directly to the card. The card makes sure the addresses are valid, and follows the instructions. The operating system is only involved when creating direct communication between the user program and the network card.

In the experiment, we will see how using a VMA library that implements communication protocols in the user environment, thus improving performance for network programs. The library implements TCP / IP protocols through direct access to the network card and transparently replaces the operating system UIs.

### Interrupts vs. Sampling (polling)

When a process waits for I / O, such as receiving information from the network or sending information, the operating system pauses the process run, allowing other processes to utilize the processor. The network card notifies the operating system of interrupt I / O events, which causes the processor to stop the current process run, process the received packets, and mark the process awaiting communication as & quot; ready to run & quot ;. Then, according to the Scheduler's scheduler decision, the connection is changed and the original process continues its run.

This interrupt mechanism allows efficient CPU utilization, by pausing I / O processes, and utilizing the processor while waiting for other calculations. However, this mechanism has overheads: the time taken from the receipt of a package to the arrival of the information to the relatively large user process. Another way to get information that integrates with the operating system bypass is to use repeated polling of I / O mode. The process requests the execution of the operation and then does not waive the CPU usage right but continues to repeatedly test whether the operation is over. In this way, the delay of receiving messages is shortened, but the processor is 100% utilized for the I / O process and sampling.

The VMA library we will use in the experiment invokes such a sampling mechanism by default.

Network Protocol Acceleration
-----------------------------

Sometimes bypassing the operating system is not enough to give the required performance. To further reduce the burden on the main processor in communication operations, transfer part of the realization of network layers and transport to the network card. The network card hardware is capable of performing the required processing at the same time as the main processor, freeing it to perform other tasks. In this experiment, we focus on the RoCE - RDMA family over Converged Ethernet protocols, using the OS core bypass in addition to accelerating the network protocol.

When sending or receiving messages, the reliable connected protocol (RC) from the RoCE family allows the user program to deliver large messages to send (up to 2GB), or large dividers to receive messages, and rely on the network card to manage the protocol severely. The network card handles packet message distribution, sending receipt certificates, arranging received messages, and resubmitting in case of packet loss. This is how the main processor is made to perform user program calculations.

Remote Direct Memory Access (RDMA) technology also allows direct access to remote memory. We refer to the end stations as the initiator and destination. The initiator wants to send information to the target, or retrieve information from it. The destination receives the request and stores the received information (if it is a send), or sends the answer (if it is a read request). Distinguish between duplex and unilateral communication, based on the degree of involvement of the main processor on the target side. In bilateral communication, the initiating party sends a message to the destination, and the destination provides a memory buffer to receive the message and receives a notification of receipt. In contrast, in unilateral communication, the initiating party is able to access remote memory directly. The initiating party sends an instruction to write or read a memory belonging to the destination, and the destination network card executes the instruction without involving the primary processor.

In the experiment, we will look at the impact of using the network protocol hardware acceleration on performance, and write a simple client-server program that uses RDMA to share remote memory.

Additional Background Material
------------------------------

Background material on RDMA technology and the operating system core can be found in the slides attached from the RDMA course at the Hebrew University.

More information about the measurement tools used in the experiment can be found at the following sources:

*   Sockperf User Guide: [sockperf man page](https://www.mankier.com/3/sockperf)
    
*   The cpufrequtils tool for CPU frequency control: [man cpufreq-set](https://manpages.debian.org/stretch/cpufrequtils/cpufreq-set.1.en.html) a>
    
*   [perftest package documentation](https://community.mellanox.com/docs/DOC-2802) that includes RDMA communication performance measurement tools
    

The RDMA programming interfaces can be read in the [RDMA Network Programming Guide](https://www.mellanox.com/related-docs/prod_software/RDMA_Aware_Programming_user_manual.pdf) (English) Of Melanox. The relevant sections are:

*   Chapter 2 RDMA-Aware Programming Overview, pages 19-24.
    
*   Chapter 4 RDMA\_CM API. The relevant lab functions are:
    
    *   `rdma_create_id`
        
    *   `rdma_create_ep`
        
    *   `rdma_listen`
        
    *   `rdma_connect`
        
    *   `rdma_get_request`
        
*   Chapter 5 RDMA Verbs API. The relevant functions are:
    
    *   `rdma_reg_msgs`
        
    *   `rdma_reg_read`
        
    *   `rdma_reg_write`
        
    *   `rdma_post_recv`
        
    *   `rdma_post_send`
        
    *   `rdma_post_read`
        
    *   `rdma_post_write`
        
    *   `rdma_get_send_completion`
        
    *   `rdma_get_recv_completion`
        

Exercise 1: Orientation
=======================

In this exercise we will introduce the system features on which we will perform the experiment.

To use the experimental computers, use the rdmauser user. To use the command line, open the console application.

On the computer a number of network cards. They can be identified using the ip link show command. Only two of them support RDMA and one will use the experiment. Find them using the command:

 ` ls / sys / class / infiniband_verbs / uverbs0 / device / net / ` 

These are two network card ports together that support RDMA. Only one of them is connected to the other server. Find the connected port by looking at the `ip link show` output - the disconnected port will appear with the `NO-CARRIER` flag.

On the trial machines, the card will appear as eth2.

Mark the network card as `$ dev` . Find the IP address of this card using the `ip addr show $ dev` command. We will mark the server IP address `$ server_ip` client `$ client_ip` .

You can set the card name and ip addresses as environment variables to make it easier to type. For example:

 ` dev = eth2
server_ip = 192.168.0.101
client_ip = 192.168.0.102 ` 

#### Question 1.1

Check the network card speed you found using the command:

 ` ethtool $ dev ` 

What is the card speed?

### Multi-core Processing

#### Question 1.2

Introduce the computer structure you are working on with the `lstopo` command. The command displays a diagram of the computer components, with processors displayed as packages, and cores as cores. How many processors are there in the machine? How many cores are there in the machine? How many wires? Is memory connected to a single connection or NUMA (Non-uniform memory architecture) architecture?

The Linux operating system attempts to balance the load on the various CPU cores, sometimes replacing the core on which a particular process is running. To prevent such swaps during each experiment, we prefer to set for the operating system that a particular kernel should be used in each experiment. To do this, we will use the `taskset` tool. The tool receives a -c parameter followed by the core number on which to run the requested program. For example, to run `ib_send_bw` on kernel # 3, we will run the command:

 ` taskset -c 3 ib_send_bw ... ` 

Note: The `taskset` command must be inserted before any command that needs to be executed on a particular kernel.

### Remote Connection

During the experiment you will work on both machines at the same time. For ease of work, connect from one machine to the other on a remote ssh connection that allows you to use the remote machine command line:

 ` ssh & lt; hostname / ip address & gt; ` 

It is recommended to open two terminal windows: one for the local machine, and one for the remote machine.

Exercise 2: TCP and bypass-kernel bypass (Kernel Bypass)
--------------------------------------------------------

In this part of the experiment we will use the TCP protocol called sockperf to measure performance. On the server side the corresponding command is:

 ` sockperf server --tcp ` 

On the client side, run sockperf to measure the data transfer rate using a sockperf throughput check. In this mode, the program tries to send the maximum number of bytes to the other side up to the size of the TCP window. Run the command:

 ` sockperf throughput --tcp -i $ server_ip --msg-size 1472 -t 10 ` 

We use a maximum message size (1472) to get a higher transfer rate and reduce the ratio of the amount of work required from the processor to the number of bytes transferred. You can control the experiment time using the `-t` parameter.

During the run, also measure processor utilization using the `top` tool or the System Monitor graphic tool. In the `top` tool, you can view the utilization of each core by itself by pressing the 1 key. The tool displays (in the third row of the output) separately the load due to user programs ( `us` ) and low priority user programs ( `ni` ), the operating system core ( `sy` \>), Wait for I / O ( `wa` ). The percentage of time the processor is waiting for inactivity is listed under `id` .

### Network Pause

To measure network latency, we will enable `sockperf` in `ping-pong` mode where each message is sent only after receiving the previous message (replace the `throughput` parameter with - `ping-pong` ). Thus measuring the total transfer time allows you to measure the network delay between the two stations. Use the more appropriate UDP protocol to measure latency by omitting the `--tcp` parameter from the `sockperf` command line (server-side and client-side), and select a small message size (16 bytes) to minimize The effect of transmission time on network delay.

### Bypass-core bypass

So far we have used the implementation of the TCP protocol within the operating system core. Now we will replace it by using the VMA library in the realization that bypasses the operating system interfaces and communicates directly with the network card. To make the program use VMA, we will temporarily set the `LD_PRELOAD = libvma.so` environment variable. In order to get direct access to the network card, we need to get a super user permission with the sudo command. In summary to run sockperf we will use the command:

 ` sudo env LD_PRELOAD = libvma.so sockperf ... ` 

Use server-side and client-side VMAs. Measure the transfer rate and network latency using the VMA library, for the same message sizes from the previous experiment (16 bytes of network latency and 1472 bytes of transfer rate). Also measure processor utilization. Compare the results and explain the results you received.

Note: VMA may print a warning that there are no large huge pages available on the machine. This warning can be ignored.

Exercise 3: use of the RoCE protocol
------------------------------------

We will now consider the use of hardware accelerated communication protocol, as a replacement for TCP. We will use a perftest application collection to measure RoCE performance.

To measure transfer rate, we use the `ib_send_bw` tool. On the server, we will run the tool without additional parameters:

    ib_send_bw

On the client (sender) side, we will run the tool with the server address as a parameter:

 ` ib_send_bw $ server_ip ` 

You can control the experiment using the `-D` parameter. The same amount of time must be passed to both the server and another client, incorrect results will be obtained.

Now run the same experiment but using a small message size (use the `-s` parameter) of 16 bytes (passed the parameter to both server and client). What is the transfer rate (number of messages per second) received, and why?

To measure network latency, we'll use the `ib_send_lat` tool, which works similarly.

Measure the transfer rate and network latency, and compare the results to results obtained using VMA. What is the source of the difference?

### Remote Memory Access

The RoCE protocol also allows access to remote memory without the processor's memory to which the memory belongs. We will demonstrate this through the `ib_write_bw` test program, where the client program executes server program memory. Its behavior is the same as other perftest programs.

Measure the data transfer rate using `ib_write_bw` using large messages (default size). During the run, measure the processor utilization on the server side and the client side. Explain the differences between the parties.

### CPU slowdown

Another way to test processor utilization is by slowing down its clock frequency. On Linux you can choose different clock frequency drivers. Selected between powersave and performance modes. In the experimental machines, the clock frequency is almost split in half in transition to powersave mode. To change all frequencies, use the command:

 ` for core in $ (seq 0 3); do sudo cpufreq-set -c $ core -g $ governor; done ` 

When replacing `$ governor` with `powersave` or `performance` as needed. Perftest tools may print a warning when the processor clock frequency does not match its maximum frequency:

 ` Conflicting CPU frequency values ​​detected: 3300.000000! = 1600.000000. CPU Frequency is not max. ` 

This warning can be ignored, or you can add the `-F` flag to cancel the warning.

Measure short messaging rate (16 bytes) when serving low and high frequency and compare results when using RDMA Write versus RDMA Send / Receive operations.

Why would we want to use a lower than normal clock frequency?

Exercise 4: Multi-Core Processing
---------------------------------

When the bottleneck in the experiment is the load on the main processor, we can achieve better performance and higher transfer rate by utilizing multiple CPU cores. Mention the [lstopo](#lstopo) results and the system structure you found. Separate cores must be distinguished from wires sharing the same processor core using Simultaneous multithreading (SMT) / Hyperthreading.

In the next experiment we will run several programs in parallel. This can be done by opening several tabs in the console ( Ctrl \+ Shift \+ T ). You can also open multiple tabs for the remote station, and use the ssh tool to open multiple parallel connections to the same station.

To see the effect of the number of processes on the transfer rate, plot a graph comparing the Y rate with the transfer rate (messages per second) versus the number of processes (X axis). Run a separate `ib_send_bw` process on each of the cores on the server (using taskset). To avoid conflicts in choosing the access port for the different processes, use the `-p` parameter to select a different port for each process. Similarly, run a separate copy of clients `ib_send_bw` with each client facing a different server. Use a small message size ( `-s` parameter) of 4 bytes. Measure the total transfer rate of all customers (messages per second), and overall processor utilization.

Now repeat the experiment but using a large message size (65536 bytes). Explain the differences between the different message sizes. What is the bottleneck in each experiment?

Part Two - RDMA Programming
===========================

In this part of the experiment we will try RDMA programming in C language using the librdmacm library. We'll change an example program that implements a simplified database with bilateral communication, and learn how to replace some of the operations with remote memory access operations to improve system performance.

The machines on which you run the experiment are shared, so before you get started, create a folder under the home folder that contains the student names in the folder name:

 ` cd ~
mkdir $ student_names
cd $ student_names ` 

Download the sample code from which we will start with the command:

 ` git clone https://github.com/haggaie/rdma-experiment.git ` 

This will copy the files to the rdma-experiment / src folder. The example program is a simplified database server. The database attaches values ​​to the keys, enabling retrieval of value given the corresponding key. The example consists of a `exp_server` server program, which receives data writing and read requests and stores them in memory, and a `exp_client` client program that allows the user to interactively store and read keys.

To build the program, run the build.sh script in the rdma-experiment / src folder. Then the prepared programs should be located in the rdma-experiment / src / build folder. To run the programs, run the server program on the server computer:

 ` exp_server [-p port_number] ` 

You can select a different port number using the `-p` parameter, for example if you share the server with multiple users.

The client program must be run with the server IP address:

 ` exp_client -s $ server_ip [-p port_number] ` 

The client program requests user input in a cyclical manner. Each time you choose the type of message - write to the database, read from it, or end the run. If the user requests to write or read, the program requests the appropriate key and value as needed, and displays the output received from the server in response.

### directory rdmacm

The program uses the rdmacm and ibverbs directories used for RDMA programming. It includes the following objects:

  

Connection ID `rdma_cm_id` (ID)

Identifies the connection with the other party on the network, enabling sending and receiving operations in front of it.

Each ID has a send queue that includes sending messages to the other party (including remote memory access operations) and a reception queue that includes memory slots for receiving messages.

Registered Memory Region (MR)

To allow the network card direct access to memory, the requested memory must be recorded in advance. After registering, an MR object is represented which represents the registered memory.

Work completion task (WC)

After successful sending or receiving, task completion data is added to an internal queue associated with ID. The program can read the data from the queue and thus identify that the operation is complete. Only after completing the action and receiving the data is the program allowed to use the memory associated with the action.

### Remote Read

To ease server load, we will modify the database read operations. Instead of sending a message to the server to read some value, we will use the RDMA Read operation to read the remote memory without server program involvement.

To send a read operation, use the `rdma_post_read` function. This takes as a parameter the identity of the connection to the server (id), the address and length of a local memory area to which the answer is copied,

To enable this, the server program already registers the database table (the variable table in the `database.c` file), and sends the MR information to the client program. The details are stored in the `db_info` variable in the `rdma_client.c` file. In order for the network card to be able to write the read result, the action also receives a local destination address. This address should also be in a registered memory area (MR). For example, you can use the existing MR `recv_mr` which includes the `recv_msg` variable.

Edit the `rdma_read` function in `rdma_client.c` , so that you submit a remote memory read request from the server. You can look at the `rpc` and `transmit_message` functions to see how the program sends messages to the server and receives replies.

### debugging

To debug, you can use the `gdb` program. To build an appropriate version of the program, you can edit `build.sh` and add the `-DCMAKE_BUILD_TYPE = Debug` parameter to the `cmake` command.

Error codes returned numerically `librdmacm` in the `errno` variable are standard POSIX codes. The list of error values ​​can be found at [errno (3)](http://man7.org/linux/man-pages/man3/errno.3.html) . The error values ​​obtained when a receive, send, or RDMA action in the `ibv_wc.status` field appears on the [Page Explanation of the `ibv_poll_cq`](http://www.rdmamojo.com/2013/02/15/ibv_poll_cq/) function on the RDMAmojo blog.

### Performance Measurement

Change the client program to perform a large number of read operations (100,000), without user interaction (printouts must be canceled using the -q parameter), and a large number of update operations. Measure the time it takes:

1.  Update actions
2.  Reading operations (sending and receiving messages using `rpc ()` )
3.  Read actions (using RDMA)

Use the [`clock_gettime` function with the `CLOCK_MONOTONIC` parameter to Read the current time before and after the measurement.](https://linux.die.net/man/3/clock_gettime)

[

Appendix A - Experiment Preparation
===================================

A virtual machine can be used to simulate an RDMA-capable network card. Download the machine settings using:

](https://linux.die.net/man/3/clock_gettime)

[git clone](https://linux.die.net/man/3/clock_gettime) [https://github.com/haggaie/rdma-experiment](https://github.com/haggaie/rdma-experiment)

\>

[VirtualBox](https://www.virtualbox.org/wiki/Downloads) and [Vagrant](https://www.vagrantup.com/downloads.html must be installed ) (version 2.0.3 or higher).

Operate the machines using vagrant up. For the first time, downloading and installing may take time.

You now have two virtual machines with client and server names connected to a virtual network. You can connect to each of them via vagrant ssh client or vagrant ssh server.

When working with SoftRoCE (at home preparation work), RoCE should be enabled after each machine restart using the command:

 ` sudo rxe_cfg start `
