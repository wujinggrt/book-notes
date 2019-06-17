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

