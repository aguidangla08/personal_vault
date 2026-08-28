# Pinging a Device Over VPN: How Routing Decides It

When you're connected to a company VPN and want to reach a device on a private IP, the VPN client doesn't automatically make every private address reachable. Whether a ping (or any traffic) reaches the device depends on your host's routing table, not just on whether the VPN is "connected."

## What has to be true

**1. A route must exist for that subnet**

Your VPN client adds routes to your system's routing table when it connects, but usually only for the subnets the VPN server has been configured to advertise. If the target IP doesn't fall inside one of those advertised subnets, your machine has no idea it should send that traffic through the tunnel.

**2. Split tunneling vs. full tunnel**

- **Split tunnel**: only specific subnets are routed through the VPN; everything else goes out your normal internet connection. This is the most common reason a "VPN-connected" host still can't reach a given internal IP.
- **Full tunnel**: all traffic goes through the VPN, so this is less likely to be the blocker.

**3. ICMP must be allowed**

Even with a correct route, `ping` specifically needs ICMP echo request/reply to be permitted by any firewall between you and the device (VPN concentrator ACLs, host firewall on the target, network segmentation rules). It's common for ping to be blocked while other protocols (SSH, HTTP, etc.) work fine — so a failed ping doesn't always mean "no connectivity."

## How to check it yourself

Run `ip route` (Linux), `route print` (Windows), or `netstat -rn` (macOS) and look for a route whose destination subnet contains the target IP, pointing at your VPN interface (`tun0`, `utun`, `ppp0`, etc.).

Example routing table:

```
10.6.1.0/24  via 10.86.91.53 dev tun0 proto static metric 50
10.6.3.0/24  via 10.86.91.53 dev tun0 proto static metric 50
10.8.1.0/24  via 10.86.91.53 dev tun0 proto static metric 50
```

- `10.6.1.37` → falls inside `10.6.1.0/24` → **covered**, routed through `tun0`.
- `10.8.8.195` → does **not** fall inside any listed subnet (`10.8.1.0/24` is close but different) → **not covered**. This traffic falls through to the default route instead of the tunnel, so the device is unreachable regardless of ICMP rules.

## Takeaways

- No matching route = traffic never enters the tunnel, full stop. This isn't something you can fix locally; it requires the VPN administrator to push a route for that subnet.
- A matching route but a failed ping = check ICMP filtering before assuming a routing problem.
- Always double-check the actual subnet mask of the target network — an address can look "close" to a covered range while sitting in a different subnet entirely.