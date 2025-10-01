# Understanding Proxy, Reverse Proxy, NGINX, Networking

### What is a Proxy or Forward Proxy?
> It is like a middleman which sits between client and server. It is a server which passes requests to the actual backend servers on behalf of clients.

> For example:
>> When you request https://example.com (just an example), instead of sending the request directly to the website, your browser sends it to the proxy server.
>> The proxy server forwards the request to the website on your behalf, then passes the response to you. 
>> **An example of Proxy is VPN.**

#### Key Points:
1. The destination server sees the request as coming from the proxy, not from your computer.
2. Your identity (client IP) is hidden.

![Forward Proxy](forward_proxy.png)

### What is a Reverse Proxy?
> A reverse proxy sits infront of servers, acting as a single entry point for clients. NGINX is a great example of reverse proxy.
* When a client requests, for example: https://myapp.com, it hits the reverse proxy first.
* The reverse proxy decides which backend server should handle the request.
* The client never sees the actual servers - it only knows about the reverse proxy.

#### Key Points:
1. The client doesn't know how many servers exist or their addresses.
2. The reverse proxy can:
    * Load balance requests across multiple servers.
    * Cache responses to improve performance.
    * Add security by hiding backend server's IP and handling SSL/TLS.
    * Terminate SSL (do encryption/decryption at the proxy instead of backend).

![alt text](reverse_proxy.png)

**So in simple terms, we can say that, the Forward Proxy hides client identity and Reverse Proxy hides Server/s identity.**

### NGINX
Nginx is a reverse proxy, written in C language. It is designed to handle many thousands of connections with very little CPU and memory. It does this by using event-driven, asynchronous, non-blocking architecture.
> NGINX has three main processes:
* Master Process.
* Worker Processes.
* Helper Processes (Optional, like cache manager).

#### Master Process
* When NGINX starts, the master process is created first.
* Responsibilities:
    * Reads and validates the configuration file (nginx.conf)
    * Opens listening sockets (e.g., port 80, 443)
    * Spawns worker processes.
    * Handles signals (reload, stop, restart) and passes them to workers. 
    * Note: The master itself does not handle the traffic. It's mainly a manager.

#### Worker Processes
* These are the real workhorses.
* Each worker:
    * Accepts new connectios.
    * Reads/Writes requests/responses.
    * Handles proxying, caching, SSL/TLS, compression, etc.
* How workers are created:
    * We configure in nginx.conf, how many workers you want (worker_processes)
    * **Usually set to the number of CPU cores.**
    * Example: **worker_processes auto;**
* How workers handle traffic:
    * Each worker is single threaded but uses an event loop.
    * Instead of creating one thread per connection, it uses epoll(Linux) or kqueue(BSD/macOS) -- system calls that efficiently tells NGINX which sockets have new data.
    * This way, each worker can handle thousands of connections simultaneously without blocking.
* Inside a worker: The worker follows a cycle:
    * Accept: Wait for new connections on listening socket.
    * Read: Parse request line, headers, body.
    * Process: Apply rules (routing, reverse proxy, cache, filters)
    * Write (send back response)
    * Close or reuse.

#### Connection handling in detail
When a client connects:
* The Kernel notifies NGINX (via epoll/kqueue).
* One worker accepts the connection.
* The worker reads data into memory buffers (non-blocking).
* Modules are applied (HTTP parser, SSL handler, proxy module, gzip, etc).
* Response is written back in chunks (non-blocking).
* The worker goes back to the event loop to check other sockets.

#### Why is this design efficient?
* Event-driven: Workers never make CPU waiting; they respond only when the OS says a socket is ready.
* Single-threaded workers: no context-switching overhead between threads.
* Multiple workers: parallel use of CPU cores.
* No per-connection thread: uses very little memory even with 10k+ connections. 
![Nginx Arch](nginx_arch.png)

### Layer-4 and Layer-7 Balancing
> Layer 4/7 refers to the **OSI** model layers.
>> When we say layer 4 balancing, we mean: balancing requests based on the information available at the transport layer.

>> When we say layer 7 balancing, we mean: balancing based on the application layer details.

#### More on Layer-4 Balancing
* In Layer 4, we see the TCP/IP stack only, nothing about the app, we have access to: Source IP, Source Port, Destination IP, Destination Port.
* NGINX looks only at IP addresses and ports.
* It doesn't know or care if the traffic is HTTP, HTTPs, FTP, or anyhting else.
* It just forwards raw network packets to the right server.
![Layer 4 Balancing](layer-4-balance.png)

#### More on Layer-7 Balancing
* At layer 7, we see the application. We have access to more context.
* We know where the client is going, which page they are visiting.
* Requires decryption.
* NGINX looks inside the request.
* It understands HTTP/HTTPs (or other application layer protocols).
* Can make smarter decisions based on:
    * URL paths (/api vs /images)
    * HTTP headers (e.g., User-Agent, Cookies)
    * Request Header (GET vs POST)
![Layer 7 Balancing](layer-7-balance.png)

#### TLS Termination
* "Termination" means, NGINX ends the TLS connection from the client.
* The client does the TLS handshake with NGINX (not directly with your backend server).
* NGINX uses the server's private key and certificates to decrypt the traffic. Once decrypted, NGINX can see the HTTP request (URL, headers, etc), so it can do Layer 7 load balancing or other processing.
* After that, NGINX either:
    * Sends the request in plain HTTP to the backend (common in private networks) or
    * Starts a new TLS session with the backend (called re-encryption).
* NGINX has the full visibility of the traffic because it decrypts it.

#### TLS Passthrough:
* In passthrough mode, NGINX does not terminate TLS.
* The TLS handshake happens directly between the client and the backend server.
* NGINX just forwards encrypted bytes to the backend server.
* Since NGINX never decrypts, it cannot see HTTP headers, paths, or methods. It can only balance at layer 4 (IP + Port).
* NGINX cannot do content-based routing here.