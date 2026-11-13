# Hardened isolated lab VM, built from scratch

> **Goal**: a VM where you can do *anything*, even leave a port open or break things, while
> nothing can climb back up to your host and its sensitive data.
>
> This guide replaces §1 of `labo-dns-vm-isolee.md`. Once the VM is built and checked, continue
> with `dns-tunnel-recursion-lab.md` (tunnel + recursion). §7 here adds the security
> parts of the Oracle server.

---

## Contents

0. [Threat model: what protects you, honestly](#0-threat-model-what-protects-you-honestly)
1. [Start clean](#1-start-clean)
2. [Prepare the host (once)](#2-prepare-the-host-once)
3. [Internet-only network](#3-internet-only-network)
4. [Hardened profile and VM](#4-hardened-profile-and-vm)
5. [Check isolation from the host](#5-check-isolation-from-the-host)
6. [Inside the VM: firewall and minimal surface](#6-inside-the-vm-firewall-and-minimal-surface)
7. [Hardened Oracle server and tunnel](#7-hardened-oracle-server-and-tunnel)
8. [Final test matrix](#8-final-test-matrix)
9. [Rules to keep it safe over time](#9-rules-to-keep-it-safe-over-time)
10. [Undo everything](#10-undo-everything)

---

## 0. Threat model: what protects you, honestly

**Scenario**: an attacker gets root inside the VM (through a port you left open, a malicious
package, the Oracle server being taken over…). What stops them from reaching your host?

```
 attacker (root in the VM)
   │
   ├─▶ Layer 1  Hypervisor (KVM/QEMU)          separate kernel, virtual hardware
   │            + QEMU runs unprivileged, confined by AppArmor + seccomp
   ├─▶ Layer 2  Nothing shared                  no folders, no devices, no host API on the network
   ├─▶ Layer 3  Internet-only network (ACL)     host IPs and LAN unreachable from the VM
   └─▶ Layer 4  Host hygiene                    updated kernel/Incus/QEMU, no needless listening services

 attacker (controls the Oracle server)
   └─▶ Layer 5  Firewall inside the VM          nothing new comes in through the tunnel
               (then layers 1–4 still stand between them and your host)
```

**Honest limits**

- No isolation is absolute. The only way through layer 1 is a **VM escape**: a bug in QEMU/KVM.
  These exist but are rare and very valuable. Your protection is to **keep the host updated**.
- **Anything you put inside the VM is exposed** if the VM is compromised. Never copy sensitive
  files, tokens or Orange credentials into it.
- The host keeps two tiny services reachable from the VM **by design**: DHCP (and DNS) from Incus's
  `dnsmasq`, otherwise the VM would get no IP. That surface is small and maintained.

> ⚠️ **Update Incus first.** You're on **6.0.0**. A vulnerability (CVE-2025-52889) was found in how
> Incus builds firewall rules for DHCP/DNS when ACLs are used on a bridge: they could partly bypass
> the `security.mac_filtering` / `ipv4_filtering` options. That's exactly the setup used here.
> Move to a current version (the Zabbly repository, see the Incus guide) before relying on this
> guide.

---

## 1. Start clean

If you created things from the earlier guides, remove them so nothing half-configured remains:

```bash
incus delete --force dnsvm 2>/dev/null
incus network delete dnsiso 2>/dev/null
incus network acl delete no-lan 2>/dev/null
incus profile delete isolated 2>/dev/null
```

---

## 2. Prepare the host (once)

### 2.1 Update everything (your layer 1 and 4)

```bash
sudo apt update && sudo apt full-upgrade -y
incus version          # must be recent, not 6.0.0
```

Reboot if the kernel was updated.

### 2.2 The Incus API must NOT listen on the network

During `incus admin init`, if you answered *yes* to "available over the network", the Incus API
listens on a TCP port. Anything able to reach it could try to control Incus.

```bash
incus config get core.https_address
```

**Empty output = good.** If it shows something (e.g. `[::]:8443`):

```bash
incus config unset core.https_address
```

### 2.3 What does your host expose?

```bash
sudo ss -tulpn | grep -vE '127\.0\.0\.|\[::1\]'
```

Every line is a service reachable from outside the host itself. On your machine you'll probably
see **k3s** (ports `6443`, `10250`…, and Traefik on `80`/`443`), maybe `dnsmasq` (Incus), `sshd`,
`cupsd`… The ACL in §3 makes them unreachable from the VM, but stop what you don't use:

```bash
sudo systemctl disable --now k3s     # when you're not doing the Kubernetes exercises
```

---

## 3. Internet-only network

### 3.1 Dedicated bridge

```bash
incus network create dnsiso \
  ipv4.address=10.70.0.1/24 ipv4.nat=true \
  ipv6.address=none
```

> If you had to add iptables rules by hand for `incusbr0` (the DHCP problem), redo them for
> `dnsiso`.

### 3.2 The ACL: everything private is forbidden, only the internet is allowed

Incus ACL rules are not applied in the order you write them, but by action: **`drop` first, then
`reject`, then `allow`**, then the default action. So "drop all private ranges + allow the rest"
does exactly what we want.

```bash
incus network acl create internet-only

# Forbidden destinations: your host's IPs, your LAN, Docker/k3s, VPNs, link-local
for net in 10.0.0.0/8 172.16.0.0/12 192.168.0.0/16 100.64.0.0/10 169.254.0.0/16; do
  incus network acl rule add internet-only egress action=drop destination=$net
done

# Everything else (= the public internet) is allowed
incus network acl rule add internet-only egress action=allow
incus network acl rule add internet-only ingress action=allow

incus network set dnsiso security.acls=internet-only
incus network acl show internet-only
```

What this blocks, concretely:

| VM tries to reach | Example | Result |
|---|---|---|
| Your host through the gateway | `10.70.0.1:6443` (k3s) | **dropped** (10.0.0.0/8) |
| Your host through its Wi-Fi IP | `192.168.1.23` | **dropped** |
| Other devices of the residence | the router, printers… | **dropped** |
| Docker / k3s internal networks | `172.17.x.x`, `10.42.x.x` | **dropped** |
| Orange VPN networks | usually `10.x` | **dropped** |
| The internet, your Oracle server | `1.1.1.1`, `ORACLE_IP` | allowed |

Bridge ACLs apply at the boundary between the bridge and the host, which is exactly where the VM
would try to climb up. DHCP still works: the request is a broadcast, and Incus automatically
allows its own DHCP/DNS services.

> **Ingress** stays `allow` here so DHCP replies and tunnel traffic come back without surprises.
> Protecting the VM from unwanted inbound traffic is the job of the firewall inside the VM (§6).

---

## 4. Hardened profile and VM

### 4.1 A dedicated profile

When you launch with `--profile isolated`, **only** this profile applies: the `default` profile
and anything it might gain in the future are ignored.

```bash
incus profile create isolated
incus profile edit isolated <<'EOF'
description: Hardened isolated VM - internet only, nothing shared with the host
config:
  limits.cpu: "2"
  limits.memory: 2GiB
  security.secureboot: "true"
devices:
  root:
    type: disk
    path: /
    pool: default
    size: 12GiB
  eth0:
    type: nic
    network: dnsiso
    security.mac_filtering: "true"
    security.ipv4_filtering: "true"
EOF
incus profile show isolated
```

- `security.mac_filtering` / `security.ipv4_filtering`: the VM can't fake another MAC or IP
  address on the bridge.
- `security.secureboot`: the VM only boots signed bootloaders and kernels.
- **Only two devices**: the root disk and the network card. No shared folder, no USB, no GPU.

Optional: reduce what the VM can ask Incus about itself (the "guest API"):

```bash
incus profile set isolated security.guestapi=false
```

If your version answers "unknown key", skip it. It's a small extra, not essential.

### 4.2 Create the VM

```bash
incus launch images:debian/13 dnsvm --vm --profile isolated

until incus exec dnsvm -- true 2>/dev/null; do sleep 3; done; echo "VM ready"
incus list dnsvm          # must have an IPv4 in 10.70.0.x
```

No IPv4? See the troubleshooting table at the end of §8.

---

## 5. Check isolation from the host

```bash
# Only "root" and "eth0", nothing else
incus config show dnsvm --expanded | sed -n '/devices:/,$p'

# Nothing dangerous in the config
incus config show dnsvm --expanded | grep -iE "raw\.|privileged|disk|usb|gpu|pci" \
  || echo "OK: nothing found"

# Incus API not on the network
incus config get core.https_address      # empty
```

Snapshot of the clean, verified VM:

```bash
incus snapshot create dnsvm clean
```

---

## 6. Inside the VM: firewall and minimal surface

```bash
incus shell dnsvm
```

### 6.1 Updates and tools

```bash
apt-get update && apt-get full-upgrade -y
apt-get install -y nftables unattended-upgrades \
  bind9-dnsutils knot-dnsutils unbound wireguard \
  netcat-openbsd curl tcpdump nmap whois ca-certificates
```

### 6.2 Firewall: nothing new comes in, from anywhere

Default policy: **drop** everything incoming, except replies to connections the VM started itself.

```bash
cat > /etc/nftables.conf <<'EOF'
#!/usr/sbin/nft -f
# Only our own table is replaced, so wg-quick's table is left alone
table inet vmfw
delete table inet vmfw

table inet vmfw {
    chain input {
        type filter hook input priority 0; policy drop;

        iif "lo" accept                                   # local services (Unbound on 127.0.0.1)
        ct state established,related accept               # replies to what WE started
        ct state invalid drop
        udp sport 67 udp dport 68 accept                  # DHCP replies
        icmp type { destination-unreachable, time-exceeded, echo-reply } accept   # traceroute, ping replies
    }

    chain forward {
        type filter hook forward priority 0; policy drop; # the VM is not a router
    }

    chain output {
        type filter hook output priority 0; policy accept;
    }
}
EOF

nft -c -f /etc/nftables.conf && echo "syntax OK"
systemctl enable --now nftables
nft list ruleset
```

What this guarantees:
- **Even if you leave a service open** (a test web server on `0.0.0.0:8080`…), nobody can connect
  to it: not the host, not the LAN, not the Oracle server through the tunnel.
- The WireGuard tunnel still works: the VM starts the connection and the server's packets are
  replies (`established`). `PersistentKeepalive` keeps that state alive.

> If you ever reload this file by hand while the tunnel is up, run
> `systemctl restart wg-quick@wg0` afterwards to be safe.

### 6.3 Check what listens

```bash
ss -tulpn
```

You should only see `127.0.0.1` addresses (Unbound) and possibly the DHCP client. If something
listens on `0.0.0.0` that you didn't install on purpose, find it and remove it.

### 6.4 Prove the VM can't reach the host or the LAN

Still in the VM:

```bash
# Internet: must work
curl -4 -s -o /dev/null -w "internet: %{http_code}\n" https://deb.debian.org

# The host through the gateway: must FAIL (timeouts)
nc -zv -w3 10.70.0.1 22
nc -zv -w3 10.70.0.1 6443
nmap -Pn -p 1-1000,6443,8443,10250 --max-retries 1 10.70.0.1    # expected: all "filtered"

# Your host's Wi-Fi IP and the residence router: must FAIL
# (replace with your values, from "ip -br addr" and "ip route" on the HOST)
nc -zv -w3 192.168.1.1 80
```

`filtered` everywhere = the ACL works.

```bash
exit
incus snapshot create dnsvm hardened
```

---

## 7. Hardened Oracle server and tunnel

Build the server and the tunnel with **`dns-tunnel-recursion-lab.md` Parts 1 to 3**, adding these
security measures.

### 7.1 Keys stay in the VM

- The SSH key for Oracle is generated **inside the VM** (Part 1.1 of the tunnel guide), never on
  your host.
- The WireGuard keys are generated where they're used (server key on the server, VM key in the VM).
  Only **public** keys are copied around.

### 7.2 Open as little as possible on Oracle

In the Security List, keep only:

| Protocol | Port | Why |
|---|---|---|
| UDP | 51820 | WireGuard |
| UDP | 443 | WireGuard fallback (only if you use it) |
| TCP | 22 | SSH, **temporary**, removed in 7.4 |

WireGuard doesn't answer at all to anyone who lacks the right key: to a scanner, the port looks dead.

### 7.3 Accept the tunnel in the VM firewall

Nothing to add: the tunnel is outgoing from the VM's point of view, so `established` covers it.
Check after `wg-quick up wg0`:

```bash
wg show        # recent "latest handshake"
```

### 7.4 Close SSH on the internet: administer the server through the tunnel

Once the tunnel works, the server is reachable at `10.99.0.1` **through** it:

```bash
# from the VM
ssh -i ~/.ssh/oracle ubuntu@10.99.0.1
```

If that works, **delete the TCP 22 rule** from the Oracle Security List. The server now exposes
nothing visible on the internet. If the tunnel breaks, Oracle's web console (**Cloud Shell** /
instance console connection) remains your backup access.

### 7.5 Server hygiene

```bash
# on the server
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
sudo sshd -T | grep -E "passwordauthentication|permitrootlogin"   # expected: no / without-password or no
```

Store nothing on this server. It's a disposable exit door.

### 7.6 If the server gets compromised, what happens?

The attacker is **in the tunnel** and can send packets to `10.99.0.2` (the VM). The VM firewall
drops every new inbound connection, so they gain nothing. And even if they got into the VM, layers
1 to 4 still separate them from your host. In that case: destroy the Oracle instance, restore the
VM from the `hardened` snapshot, generate new keys.

---

## 8. Final test matrix

| # | From | Test | Expected |
|---|---|---|---|
| 1 | host | `incus config show dnsvm --expanded` | only `root` + `eth0` |
| 2 | host | `incus config get core.https_address` | empty |
| 3 | VM | `curl -4 ifconfig.me` | ORACLE_IP (tunnel up) |
| 4 | VM | `nmap -Pn -p 1-1000,6443 10.70.0.1` | all `filtered` |
| 5 | VM | `nc -zv -w3 <host Wi-Fi IP> 22` | timeout |
| 6 | VM | `nc -zv -w3 <residence router IP> 80` | timeout |
| 7 | VM | `dig @192.0.2.1 example.com +time=3 +tries=1` | timeout (no hijacking) |
| 8 | Oracle | `nc -zv -w3 10.99.0.2 22` and `ping -c2 10.99.0.2` | no answer |
| 9 | outside | `nmap -Pn ORACLE_IP` (from the VM with the tunnel **down**) | no open TCP port (after 7.4) |
| 10 | VM | `python3 -m http.server 8000 &`, then from the host `curl -m3 http://<VM IP>:8000` | timeout (VM firewall) |

Test 10 simulates exactly your fear, "I left a port open by mistake": nobody can use it.
Stop it afterwards with `kill %1` in the VM.

### Troubleshooting

| Symptom | Cause / fix |
|---|---|
| VM has no IPv4 | host firewall blocks DHCP on `dnsiso` (redo your iptables fix for this bridge); otherwise test without the ACL: `incus network unset dnsiso security.acls`, then restart the VM |
| No internet in the VM | ACL default: check `incus network acl show internet-only` contains the `egress action=allow` rule |
| Test 4 shows `open` ports | ACL not attached: `incus network get dnsiso security.acls` |
| Tunnel does not come up after the firewall | `nft list ruleset` must still contain the `wg-quick` table; `systemctl restart wg-quick@wg0` |
| `incus exec` stopped working | normal network changes never affect it; it's the VM agent, see the Incus guide |

---

## 9. Rules to keep it safe over time

1. **Never** add a `disk` device (shared folder) to this VM. Need a file inside? Use
   `incus file push file dnsvm/root/` for that one file.
2. **Never** put secrets in it: no Orange tokens, no personal SSH keys, no API keys.
3. **Don't** use `incus file mount` on this VM (that exposes the VM's files on your host).
4. **Update** the host (kernel, Incus, QEMU) regularly: it's your main protection against escapes.
5. **Snapshot before experiments**, and if in doubt **rebuild instead of repairing**:
   `incus snapshot restore dnsvm hardened`.
6. Before changing the `isolated` profile or the `internet-only` ACL, ask yourself: "does this
   open a path toward the host?"
7. Re-run the §8 test matrix after any big change.

---

## 10. Undo everything

```bash
incus delete --force dnsvm
incus profile delete isolated
incus network delete dnsiso
incus network acl delete internet-only
```

On Oracle: **Terminate** the instance (and its boot volume) from the console.
Your host is back exactly as it was.

### References

- Incus network ACLs (rule order, bridge limitations): <https://linuxcontainers.org/incus/docs/main/howto/network_acls/>
- Incus security overview: <https://linuxcontainers.org/incus/docs/main/explanation/security/>
- nftables wiki: <https://wiki.nftables.org/>
- WireGuard: <https://www.wireguard.com/>
