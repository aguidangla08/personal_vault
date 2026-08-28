# Network Field Notes

A short reference to build on as we keep debugging. Each section is deliberately brief — we can expand any part you want to go deeper on.

## 1. The network stack

Everything below is one story told at four layers, stacked on top of each other. Each layer only trusts the one underneath — it has no idea whether the layers below it are actually healthy. That's the recurring theme of this whole doc: a layer can report success while the one above it silently fails.

|Layer|What it does|Protocols|
|---|---|---|
|4 — Application|Log in, resolve a name, load a page — the protocol that carries your intent|SSH, DNS, HTTP, HTTPS|
|3–4 — Transport|A connection between a source and destination port; reliable/ordered (TCP) or fire-and-forget (UDP)|TCP, UDP|
|3 — Network|Getting a packet to the right machine; IP addressing/routing plus ICMP control messages|IP, ICMP|
|1–2 — Link|Actual wires, radio, and local addressing between directly connected devices|Ethernet, Wi-Fi, ARP|

The diagnostic tools in this doc each poke at a different layer on purpose: `ping` tests layer 3 (is there a host here at all, via ICMP), `nc` tests layer 3–4 (will something accept a connection on this port), and `ssh`/`dig` test layer 4 (does the actual service behave). That's why a working `ping` tells you almost nothing about whether SSH will work — they're not testing the same layer.

## 2. IP addresses and ports

Every device on a network has an **IP address** (e.g. `192.168.1.10` or `34.201.45.9`) — think of it as a street address for a machine. But a single machine can run many services at once (a web server, an SSH daemon, a database...), so IP alone isn't enough to say _which_ service you want to talk to.

That's what a **port** is: a number (0–65535) that identifies a specific service on that machine. The combination `IP:PORT` (e.g. `34.201.45.9:22`) uniquely identifies one service on one machine.

Some ports are "well-known" by convention:

|Port|Service|
|---|---|
|22|SSH|
|80|HTTP|
|443|HTTPS|
|53|DNS|
|3306|MySQL|

Nothing enforces these — a service _could_ run on any port — but sticking to convention avoids confusion.

## 3. DNS — how names become IPs (and IPs become names)

Humans use names (`gitlab.com`), computers use IPs (`172.65.251.78`). **DNS (Domain Name System)** is the distributed lookup service that translates one into the other. There are actually two distinct directions, and they work differently.

### 3a. Forward lookup: name → IP

This is the everyday case. When you run `ssh myserver.company.com`, before any network connection happens, your machine asks: "what IP is behind this name?"

How that answer gets found:

1. Your machine first checks its **local resolver cache** and `/etc/hosts` (a plain text file where you can hardcode name→IP mappings yourself — useful for testing).
2. If not cached, it asks a **recursive resolver** — usually your ISP's, your company's internal DNS server, or a public one like `8.8.8.8` (Google) or `1.1.1.1` (Cloudflare). This is configured in `/etc/resolv.conf` on Linux.
3. That resolver, if it doesn't already know the answer, walks the DNS hierarchy: it asks a **root server** ("who handles `.com`?"), then a **TLD server** for `.com` ("who handles `company.com`?"), then that domain's **authoritative nameserver** ("what's the IP for `myserver.company.com`?"). This is called **recursive resolution**.
4. The answer — an **A record** (IPv4) or **AAAA record** (IPv6) — comes back and gets cached for a while (controlled by the record's **TTL**, time-to-live) so the whole chain doesn't need repeating every time.

Do it yourself:

```bash
dig myserver.company.com
# or, simpler output:
nslookup myserver.company.com
# or, to see the exact resolver path being used:
dig +trace myserver.company.com
```

### 3b. Reverse lookup: IP → name

This is the opposite direction, and it's a genuinely separate mechanism — DNS doesn't just "look backward" through the same records. There's a dedicated namespace for it called **`in-addr.arpa`** (for IPv4) / **`ip6.arpa`** (for IPv6).

Here's the trick: an IP like `34.201.45.9` gets reversed and turned into a domain-style name: `9.45.201.34.in-addr.arpa`. DNS then does a normal lookup on _that_ name, looking for a special record type called a **PTR record**, which points back to a hostname.

```bash
dig -x 34.201.45.9
# equivalent to:
dig PTR 9.45.201.34.in-addr.arpa
```

The critical thing to understand: **a PTR record only exists if whoever controls that IP block explicitly created one.** Forward DNS (name→IP) is set up by whoever owns the _domain_; reverse DNS (IP→name) is set up by whoever owns the _IP address block_ — usually your hosting/cloud provider (AWS, GCP, a datacenter), not you. That's why reverse lookups very often come back empty, even for servers that have a perfectly good forward name.

This directly explains the `UseDNS` behavior mentioned earlier: when you `ssh` into a server by raw IP, the _server itself_ often tries to reverse-resolve _your_ connecting IP (to log a hostname instead of just a number, or to match against access rules). If your IP has no PTR record, or the reverse lookup path is slow/broken, sshd can sit there waiting on that lookup before continuing — which can look exactly like a hang, even though it's unrelated to the SSH protocol itself.

### Quick mental model

|Direction|Record type|Who controls it|Command|
|---|---|---|---|
|name → IP|A / AAAA|Domain owner|`dig example.com`|
|IP → name|PTR|IP block owner (usually the hosting provider)|`dig -x <ip>`|

If you connect by raw IP directly (like we've been doing — `ssh <ip>`), forward DNS is skipped entirely for _your_ connection — but as explained above, the server side may still trigger a reverse lookup on you.

## 4. TCP — the transport underneath

**TCP (Transmission Control Protocol)** is how two machines establish a reliable, ordered connection over IP. Before any actual data (like an SSH banner) is exchanged, TCP does a **three-way handshake**:

```
Client                    Server
  |------ SYN ----------->|      "I want to connect"
  |<---- SYN-ACK ---------|      "OK, acknowledged"
  |------ ACK ----------->|      "Confirmed, let's go"
```

Once this completes, the connection is "established" — but that only means _a pipe exists_. It says nothing about whether the application on top (SSH, HTTP, etc.) is actually working. This is exactly the gap you've been hitting: `ssh -vvv` reported "Connection established" (TCP succeeded), but then nothing came back at the _application_ layer (SSH itself).

A TCP connection can fail in different ways, and each tells you something different:

- **Refused** — nothing is listening on that port at all (fails instantly)
- **Reset** (`RST`) — something actively tore down the connection after accepting it (a firewall rule, an overloaded service, a proxy)
- **Timeout / hang** — packets are going into a black hole; nothing responds at all, and nothing actively rejects it either (often a silent firewall drop, a misconfigured load balancer, or a genuinely dead service that never replies)

## 5. SSH — what happens on top of TCP

**SSH (Secure Shell)** is a protocol for secure remote login and command execution, built on top of a TCP connection (traditionally port 22). Roughly, once TCP is established, SSH goes through these phases:

1. **Identification string exchange** — both sides announce their SSH version (`SSH-2.0-OpenSSH_9.6p1...`). This is the exact phase where your connection is hanging — your client has sent its string and is waiting for the server's.
2. **Key exchange (KEX)** — client and server agree on encryption algorithms and generate a shared session key, so everything from this point on is encrypted.
3. **Host key verification** — your client checks the server's identity against `~/.ssh/known_hosts` (this is the "authenticity of host can't be established" prompt you may have seen before, on a _different_ connection).
4. **Authentication** — password, public key, etc.
5. **Session** — once authenticated, you get a shell, or a command runs, or a tunnel opens.

Because SSH layers on top of TCP, a working TCP handshake is necessary but not sufficient — everything can still go wrong afterward, which is what we're seeing.

## 6. ICMP and `ping`

**ICMP (Internet Control Message Protocol)** is a network-layer protocol that sits alongside IP, not on top of TCP or UDP. It carries diagnostics and error reports, not application data — it's plumbing, not payload.

The relevant message pair is **echo request / echo reply** — that's literally what `ping` sends and listens for. When you `ping` an IP, your machine sends an ICMP echo request; if a host at that address is up and configured to respond, it sends back an ICMP echo reply, and `ping` reports the round-trip time.

```bash
ping -c 4 34.201.45.9
```

ICMP also carries other control messages worth knowing: **destination unreachable** (host or port can't be reached), **time exceeded** (used by `traceroute` to map hops), and **redirect** (a router telling you to use a better route), among others.

**The catch:** ICMP has no concept of ports or services — it's a yes/no "is this address reachable and willing to answer" signal, and nothing more. It's also routinely disabled or filtered by firewalls, security groups, and hardened hosts, precisely because it can be used for reconnaissance (mapping live hosts) or abuse (ping floods).

So a non-response to `ping` just means "this host didn't answer ICMP" — which could mean nothing lives there, or could mean something lives there and ICMP is blocked. You can't tell which from the outside. **A live machine can look "unreachable" to `ping` for reasons that have nothing to do with whether it's actually up.**

This is the direct answer to "is an IP available": `ping` can confirm an IP is _in use_ (a reply means someone's there), but it can never confirm an IP is _free_ — silence is ambiguous, not conclusive.

## 7. `nc` (netcat) — the "swiss army knife" for testing raw connections

`nc` lets you open a raw TCP (or UDP) connection without any protocol logic on top — no SSH, no HTTP, nothing. It's useful precisely because it lets you isolate _which layer_ is broken, and unlike `ping`, it's testing at the transport layer against one specific port.

```bash
nc -v <ip> 22
```

- If this immediately prints something like `SSH-2.0-OpenSSH_9.6p1 Ubuntu...`, the server is alive and answering at the TCP+banner level — meaning any further hang would have to be an SSH-client-specific issue (unlikely, but worth knowing).
- If `nc` also just hangs with no banner, it confirms the problem is **not SSH-specific** — it's below SSH, at the TCP/network level: the port accepts connections but nothing behind it responds.

This is exactly the test I suggested for your current issue — it tells us whether to keep looking at SSH config, or shift entirely to "is the server/network even alive."

### `ping` vs `nc` for checking IP availability

||`ping`|`nc`|
|---|---|---|
|Layer tested|Network (ICMP)|Transport (TCP/UDP, one port)|
|Confirms "in use"|Yes, if it replies|Yes, if a port accepts or shows a banner|
|Confirms "free"|Never — silence is ambiguous|Never — a closed port ≠ no host|
|Common false negative|ICMP blocked by firewall|Wrong/filtered port, needs a guess|

For allocating IPs in CI, the reliable check is usually at the allocation layer itself — ARP/neighbor tables (`arp -a` / `ip neigh`), DHCP lease reservations, or your IPAM/inventory source — rather than an active probe of either kind.

## 8. Putting it together for your current problem

Your `-vvv` trace shows:

```
debug1: Connection established.        <- TCP handshake succeeded
debug1: Local version string ...       <- your client sent its SSH banner
[hang]                                  <- server never sent its banner back
```

This means: the network path allows a TCP connection to be _opened_, but no SSH-layer response ever comes back. That's consistent with sshd being unhealthy, a proxy/load-balancer accepting connections it can't actually forward, or a firewall silently dropping return traffic. It is **not** something fixable from ssh flags or client-side config — the next step is checking the server/network side directly (which is where the `nc` test and checking server health come in).

---

Let me know which part you want to go deeper on — TCP handshakes and packet-level detail, SSH's key exchange and host keys, DNS record types, or something else.