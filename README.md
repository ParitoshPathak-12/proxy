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

