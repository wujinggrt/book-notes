1. [Intro](##3-4-Principles-of-Reliable-Data-Transfer)

## 3.1.2

IP
1. best-effort delivery service
2. unreliable service

The most fundamental responsibility of **UDP** and **TCP** is to *extend* IP's delivery service between two end system to a delivery service **between two processes** running on the end systems.

Extending host-to-host delivery to process-to-process delivery is called **transport-layer multiplexing** and **demultiplexing**.

Like IP, UDP does not guarantee that data sent by one process will arrive intact.

TCP: **reliable data transfer**. 
1. Converts IP's unreliable to reliable. 
2. Ensuring reliable **sending** and correct **order**.
3. Provide **congestion control**.(拥塞控制)

TCP regulates the rate at the sending sides of TCP connections can send traffic into the network.

## 3.2 Multiplexing and Demultiplexing

Multiplexing and demultiplexing refers to **extending** the host-to-host delivery **service** provided by the network layer (前面是 multiplexing, host 之间。 后面是 demultiplexing, process 之间) to a process-to-process delivery service for app runing on hosts.

A process can have one or more **sockets**.(Doors the data passes from the process to the network still the network to the process)

The transport layer in the receiving host **does not** acctually deliver data directly to a process, but instead to an **intermediary socket**.

Transport-layer segment has a set of fields to direcs itself to appropriate socket.

* **demultiplexing**: delivering the data in a transport-layer segment to the **correct socket**.
* **multiplexing**: **gathering** data chunks at the source host from different sockets, **encapsulating** each with **header information** (used by demultiplexing) to create segment and passing to the network layer. to create segment and passing to the network layer.

In short, demultiplexing works for arriving segments, multiplexing for sending.

Multiplexing requires:
1. sockets have unqiue identifiers,
2. each segment have **special fields** that indicate where the receiving socket

Special fields:**source port number field** and **destination port number field**.(Each port number is a 16-bit number)

Each socket in the host could be **assigned** a port number, and when a segment arrives at the host, the transport layer **examines** the dest port and directs the segment to it.

#### Connectionless Multiplexing and Demultiplexing

UDP: identified by a **two-tuple** consisting dest IP addr and port number. 

As a consequence, two **different source** (both IP addr and port number) may target to a dest, then the two segments will be directed to the **same** dest process via the same dest **socket**.

What is the purpose of the source port number (as in wrapped segments head)? For source port number serves as a "return address" when dest want to send a segment **back** to A in new segment.(traceroute)

Summary, UDP socket only **identified** by two-tuple, which is dest IP and dest port. The src IP and port serves as a "return address".

### Connection Multiplexing and Demultiplexing

Diff to UDP, TCP socket is **identified** by **four-tuple** (src IP and port, dest IP and port), the host use four values to direct (demultiplex) the segment to the appropriate socket.

In constrast with UDP, two arriving TCP segments with **different src IP addr** or **port numbers** will be directed to two **differen sockets**. (characterastic of four-tuple identifier) Newly created socket is identified by thes four values.

Ensure host by IP, process by port number. Socket is the interface only.

## 3.3 Connectionless Transport: UDP

Aside from the multiplexing/demultiplexing function and some light error checking, UDP add nothing to IP. If a app chooses UDP, it is almost talking with IP.

UDP **takes** message from the application process, **attaches** src and dest port**, **adds** two other small fields, and **passes** the resulting segment to the network layer.

UDP has **no handshaking** between sending and receiving transport-layer entities before sending a segment. For this reason, UDP is said to be **connnectionless**.

e.g. DNS is an application-layer protocol that typically uses UDP.

Better suited for UDP for the following reasons:
0. **Finer(精确) application-level control over what data is sent, and when.** UDP immediately pass the segment to network layer after package rather than more links. (real-time, minimum sending rate, tolerate some data loss)
0. **No connection establishment.**
0. **No connection state.**
0. **Small packet header overhead.** TCP segment: 20 bytes, whereas UDP has only 8.

UDP is possible to have reliable data transfer. e.g. QUIC protocol.

### 3.3.1 UDP Segment Structure

The UDP **header** has only **four fields**, **each** consisting of **two bytes**, that is, **4 * 2 * 8** bits in total.The **data field** of the UDP segment is the application data. (Not in header)

Header:
0. The **src/dst port** #, 2 * 2 * 8 bits.
0. The **length field** specifies the **number of bytes** in the UDP **segment** (**header plus data**). This field is needed since the size of data field may **differ** to next segment.
0. The **checksum** is used by receiving host to check whether **errors** have been introduced into the segment.

### 3.3.2 Chceksum (question: what eactly the 3 16-bit words is?)

At the sender side, sum of all the 3 16-bit words in the segment, and make 1s complement (取反) as checksum.

At the receiver side, all four 16-bit words are added, **including** the checksum. If no errors are introduced into the packet, it turns out 1111 1111 1111 1111. If one of the bits is a 0, that errors have been introduced.

Why UDP provides a checksum in first place? The reason is that there is no guarantee that all th lins beteween two side provide error checking.

Because IP is supposed to run over just about any layer-2 protocol, it is useful for transport layer to provide error checking.

Although UDP provides errors checking, it does not do anything to **recover** from an error.

## 3.4 Principles of Reliable Data Transfer

No transferred data bits are corrupted or lost.

The task is difficult by the fact that the layer **below** the reliable data transfre protocol may be **unreliable**. Such as best effort transfer IP. TCP over the IP.

View the lower layer simply as unreliable point-to-point channel.

Assumption: packets possibly being lost.

The sending side invoke *rdt_send()*, it will pass the data. (Application layer need to pass data, it will be called, implemented in transport layer.)

While the receiving side invokes *rdt_rcv()*. (Transport layer call to receive.) When *rdt* protocol wants to deliver data to the upper layer, it will do so by calling *deliver_data()*. (Pass to application layer call)

The term "packet" is more appropriate here rather than "segment".

Both sending and receiving side of *rdt* call to the other side by a call to *udt_send()*.

### 3.4.1 Buildling a RDT Protocol

#### Perfectly Reliable Channel: rdt1.0

Simplest case: the underlying channel is completely reliable. We'll call it **rdt1.0** protocol.

The **finite-state machine (FSM)** definition for *rdt1.0* sender and receiver. 
