# Tracing a Hostname Back Through DNS

Field notes from a session spent chasing an IP set up by your company — how lookups resolve, what the local stub resolver is doing, and how to point queries at your own DNS.

## 1. Reading a forward lookup

```
$ nslookup gitlab.com
Server:         192.168.1.1
Address:        192.168.1.1#53

Non-authoritative answer:
Name:   gitlab.com
Address: 172.65.251.78
```

- **Server** is the resolver that answered your query (often your router, e.g. `192.168.1.1`), not the domain's own nameserver. It forwards the request upstream and hands back whatever it gets.
- **Non-authoritative answer** just means the answer came from a resolver's cache or an upstream forwarder — not the domain's own authoritative nameservers. It's routine, not an error.

## 2. 127.0.0.53 — the local stub resolver

On most modern Debian/Ubuntu systems (common on CI runners), DNS doesn't go straight to your router. `/etc/resolv.conf` points every app at `127.0.0.53` — a loopback address running **systemd-resolved**, a local stub DNS server. It forwards your query to the real upstream servers (from DHCP or manual config), caches the response, and returns it.

Seeing `Server: 127.0.0.53` just means the local stub answered — the actual upstream may not even have been contacted if the answer was cached.

```
$ resolvectl status          # see the real upstream servers behind the stub
$ cat /etc/resolv.conf        # confirm what nameserver apps are pointed at
```

## 3. Private IP ranges (RFC 1918)

Reserved for internal networks only — not routable on the public internet.

|Range|CIDR|Typical use|
|---|---|---|
|10.0.0.0 – 10.255.255.255|10.0.0.0/8|Large corporate networks|
|172.16.0.0 – 172.31.255.255|172.16.0.0/12|Mid-size networks, containers|
|192.168.0.0 – 192.168.255.255|192.168.0.0/16|Home & small office routers|

Easy to eyeball-confuse: `172.65.x.x` looks private but isn't — the 172 private block only spans `172.16`–`172.31`.

**Checked in this session:** `172.65.251.78` (gitlab.com, fronted by Cloudflare) sits outside `172.16.0.0/12` — it's a real public IP, not an internal address.

## 4. If a company IP _is_ private

If your company's IP does fall in a reserved range, here's what's typically behind it:

- **Split-horizon / internal DNS** — the company runs its own DNS server that overrides public records on the corporate network. A name (often an internal alias, sometimes the public name itself) resolves to a private IP on-network, pointing at a self-hosted service — while off-network, it resolves differently or not at all.
- **Hosts file override** — less common at scale, but `/etc/hosts` can hardcode a name to an IP directly, bypassing DNS.
- **Proxy or mirror with a public IP** — a caching proxy or artifact mirror the company manages, reachable at a real public address even though it's "internal" in ownership.

## 5. Reverse lookups against a specific DNS server

Point the query at a specific server by adding it as an argument, rather than relying on the machine's default resolver:

```
$ nslookup -type=PTR <ip-address> <company-dns-ip>
$ dig -x <ip-address> @<company-dns-ip>
```

Find the company resolver's address first if it isn't already the default:

```
$ resolvectl status
$ cat /etc/resolv.conf
```

**Common outcome:** `** server can't find <ip>: NXDOMAIN` — many internal networks keep forward records but never configure reverse (PTR) ones. If that happens, a DHCP lease table or internal IPAM/asset inventory is usually the more reliable way to map an internal IP to a hostname.

---

_Compiled from a live troubleshooting session — gitlab.com / CI network debugging._