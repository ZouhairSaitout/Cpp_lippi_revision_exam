# Full DNS recursion through a WireGuard tunnel: lab guide

> **Goal**: resolve `www.google.com` **by hand** with `dig`, from the root servers down, like a
> recursive resolver does, from **any network** (residence, office without VPN, anywhere).
>
> **Problem**: some networks hijack port 53. Recursion *needs* plain port 53, because root, TLD and
> authoritative servers only speak plain DNS.
>
> **Solution**: your isolated Incus VM (`dnsvm`) sends **everything** into an encrypted WireGuard
> tunnel to a free Oracle Cloud server. The traffic comes out there, and *only then* goes to the
> root servers on port 53. The local network sees nothing but encrypted UDP.

This guide continues `labo-dns-vm-isolee.md`: it assumes the `dnsvm` VM and the `dnsiso`
network from §1 of that file already exist.

---

## Contents

0. [The big picture](#0-the-big-picture)
1. [Part 1: the Oracle Cloud server](#part-1--the-oracle-cloud-server)
2. [Part 2: WireGuard on the server](#part-2--wireguard-on-the-server)
3. [Part 3: WireGuard inside the VM only](#part-3--wireguard-inside-the-vm-only)
4. [Part 4: prove the tunnel beats the interception](#part-4--prove-the-tunnel-beats-the-interception)
5. [Part 5: the exercise, recursion by hand with dig](#part-5--the-exercise-recursion-by-hand-with-dig)
6. [Part 6: let Unbound do it (mode C)](#part-6--let-unbound-do-it-mode-c)
7. [Part 7: network and security tools tour](#part-7--network-and-security-tools-tour)
8. [When the tunnel is blocked](#8-when-the-tunnel-is-blocked)
9. [Cleanup and troubleshooting](#9-cleanup-and-troubleshooting)

---

## 0. The big picture

```
 your host (unchanged)
 └── VM dnsvm ──── wg0 (encrypted, UDP 51820) ────────▶ Oracle server ──▶ port 53 ──▶ root servers
       │                                                   (NAT)                       .com servers
       └── enp5s0 → residence network: sees only                                       google.com servers
                    encrypted UDP to one IP
```

- **Recursion happens in the VM** (your `dig` or your Unbound).
- The Oracle server is just an **exit door**: it decrypts and forwards. It doesn't resolve anything.
- Your **host never changes**: WireGuard is installed and configured only inside the VM.

> Using a personal VPN is ordinary, but if your residence's rules say anything about VPNs, follow them.

---

## Part 1: the Oracle Cloud server

### 1.1 Create an SSH key (inside the VM, to keep everything isolated)

```bash
incus shell dnsvm
ssh-keygen -t ed25519 -f ~/.ssh/oracle -N ""
cat ~/.ssh/oracle.pub          # copy this line, you'll paste it into Oracle
```

### 1.2 Create the account

1. Go to **oracle.com/cloud/free** and create an account. A card is required for identity
   verification; **Always Free** resources are not billed.
2. Choose your **home region** carefully, because it can't be changed later. **France Central
   (Paris)** or **France South (Marseille)** are the closest to you.

### 1.3 Create the instance

In the console: **Compute → Instances → Create instance**.

| Setting | Value |
|---|---|
| Name | `dns-exit` |
| Image | **Canonical Ubuntu 24.04** |
| Shape | **VM.Standard.E2.1.Micro** (marked *Always Free-eligible*) |
| Networking | create a new VCN with a **public subnet**, and tick **assign a public IPv4 address** |
| SSH keys | **Paste public key** → the `oracle.pub` line from 1.1 |

Create it, wait for the **Running** state, and write down the **public IP** (called `ORACLE_IP`
below).

> The *Ampere A1* shape is also free and more powerful, but often shows "out of capacity".
> The E2.1.Micro is plenty for a tunnel.

### 1.4 Open the WireGuard port in Oracle's cloud firewall

**Networking → Virtual Cloud Networks → your VCN → your subnet → Default Security List →
Add Ingress Rules**:

| Field | Value |
|---|---|
| Source CIDR | `0.0.0.0/0` |
| IP Protocol | **UDP** |
| Destination Port Range | `51820` |

(Add a second identical rule for UDP **443**, which you'll need for the fallback in §8.)

### 1.5 First connection

From the VM:

```bash
ssh -i ~/.ssh/oracle ubuntu@ORACLE_IP
```

The user is `ubuntu` on Canonical images.

---

## Part 2: WireGuard on the server

All commands in this part run **on the Oracle server**.

### 2.1 Install and generate keys

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y wireguard

wg genkey | sudo tee /etc/wireguard/server.key | wg pubkey | sudo tee /etc/wireguard/server.pub
sudo chmod 600 /etc/wireguard/server.key
sudo cat /etc/wireguard/server.key     # needed in 2.4
cat /etc/wireguard/server.pub          # needed in Part 3
```

### 2.2 Let the server forward packets

```bash
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-wireguard.conf
sudo sysctl --system
ip route | grep default        # note the interface name, usually "ens3"
```

### 2.3 ⚠️ Oracle's own iptables rules

Oracle's Ubuntu images ship with **strict iptables rules inside the server** that block everything
except SSH, **in addition to** the cloud firewall from 1.4. This is the #1 Oracle gotcha.

```bash
sudo iptables -I INPUT -p udp --dport 51820 -j ACCEPT
sudo iptables -I INPUT -p udp --dport 443 -j ACCEPT      # for the §8 fallback
sudo netfilter-persistent save                           # survive reboots
```

### 2.4 Server config

Get the VM's public key first (Part 3.1), then:

```bash
sudo nano /etc/wireguard/wg0.conf
```

```ini
[Interface]
Address    = 10.99.0.1/24
ListenPort = 51820
PrivateKey = <contents of /etc/wireguard/server.key>

# Accept forwarded tunnel traffic, and NAT it out to the internet
PostUp   = iptables -I FORWARD 1 -i wg0 -j ACCEPT; iptables -I FORWARD 1 -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -s 10.99.0.0/24 -o ens3 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -s 10.99.0.0/24 -o ens3 -j MASQUERADE

[Peer]
# dnsvm
PublicKey  = <dnsvm public key from Part 3.1>
AllowedIPs = 10.99.0.2/32
```

Replace `ens3` if 2.2 showed another name.

```bash
sudo systemctl enable --now wg-quick@wg0
sudo wg show
```

### 2.5 Basic hardening (it's a public server)

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades      # answer Yes
```

SSH already accepts keys only on Oracle images. Check:
`sudo sshd -T | grep passwordauthentication` should show `no`.

---

## Part 3: WireGuard inside the VM only

All commands in this part run **inside `dnsvm`** (`incus shell dnsvm`). Your host stays untouched.

### 3.1 Install and generate keys

```bash
apt-get install -y wireguard nftables
cd /etc/wireguard
umask 077
wg genkey | tee vm.key | wg pubkey > vm.pub
cat vm.pub         # paste this into the server's [Peer] (Part 2.4)
cat vm.key         # needed just below
```

### 3.2 VM config

```bash
nano /etc/wireguard/wg0.conf
```

```ini
[Interface]
Address    = 10.99.0.2/24
PrivateKey = <contents of /etc/wireguard/vm.key>
MTU        = 1380

[Peer]
PublicKey           = <contents of server.pub>
Endpoint            = ORACLE_IP:51820
AllowedIPs          = 0.0.0.0/0          # ALL of the VM's traffic goes into the tunnel
PersistentKeepalive = 25
```

- `AllowedIPs = 0.0.0.0/0` sends everything through the tunnel, which is what you want: DNS, `apt`,
  `curl`, all of it. `wg-quick` automatically keeps the route to `ORACLE_IP` itself outside the tunnel.
- No `DNS =` line: we manage DNS ourselves (Unbound, or `dig @server`).
- `MTU = 1380` leaves room for the extra headers and avoids weird stalls.

### 3.3 Start it

```bash
wg-quick up wg0
wg show                         # "latest handshake: a few seconds ago" = it works
systemctl enable wg-quick@wg0   # start automatically when the VM boots
```

Turn it off: `wg-quick down wg0`. The VM is back on the plain network.

Snapshot from the host:

```bash
incus snapshot create dnsvm tunnel-ok
```

> `incus exec` / `incus shell` keep working whatever happens to the VM's network: they go through
> a virtual device, not through the network.

---

## Part 4: prove the tunnel beats the interception

Run these **inside the VM**, first with the tunnel **down**, then **up**, and compare.

### 4.1 Where do I come out?

```bash
curl -4 -s ifconfig.me; echo
```

Tunnel up → **ORACLE_IP**. Tunnel down → the residence's public IP.

### 4.2 Is anyone answering in place of real servers?

```bash
dig @192.0.2.1 example.com +time=3 +tries=1
```

`192.0.2.1` is a documentation address, so no DNS server exists there.
**Tunnel up → timeout** (correct). Tunnel down → an answer (= hijacking).

### 4.3 Is the root server really the root?

```bash
dig @198.41.0.4 www.google.com +norec
```

Tunnel up → `ANSWER: 0` and an AUTHORITY section listing `com.` name servers: that's a genuine
**referral**.

### 4.4 What the local network sees

On the VM, in one terminal:

```bash
tcpdump -ni enp5s0 'not udp port 51820'
```

In another (`incus shell dnsvm`):

```bash
dig @198.41.0.4 www.google.com +norec
```

The first terminal shows **nothing**: on the real interface, only encrypted packets to port 51820
leave. Now watch inside the tunnel:

```bash
tcpdump -ni wg0 port 53
```

There your plain DNS queries are visible, but only from the VM's point of view.

---

## Part 5: the exercise, recursion by hand with dig

This is what a recursive resolver does for every name. You'll be the resolver.

**Key flag: `+norec`** (no recursion desired). It tells the server: *"don't resolve it for me,
just tell me what you know."*

Read three sections of each response:

| Section | Meaning |
|---|---|
| **ANSWER** | the final answer, if the server knows it |
| **AUTHORITY** | "I don't know, ask these name servers" (a **referral**) |
| **ADDITIONAL** | the IP addresses of those name servers (**glue**), so you can contact them |

And the **`aa`** flag in the header: *authoritative answer*, meaning the server owns that zone.

> The IPs below are real and stable, but **read them from your own outputs** rather than trusting
> this page. That's the whole point of the exercise.

### Step 0: find a root server

There are 13 root server *names* (`a` to `m.root-servers.net`), each served by many machines
worldwide (anycast).

```bash
dig . NS +short
dig a.root-servers.net A +short       # 198.41.0.4
```

### Step 1: ask the root

```bash
dig @198.41.0.4 www.google.com A +norec
```

You get no answer, but:
- **AUTHORITY**: `com. NS a.gtld-servers.net.` … `m.gtld-servers.net.`
- **ADDITIONAL**: their IPs (e.g. `a.gtld-servers.net. A 192.5.6.30`)
- no `aa` flag: the root isn't authoritative for `google.com`

### Step 2: ask a `.com` server

```bash
dig @192.5.6.30 www.google.com A +norec
```

- **AUTHORITY**: `google.com. NS ns1.google.com.` … `ns4.google.com.`
- **ADDITIONAL**: `ns1.google.com. A 216.239.32.10` (glue)

### Step 3: ask Google's own name server

```bash
dig @216.239.32.10 www.google.com A +norec
```

- **ANSWER**: `www.google.com. 300 IN A 142.250.x.x`
- flags include **`aa`**: this is the authoritative source. **Done.**

### Step 4: compare with the automatic version

```bash
dig +trace www.google.com
```

`+trace` does exactly your steps 1 to 3 and prints each hop.

### Exercises to go further

1. **Other record types, same path**: redo steps 1–3 with `MX google.com`, `TXT google.com`,
   `AAAA www.google.com`. The referrals are identical; only the final answer changes.

2. **A CNAME chain**: resolve `www.microsoft.com` by hand. At the end you get a **CNAME**
   (an alias to a CDN domain) instead of an IP. A resolver must then **start again from the root**
   for the new name. Follow the whole chain until you reach an `A` record and count the hops.

3. **Glueless delegation**: if a referral's **ADDITIONAL** section is empty, you have name server
   *names* but no IPs. You must first resolve the name server's name (from the root!) before
   continuing. Find such a case (try domains in different TLDs, e.g. `.fr`, `.org`, `.io`).

4. **TTLs**: note the TTL at each level (root → TLD → domain). Why are the upper levels cached for
   days and `www.google.com` for minutes?

5. **The DNSSEC chain by hand**: `google.com` is not signed, so use `cloudflare.com`:
   ```bash
   dig @198.41.0.4 com DS +norec +dnssec          # the root vouches for .com
   dig @192.5.6.30 cloudflare.com DS +norec +dnssec   # .com vouches for cloudflare.com
   ```
   Each parent publishes a **DS** (hash of the child's key) and signs it (**RRSIG**). That's the
   chain of trust.

6. **Over TCP**: add `+tcp` to each step. Same answers, different transport.

7. **Reverse DNS by hand**: `dig -x 8.8.8.8 +trace`, then do it step by step: the name is
   `8.8.8.8.in-addr.arpa`.

### Challenge: write your own mini recursive resolver

Simple Python with `dnspython` (`apt-get install -y python3-dnspython`). It follows referrals,
handles glueless delegations and CNAMEs:

```python
# miniresolver.py  usage: python3 miniresolver.py www.microsoft.com
import sys
import dns.flags, dns.message, dns.query, dns.rdatatype

ROOT = "198.41.0.4"   # a.root-servers.net

def resolve(name, server=ROOT, depth=0):
    indent = "  " * depth
    query = dns.message.make_query(name, "A")
    query.flags &= ~dns.flags.RD                 # +norec
    resp = dns.query.udp(query, server, timeout=3)
    print(f"{indent}{name} @ {server}")

    # 1. Final answer?
    if resp.answer:
        last = resp.answer[-1]
        for rrset in resp.answer:
            print(f"{indent}  ANSWER {rrset}")
        if last.rdtype == dns.rdatatype.CNAME:   # alias: start again from the root
            return resolve(str(last[0].target), ROOT, depth + 1)
        return [rr.to_text() for rr in last]

    # 2. Referral?
    ns_names = [str(rr.target) for rrset in resp.authority
                if rrset.rdtype == dns.rdatatype.NS for rr in rrset]
    if not ns_names:
        print(f"{indent}  no answer (NXDOMAIN or no data)")
        return []
    print(f"{indent}  referral to {ns_names[0]}")

    glue = [rr.to_text() for rrset in resp.additional
            if rrset.rdtype == dns.rdatatype.A for rr in rrset]
    if glue:
        return resolve(name, glue[0], depth + 1)

    # 3. Glueless: resolve the name server's own name first
    print(f"{indent}  no glue, resolving {ns_names[0]} first")
    ns_ips = resolve(ns_names[0], ROOT, depth + 1)
    return resolve(name, ns_ips[0], depth + 1) if ns_ips else []

print(resolve(sys.argv[1]))
```

Ideas to improve it: try the next server when one times out, add a cache, support `AAAA`, print
the time spent per hop.

---

## Part 6: let Unbound do it (mode C)

With the tunnel up, full recursion from the root works from anywhere. Inside the VM, use the
recursive config from `labo-dns-vm-isolee.md` §6:

```bash
rm -f /etc/unbound/unbound.conf.d/dot.conf /etc/unbound/unbound.conf.d/doh.conf
cat > /etc/unbound/unbound.conf.d/recursif.conf <<'EOF'
server:
    interface: 127.0.0.1
    access-control: 127.0.0.0/8 allow
    log-queries: yes
    log-replies: yes
EOF
unbound-checkconf && systemctl restart unbound

dig @127.0.0.1 www.google.com
dig @127.0.0.1 dnssec-failed.org        # must be SERVFAIL (DNSSEC validation works)
dig @127.0.0.1 whoami.akamai.net +short # must be ORACLE_IP: YOU are the resolver now
```

Watch it work: `journalctl -u unbound -f` in one terminal, queries in another.
Run the same query twice and compare `Query time` (the second one comes from the cache).

---

## Part 7: network and security tools tour

All in the VM. Install what's missing:

```bash
apt-get install -y bind9-host bind9-dnsutils knot-dnsutils whois mtr-tiny traceroute \
  nmap netcat-openbsd curl openssl tcpdump tshark iproute2
```

> ⚖️ **Rule for scanning tools**: only scan **your own machines** (your VM, your Oracle server)
> and **scanme.nmap.org**, which the Nmap project explicitly allows for light testing. Scanning
> other people's systems without permission can be illegal.

### DNS

```bash
# host: short and readable (your tutor's favourite)
host www.google.com
host -t MX gmail.com
host -t NS orange.com
host 8.8.8.8                       # reverse lookup
host -v www.orange.com             # verbose: shows dig-like sections

# dig: the precise tool
dig orange.com ANY +noall +answer
dig orange.com SOA +multiline      # zone serial, refresh, expire...
dig +short TXT orange.com          # SPF, domain verifications...

# kdig: same idea, plus DoT/DoH
kdig @1.1.1.1 +tls-ca +tls-hostname=cloudflare-dns.com example.com

# zone transfer (AXFR) on a zone made public FOR TRAINING
dig NS zonetransfer.me +short
dig AXFR zonetransfer.me @nsztm1.digi.ninja
```

The AXFR is a classic misconfiguration: a name server that hands its **whole zone** to anyone. Ask
yourself how DSecO could detect this, and how to represent it in the ontology.

### Who owns what

```bash
whois orange.com | head -40               # domain: registrar, dates
whois 8.8.8.8 | grep -iE "orgname|netname|cidr"
whois -h whois.radb.net 8.8.8.8           # origin AS, links to the AutonomousSystem class in DSecO
```

### Paths and connectivity

```bash
ip -br addr ; ip route                     # your interfaces and routes (see wg0!)
ss -tulpn                                  # what's listening on this machine
ping -c 3 1.1.1.1
traceroute 1.1.1.1                         # hop by hop
mtr -rwc 10 1.1.1.1                        # traceroute + ping statistics
tracepath 1.1.1.1                          # also discovers the MTU
nc -zv -w3 1.1.1.1 853                     # is a TCP port reachable?
```

Run `traceroute 1.1.1.1` with the tunnel up, then down: the first hops change completely.

### HTTP and TLS (useful for the HTTPO companion ontology)

```bash
curl -sI https://www.orange.com            # headers: server, redirections, security headers
curl -sv https://www.orange.com -o /dev/null 2>&1 | grep -iE "subject|issuer|expire|SSL connection"

# the certificate in detail
openssl s_client -connect www.orange.com:443 -servername www.orange.com </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```

The `subjectAltName` lists every FQDN the certificate covers, which gives you another FQDN ↔
service relationship to model.

### nmap

```bash
# your own Oracle server, from the VM: which ports does the internet see?
nmap -Pn ORACLE_IP                         # TCP: only 22 should be open
sudo nmap -Pn -sU -p 51820,443 ORACLE_IP   # UDP: WireGuard often shows "open|filtered" (it stays silent by design)

# service and version detection on the official playground
nmap -sV scanme.nmap.org

# your own Unbound
nmap -sU -p 53 --script dns-recursion 127.0.0.1
```

### Capture and analyse

```bash
tcpdump -ni wg0 -w recursion.pcap port 53 &
dig @198.41.0.4 www.google.com +norec
dig @192.5.6.30 www.google.com +norec
dig @216.239.32.10 www.google.com +norec
kill %1

tshark -r recursion.pcap -T fields -e ip.dst -e dns.qry.name -e dns.flags.authoritative -e dns.a
```

You see your manual recursion as raw data. This is exactly what the `pcap2ttl.py` script in the
Incus guide (§6.7) can turn into RDF triples.

---

## 8. When the tunnel is blocked

Some networks block unusual UDP ports. Try, in order:

### 8.1 WireGuard on UDP 443

UDP 443 is used by QUIC/HTTP3 (YouTube, Google…), so it's rarely blocked.

On the server, in `/etc/wireguard/wg0.conf`: `ListenPort = 443`, then
`sudo systemctl restart wg-quick@wg0`. You already opened UDP 443 in 1.4 and 2.3.

In the VM, in `/etc/wireguard/wg0.conf`: `Endpoint = ORACLE_IP:443`, then
`wg-quick down wg0 && wg-quick up wg0`.

### 8.2 Only TCP 443 gets out (very strict networks)

WireGuard only speaks UDP, so it must be wrapped in a TCP/WebSocket connection on port 443, for
example with **wstunnel** (github.com/erebe/wstunnel). This is a more advanced setup. The main
pitfall is that the wstunnel connection itself must not be routed into the tunnel (you need an
explicit route to `ORACLE_IP` via the VM's normal gateway). Do it as a challenge once everything
else works.

### 8.3 Quick diagnosis

```bash
wg show                    # "latest handshake" missing or old → packets don't get through
tcpdump -ni enp5s0 udp port 51820    # do packets leave? do any come back?
```

No reply packets usually means one of three things: the cloud Security List (1.4), Oracle's
internal iptables (2.3), or the local network blocking that port.

---

## 9. Cleanup and troubleshooting

### Reset the VM

```bash
incus snapshot restore dnsvm tunnel-ok      # tunnel working, back to a known state
incus snapshot restore dnsvm propre         # brand new VM
```

### Stop the tunnel without deleting anything

```bash
incus exec dnsvm -- wg-quick down wg0
incus exec dnsvm -- systemctl disable wg-quick@wg0
```

### Troubleshooting

| Symptom | Likely cause |
|---|---|
| `wg show`: no handshake | Security List (1.4) or `iptables` on the server (2.3), or wrong public keys |
| Handshake OK but no internet in the VM | `ip_forward` (2.2), wrong interface name in `PostUp` (`ens3`?), or missing MASQUERADE |
| Small pings work, `apt`/`curl` stall | MTU: lower `MTU` to 1280 in the VM's `wg0.conf` |
| VM has no network at all after `wg-quick up` | `wg-quick down wg0`, check the `Endpoint` IP, try again |
| `ssh` to Oracle hangs | your local network blocks port 22: connect from the VM through the tunnel, or use the Oracle web console (Cloud Shell) |
| Oracle instance stopped by itself | Oracle may reclaim Always Free instances it considers idle; restart it from the console |

### Resources

- WireGuard quick start: <https://www.wireguard.com/quickstart/>
- Oracle Always Free: <https://www.oracle.com/cloud/free/>
- Root servers: <https://root-servers.org/>
- dnspython: <https://dnspython.readthedocs.io/>
- Nmap reference: <https://nmap.org/book/man.html>
- zonetransfer.me (deliberately misconfigured zone for training): <https://digi.ninja/projects/zonetransferme.php>

**Suggested order**: Part 1 → 2 → 3 → 4 (proof) → **Part 5 (the core exercise)** → Part 6 →
Part 7 at your own pace.
