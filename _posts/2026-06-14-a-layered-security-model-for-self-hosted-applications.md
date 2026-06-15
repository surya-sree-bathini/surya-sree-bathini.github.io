---
title:  "A Layered Security Model for Self-Hosted Applications"
date:   2026-06-14 22:19:50 +0530
categories:
  - blog
tags:
  - DevOps
  - Networking
---
## Hosting Internal Services Publicly

The scenario is common: you have a service where **some routes must be reachable from the
public internet** (a webhook, a service-to-service API, an integration
endpoint) while **everything else - the admin UI, the management APIs, the
database - must never touch the internet at all.**

"It's behind authentication" feels like an answer to that. It isn't a
complete one. Auth is *a* layer; it is not a perimeter. This guide is about
building the perimeter underneath it: what each network layer actually
protects, what it can't, and the gotchas that quietly undo the whole thing
(the big one: Docker bypasses your host firewall - really).

Nothing here is exotic. It's a stack of boring, well-understood pieces
arranged so that if any single one fails, the next one is still in the way.

---

### What "not on the internet" actually means

Ask three people whether a service is "on the internet" and you'll get
three different answers, all of them partly right:

- **Network engineer:** "It's not in a public subnet / no public route to it."
- **Application engineer:** "Auth middleware blocks unauthorized access."
- **Security engineer:** "The port isn't even listening on a public interface."

Each describes a *different layer*. None of them is the whole story. A
service is genuinely not-on-the-internet only when several of these are
true at once, because each one says *no* to a different kind of mistake and
a different kind of attacker.

The mental model that makes the rest of this guide click:

```
┌────────────────────────────────────────────────────────────────┐
│  Layer 7 (HTTP)     reverse proxy - path-level: allow-listed   │
│                     routes only; everything else is dropped    │
│  Layer 4 (TCP/UDP)  host firewall - only 22/80/443 open        │
│  Layer 4 (TCP/UDP)  cloud firewall - same idea, at the VPC edge│
│  Layer 3 (kernel)   no host port published for the service -   │
│                     it exists only on an internal bridge       │
└────────────────────────────────────────────────────────────────┘
                              │
                       Public Internet
```

A packet from the internet has to clear **every lower layer** before the
next one even gets to look at it. The cloud firewall decides whether the
packet enters the VM at all; the host firewall decides whether the kernel
accepts it; the kernel can only hand it to a proxy if a port is actually
listening; the proxy decides whether the *path* is one it's willing to
forward. Four independent "no" gates, each cheap, each catching a class of
error the others can't see.

The rest of this is each layer in turn, bottom to top.

---

### Layer 3: make the port not exist

The strongest control is the one where there's nothing to attack. Before
any firewall rule, ask: **does the sensitive service even have a port on a
public interface?** If it doesn't, no amount of misconfigured firewall can
expose it.

This is where containers trip people up, so it gets its own section below
(see "The Docker gotcha"). The short version: the default way of running a
container publishes its port on **all** host interfaces, including the
public one. The fix is to *not publish a host port at all* and reach the
container over an internal bridge network instead. We'll get there. Hold
the thought that "the port simply isn't there" is the cleanest defense, and
everything above this layer is insurance for when it *is* there.

---

## Layer 4: the cloud firewall

Every cloud has a network firewall that sits **in front of** your VM,
evaluated at the network edge before a packet reaches the instance's
network interface. AWS calls them Security Groups; other clouds have the
same concept under different names. The defining trait: it's configured on
the *cloud control plane*, not inside the VM. The OS running inside the VM
doesn't even know these rules exist.

A typical rule set for a host that exposes only public web ports:

| Port  | Source            | Why                                                      |
| ----- | ----------------- | -------------------------------------------------------- |
| 22    | specific admin IPs| SSH - administrative access only                         |
| 80    | `0.0.0.0/0`       | ACME HTTP-01 cert challenges + redirect to HTTPS         |
| 443   | `0.0.0.0/0`       | The actual public traffic (HTTPS)                        |
| 5432  | (not present)     | Database - deliberately has no rule at all               |

That's the entire power of this layer: **yes/no per port, per source IP.**
It cannot see HTTP. It cannot rate-limit. It can't tell `/healthz` from
`/admin/secrets` - those are layer-7 concepts and this is a layer-4 filter.
It's the heavy iron at the front gate, and that's exactly what you want it
to be.

### Dynamic admin IPs for SSH

Hard-coding your home IP into the SSH rule breaks the moment you're on a
different network (coffee shop, hotspot, ISP re-assignment). A small
wrapper around your SSH session keeps it current instead:

```bash
MY_IP=$(curl -s ifconfig.me)
# add this IP to the SSH ingress rule on the cloud firewall, e.g.:
aws ec2 authorize-security-group-ingress \
    --group-id sg-xxxxx --protocol tcp --port 22 \
    --cidr "$MY_IP/32" || true
ssh -i ~/.ssh/host.pem user@host
# (optional) revoke the rule afterwards to keep the allow-list tight
```

The principle generalizes to any cloud: **never leave SSH open to
`0.0.0.0/0`.** Narrow it to known IPs, and automate the narrowing so the
friction doesn't tempt you into widening it.

### A CIDR notation primer

Source IPs in firewall rules are written in CIDR notation - `IP/N`, where
`N` is how many leading bits are fixed:

| CIDR              | Meaning                            | Addresses    |
| ----------------- | ---------------------------------- | ------------ |
| `203.0.113.42/32` | exactly that one IP                | 1            |
| `203.0.113.0/24`  | `203.0.113.0` – `203.0.113.255`    | 256          |
| `10.0.0.0/16`     | a `/16`, common for private VPCs   | 65,536       |
| `0.0.0.0/0`       | every IPv4 address - "the world"   | 4.3 billion  |

A single host is `/32`. A typical office subnet is `/24`. "Open to the
world" is `/0`. That's nearly all the CIDR you need to read and write
firewall rules.

---

## Layer 4 again: the host firewall

A host firewall lives **inside the VM** and is enforced by the kernel - on
Linux this is `iptables`/`nftables`, usually fronted by something friendly
like UFW (Uncomplicated Firewall). It does the same job as the cloud
firewall (port-level allow/deny) in a different place.

The obvious objection: *if the cloud firewall already does this, why run a
second one?* The honest answer is **defense in depth against your own
future mistakes.** The cloud firewall is the primary control. The host
firewall catches the day someone runs `python -m http.server` for a quick
debug, or installs a metrics exporter that helpfully binds `0.0.0.0:9100`,
or widens a cloud rule "just for a minute" and forgets. The host firewall
says *those ports aren't open on this machine*, and the mistake stays
contained to the box.

A minimal, mirror-the-cloud-rules config:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow 22/tcp   comment "SSH"
sudo ufw allow 80/tcp   comment "HTTP (ACME + redirect)"
sudo ufw allow 443/tcp  comment "HTTPS"

sudo ufw enable         # ONLY after the SSH rule exists, or you lock yourself out
sudo ufw status verbose
```

Things worth knowing before you trust it:

- **`enable` over SSH is the classic foot-gun.** Your *current* session
  survives, but the next connection is refused if you forgot the SSH allow.
  Always confirm the SSH rule is present *before* enabling.
- **Order matters when rules overlap.** UFW is first-match; use
  `ufw insert <num>` to place a rule at a specific position.
- **IPv6 is separate.** It's on by default; make sure your v6 rows mirror
  your v4 rows, or you'll close a door on one protocol and leave it open on
  the other.
- **`ufw reset` wipes everything** - handy for clean re-runs of a setup
  script, destructive if you have rules you care about.

### The Docker gotcha

This is the one that surprises nearly everyone, and it silently defeats
everything above. **Docker bypasses the host firewall.**

When you publish a container port the usual way:

```yaml
ports:
  - "8080:8080"
```

Docker writes its *own* `iptables` rules into a chain that sits **ahead of**
the firewall's chains. The result: port 8080 is open to the entire internet
even though `ufw status` proudly says "deny incoming." UFW isn't lying -
it's simply consulted *after* Docker has already accepted and NAT'd the
packet. Your host firewall and your container port-publishing are fighting,
and the container wins.

A common first fix is to bind the publish to loopback:

```yaml
ports:
  - "127.0.0.1:8080:8080"
```

That does remove it from the public interface - but it still creates a
Docker NAT rule and a host-side listener. There's a cleaner approach that
sidesteps the whole mechanism: **don't publish a host port at all.**

```yaml
services:
  app:
    # note: NO `ports:` line
    networks:
      app-net:
        ipv4_address: 172.20.0.53
networks:
  app-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24
```

Now the service lives only on a user-defined Docker bridge with a pinned
IP. The host kernel owns the bridge interface, so **any process on the
host** - a reverse proxy, sshd, a `curl` you type - can route to
`172.20.0.53` directly, because it's just a normal local IP on a
directly-attached network. But there is no host-side port, no Docker NAT
rule, and nothing reachable from outside the VM regardless of what the
firewalls say.

The pinned `ipv4_address` is load-bearing: without it, Docker hands out a
different bridge IP on each container restart, and anything that addresses
the container by IP (your proxy config, your SSH tunnel commands) breaks
silently after the next deploy.

This is the strongest layer in the stack precisely because it's not a
*rule* that can be misconfigured - the port doesn't exist on any host
interface, not even loopback. Short of editing the compose file, it cannot
become public.

---

## Layer 7: a reverse proxy as a path filter

So the service now sits on an internal bridge IP and the internet can't
touch it. How does the legitimately-public traffic get in?

A reverse proxy. It runs **on the host**, listening on the public ports (80
and 443), and because it's on the same host that owns the bridge interface,
it can reach the internal service over a normal local route. The internet
reaches the proxy; the proxy reaches the service across the bridge. The
proxy is the seam between the two worlds.

```
Internet ──► proxy :443 (public) ──► 172.20.0.53:8080 (internal bridge)
                  │
                  └─ forwards ONLY the allow-listed path prefix
                     everything else → dropped
```

The key property: the proxy **re-originates** the connection. The public
client opens a TCP connection to the proxy; the proxy opens a *separate*
connection to the backend and copies bytes between them. From the backend's
point of view the client is always the bridge gateway (the host). From the
internet's point of view the only ports that exist are 80 and 443. The two
sides never share a socket.

An abridged nginx config that allows exactly one prefix and silently drops
the rest:

```nginx
limit_req_zone $binary_remote_addr zone=public_api:10m rate=10r/s;

server {
    listen 443 ssl http2;
    server_name app.example.com;

    ssl_certificate     /etc/letsencrypt/live/app.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;

    set $backend http://172.20.0.53:8080;

    location /api/v1/public/ {
        limit_req zone=public_api burst=20 nodelay;
        proxy_pass $backend;
    }

    location / {
        return 444;     # close the connection silently
    }
}
```

A few deliberate choices in there:

- **Group everything public under one path prefix.** Then the proxy matches
  a single prefix, and adding a new public endpoint upstream doesn't require
  a proxy edit. Without a prefix convention you accumulate exact-path
  matches and someone eventually forgets to add one.
- **`return 444`** is nginx's "close the TCP connection and send nothing."
  It leaves no fingerprint that a URL does or doesn't exist - whether a
  scanner probes `/admin` or `/api/v1/users`, it gets identical silence.
- **Rate limiting lives here**, not in the app. Layer 7 is the natural place
  for it because it's the first thing that can actually read the HTTP
  request.
- **TLS terminates here.** Certificates from Let's Encrypt via
  `certbot --nginx` (or the proxy's built-in ACME) handle issuance and
  renewal on a timer. Nothing exotic.

### Why filter paths when the app already has auth?

You could argue the backend already gates everything behind auth, so why
duplicate that at the proxy? Three reasons:

1. **Defense in depth.** The day someone mounts a new admin route and
   forgets the auth middleware, the public surface is *still* only the paths
   the proxy explicitly forwards. The bug stays internal.
2. **Attack surface and noise.** Path filtering means scanners and
   brute-force tools see nothing useful - fewer probes reach the backend,
   less log noise, less wasted load.
3. **A place to put policy.** Per-route rate limits, IP restrictions, WAF
   rules - when you need them, the proxy is where they naturally go, and you
   already have the seam.

Auth answers "is this caller allowed?" The proxy answers "should this
request reach the application at all?" Those are different questions and you
want both answered.

---

## Admin access without a public UI

If the management UI and admin APIs aren't on the internet, how does the
team use them? **SSH port-forwarding.**

```bash
ssh -i ~/.ssh/host.pem -L 8443:172.20.0.53:8080 user@app.example.com
```

The `-L 8443:172.20.0.53:8080` opens a listener on **your laptop's** port
8443, tunnels it through the SSH session to the remote host, and exits at
the internal bridge IP on port 8080. Browsing to `http://localhost:8443/`
on your laptop now reaches the full UI and full API - over the encrypted
SSH channel, gated by the SSH key and the firewall's admin-IP allow-list.

Why this is sound:

- The only thing exposed is **port 22**, already restricted to known admin
  IPs and protected by key-based auth.
- `ssh -L` can forward to any address the *remote host* can reach. The host
  owns the bridge, so the internal IP is a local route for it - sshd opens
  the connection exactly as the on-host proxy would.
- The backend sees a connection from the bridge gateway, identical to a
  proxied request. Nothing special is required on the application side.
- **There is no public login URL to brute-force.** "Internal-only" stays
  literally true even though the team uses the tool daily.

For larger teams or stricter environments, the cloud-native equivalent is a
**bastion host** or a **session-manager service** (e.g. AWS SSM Session
Manager) that lets you eliminate the open port 22 entirely. SSH
port-forwarding is the right amount of machinery for a small team; the
managed options are there when you outgrow it.

---

## The database: native on the host, off every public surface

It's tempting to run everything in containers, but a database often belongs
*natively* on the host: simpler backups, straightforward
point-in-time recovery, no volume juggling, one fewer thing to orchestrate.
You can always move it to its own VM later if compute or storage pressure
shows up. The interesting part is the container-to-host networking.

From inside the application container, `localhost` is the *container*, not
the host - so `localhost:5432` won't find a database running natively on
the VM. Two changes bridge the gap.

**1. Expose the host to the container by name.** Most container runtimes can
inject a hostname that resolves to the bridge gateway:

```yaml
services:
  app:
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

Inside the container, `host.docker.internal` now resolves to the bridge
gateway IP (e.g. `172.20.0.1`) - the host as seen from the container's
network.

**2. Tell the database to listen on the bridge gateway** (in addition to
loopback) and to accept connections from the bridge subnet:

`postgresql.conf`:
```
listen_addresses = 'localhost,172.20.0.1'
```

`pg_hba.conf`:
```
host    app_db   app_user   172.20.0.0/24   scram-sha-256   # the app container
host    app_db   app_user   127.0.0.1/32    scram-sha-256   # SSH-tunneled admin
```

The application then connects via the bridge name:

```
DB_URL=postgres://app_user:<password>@host.docker.internal:5432/app_db
```

Crucially, the database still has **no public surface anywhere.** The bridge
IP is internal to the VM; loopback is internal to the VM; neither appears in
any firewall rule. There is no port 5432 reachable from the internet at any
layer - and the database believes it's only talking to local clients.

### Connecting a DB GUI from your laptop

Because the database listens only on loopback and the bridge, a tool like
DBeaver or pgAdmin can't connect directly. Use the same SSH tunnel trick as
the UI - either add another forward to your SSH command:

```bash
ssh -i ~/.ssh/host.pem \
    -L 8443:172.20.0.53:8080 \
    -L 5433:127.0.0.1:5432 \
    user@app.example.com
# laptop: UI at localhost:8443, DB at localhost:5433
```

…or use the GUI's built-in SSH-tunnel option, pointing it at port 22 with
the same key and setting the DB host to the *remote-side* `localhost:5432`.
Either way the database connection rides the same SSH channel that gates all
other admin access - one perimeter, not two.

---

## Tracing requests through the stack

The payoff of all this is that you can trace any request and see exactly
where it lives or dies.

**Allowed - legitimate public traffic:**

```
1. POST https://app.example.com/api/v1/public/event
2. Cloud FW: 443 allowed from 0.0.0.0/0  → packet enters the VM
3. Host FW:  443 allowed                 → kernel accepts → handed to proxy
4. Proxy:    path matches the allow-list → forwards to 172.20.0.53:8080
5. App (via bridge): authenticates the request → 200 OK
6. Proxy copies the response back to the caller.
```

**Blocked - someone probes for admin routes:**

```
1. GET https://app.example.com/api/v1/admin/users
2. Cloud FW: 443 allowed → enters VM
3. Host FW:  443 allowed → accepts → handed to proxy
4. Proxy: no matching location → falls through to `return 444`
5. Connection dropped. The app never sees the request.
```

**Blocked - someone tries to skip the proxy:**

```
1. curl http://<public-ip>:8080/api/v1/admin/users
2. Cloud FW: 8080 not in the allow-list → dropped at the edge.
   (And even if it weren't: there is no host listener on 8080 at all -
    the port exists only on the internal bridge, unreachable from outside
    the VM. Connection refused at the kernel.)
```

**Allowed - admin uses the internal UI:**

```
1. Laptop: ssh -L 8443:172.20.0.53:8080 user@app.example.com
2. Cloud FW: 22 allowed from the admin IP → enters VM
3. Host FW:  22 allowed → sshd accepts → SSH session established
4. Laptop: open http://localhost:8443/
5. The tunnel forwards bytes to 172.20.0.53:8080 over the bridge
6. App serves the full UI + API. Admin works normally.
```

Every *allowed* path is a deliberate decision. Every *blocked* path has **at
least two** independent layers saying no.

---

## Verifying it - don't trust, check

A design is only as good as your ability to prove it. Run these on the host
and from a remote machine before you call it done.

```bash
# On the host: confirm the service has NO host-side listener at all.
sudo ss -tlnp | grep 8080
# Expect: nothing.

# On the host: confirm the host knows the bridge route.
ip route get 172.20.0.53
# Expect: ... dev <bridge interface> ...

# On the host: confirm the host CAN reach the service over the bridge.
curl http://172.20.0.53:8080/healthz
# Expect: 200.

# From your laptop: confirm the service is NOT reachable directly.
curl -v http://<public-ip>:8080/healthz
curl -v http://<public-ip>:8080/healthz
# Expect: connection refused / timeout for both.

# From your laptop: confirm the allowed public path works over TLS,
# and a non-allowed path is dropped.
curl -v https://app.example.com/api/v1/public/health   # expect 200
curl -v https://app.example.com/api/v1/admin/users     # expect dropped (no response)
```

If any of these doesn't match the expectation, a layer isn't doing what you
think - find out which before you ship.