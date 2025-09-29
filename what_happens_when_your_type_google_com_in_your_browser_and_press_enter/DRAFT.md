# What happens when you type `https://www.google.com` and press Enter?

> Short answer: a **DNS lookup**, a **secure connection** handshake, then your request flows through **firewalls → load balancer → web server → application server → database**, and the response takes the reverse path back to your browser to render the page. Long answer below.

---

## 1) You press Enter: the browser prepares the request
- Parses `https://www.google.com` → scheme `https`, host `www.google.com`, default port **443**.
- Checks in-memory cache, OS DNS cache, then the system **hosts** file for the hostname.

## 2) DNS resolution (who is `www.google.com`?)
1. If not cached, the browser asks your OS DNS stub resolver.
2. The resolver queries your **recursive** DNS (usually your ISP or a public resolver like 8.8.8.8).
3. The recursive resolver follows the DNS hierarchy:
   - **Root** → **.com** TLD → **google.com** authoritative name servers.
4. It returns one or more **A/AAAA records** (IPv4/IPv6) for `www.google.com`, often **geo-anycasted** and short-TTL for fast steering.

**Result:** your browser has an **IP address** to connect to (likely fronted by Google’s global edge).

## 3) TCP/IP: establishing transport (or QUIC)
With HTTPS, two transport options exist:

- **Traditional path (HTTP/2 over TCP):**
  1. TCP **3-way handshake** (SYN → SYN/ACK → ACK) to port **443**.
  2. Optional **TCP Fast Open** and then TLS handshake (see next section).

- **Modern path (HTTP/3 over QUIC):**
  - QUIC runs over **UDP/443** and folds connection setup + encryption into fewer round trips.  
  - Browsers learn HTTP/3 support via **Alt-Svc/ALPN** and may directly use QUIC next time.

Projects often require the TCP explanation, so we’ll continue assuming TCP.

## 4) Firewalls on the path
- **Client side:** your laptop/mobile and local gateway usually run **stateful firewalls** that allow **outbound 443**.
- **Internet path:** ISP edges and backbone enforce ACLs / anti-DDoS policies.
- **Server side:** cloud **security groups / NACLs / iptables** allow **inbound 443** only to the load balancer and block everything else.

## 5) HTTPS (TLS) handshake
- The client says “I support these cipher suites + ALPN (e.g., h2, http/1.1) + SNI=`www.google.com`”.
- The server (often the **load balancer/edge proxy**) presents a **certificate** valid for `www.google.com`.
- The browser validates trust chain (Root → Intermediate → Leaf) and checks revocation/OCSP stapling.
- They agree on a key via **(EC)DHE** and derive symmetric keys.  
- With **TLS 1.3**, this is 1-RTT (or **0-RTT** for resumption), much faster than older TLS.

From now on, everything is **encrypted in transit**.

## 6) Load balancer (LB)
Your TLS session generally terminates on a global edge/load balancer:
- **Why:** scale, termination offload, **DDoS** protection, routing to the nearest healthy backend.
- **Algo:** Round-Robin / Least-Connections / EWMA / consistent hashing, often with **health checks**.
- **Modes:** L4 (TCP/UDP passthrough) or L7 (HTTP/2, header-aware routing, gzip/brotli, caching).

LB forwards your decrypted HTTP request to a backend pool (sometimes re-encrypting with **mTLS**).

## 7) Web server
- **Nginx/Envoy/Apache** receives the request, handles keep-alives, HTTP/2 multiplexing, static files, compression, caching.
- For dynamic routes, it forwards internally (FastCGI/uwsgi/HTTP) to the **application server**.

## 8) Application server
- Runs the business logic (**Go/Java/Node/Python/…**).  
- Validates the request, checks auth/session, applies routing, reads/writes data via services/caches/DB, builds a response (HTML/JSON).

## 9) Database (and friends)
- The app issues SQL/NoSQL queries to a **primary** (writes) or **replicas** (reads).
- Uses connection pools, prepared statements, and retries on transient errors.
- Stronger systems add **caches** (CDN/edge caches, Redis/Memcached), **message queues**, and **circuit breakers** to reduce DB load and cascade failures.

## 10) Response all the way back
- App server → web server → (maybe gzip/brotli) → LB → encrypted to client → browser.
- Browser parses headers, caches as allowed, streams body; if HTML, it:
  - Parses/executes JS, downloads CSS/images/fonts (often via **HTTP/2 multiplexing**),  
  - Updates the **DOM** and paints.

---

## Where correctness and performance come from
- **DNS:** low TTLs + anycast = proximity and agility.
- **TCP/QUIC:** congestion control, loss recovery; QUIC minimizes latency and head-of-line blocking.
- **TLS:** privacy + integrity; **HSTS** prevents downgrade to HTTP.
- **LB:** health checks, fast failover, rate limiting, WAF rules.
- **Web/App:** caching, compression, microservices boundaries, backpressure.
- **DB:** replication, indexes, read/write splitting, backups, PITR.
- **Monitoring:** SLOs, golden signals (latency, traffic, errors, saturation), logs, traces, metrics.

---

## TL;DR checklist (hits the interview points)
- **DNS request:** recursive resolution to A/AAAA for `www.google.com`.  
- **TCP/IP:** 3-way handshake (or QUIC/HTTP-3).  
- **Firewall:** client & server rules allow 443, block others.  
- **HTTPS/SSL:** TLS 1.3 handshake, cert validation, ALPN.  
- **Load-balancer:** routes to healthy backend (RR/LC/EWMA), L7 features.  
- **Web server:** terminates HTTP, serves static, proxies dynamic.  
- **Application server:** business logic creates the response.  
- **Database:** queries/transactions, often via replicas and caches.

---
**Diagram:** see the architecture flow image referenced in the project.
