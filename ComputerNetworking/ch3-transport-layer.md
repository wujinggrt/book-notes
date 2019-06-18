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


