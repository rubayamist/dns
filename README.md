# Local Web Server + Custom DNS Lab (Windows CLI + VirtualBox)

A hands-on networking lab. You will:

1. Host a webpage from `C:\mywebsite` on your Windows machine.
2. Open it from a **second computer or a VirtualBox VM** by typing only the IP address.
3. Map a **custom hostname** to that IP using the Windows `hosts` file, and verify it from the command line.

```
Client (VM / other PC)                Server (Windows host)
+---------------------+               +----------------------+
|  browser            |  HTTP GET /   |  python -m http.server|
|  http://192.168.0.111 ------------> |  port 80 or 8000      |
|                     | <------------ |  C:\mywebsite\index.html
+---------------------+   200 OK      +----------------------+
```

---

## Table of Contents

- [Part 0 — Requirements](#part-0--requirements)
- [Part 1 — Build and serve the page](#part-1--build-and-serve-the-page)
- [Part 2 — Open the firewall](#part-2--open-the-firewall)
- [Part 3 — Reach it from a VirtualBox VM](#part-3--reach-it-from-a-virtualbox-vm)
- [Part 4 — Reach it from another physical PC](#part-4--reach-it-from-another-physical-pc)
- [Part 5 — Custom DNS with the hosts file](#part-5--custom-dns-with-the-hosts-file)
- [Part 6 — DNS command cheat sheet](#part-6--dns-command-cheat-sheet)
- [Troubleshooting](#troubleshooting)
- [Cleanup](#cleanup)

---

## Part 0 — Requirements

| Item | Notes |
|---|---|
| Windows 10/11 | Acts as the **server** |
| Python 3.x | `python --version` must work in CMD |
| Oracle VirtualBox | Any guest OS with a browser |
| Admin rights | Needed for the firewall rule, port 80, and the hosts file |

Find the server's IP address:

```cmd
ipconfig
```

```text
Wireless LAN adapter Wi-Fi:
   IPv4 Address. . . . . . . . . . . : 192.168.0.111
   Default Gateway . . . . . . . . . : 192.168.0.1
```

Write down that IPv4 address. Everywhere this README says `192.168.0.111`, substitute yours.

---

## Part 1 — Build and serve the page

### 1.1 Create the folder and page

```cmd
mkdir C:\mywebsite
cd C:\mywebsite
notepad index.html
```

Paste, then save:

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>My Local Website</title>
</head>
<body>
    <h1>Welcome to My Local Web Server</h1>
    <p>This page is served from a Windows machine on the LAN.</p>
    <p>Server IP: 192.168.0.111</p>
</body>
</html>
```

### 1.2 Start the server

Bind to `0.0.0.0` so the server accepts connections from **other machines**, not just from itself. This is the single most-missed step.

```cmd
cd C:\mywebsite
python -m http.server 8000 --bind 0.0.0.0
```

```text
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

Leave this window open. Closing it stops the server.

> `--bind 127.0.0.1` (or omitting the bind on some setups) makes the server reachable **only from the same machine**. Remote clients get "connection refused."

### 1.3 Typing only the IP, with no port

A browser assumes port **80** when you type a bare IP. To get `http://192.168.0.111` with nothing after it, serve on port 80 — in an **Administrator** CMD, since ports below 1024 are privileged:

```cmd
cd C:\mywebsite
python -m http.server 80 --bind 0.0.0.0
```

Check first that nothing already owns port 80:

```cmd
netstat -ano | findstr :80
```

If a line shows `0.0.0.0:80 ... LISTENING`, something (IIS, `http.sys`, Skype, a printer service) has it. Identify the owner:

```cmd
tasklist /fi "pid eq 4"
```

Simplest fix: stay on **8000** and type `http://192.168.0.111:8000`. The lab works identically; you just include the port.

### 1.4 Local smoke test

```cmd
curl http://127.0.0.1:8000
```

HTML comes back → the server works. Any failure at this point is a Python/folder problem, not a network problem.

---

## Part 2 — Open the firewall

Windows Defender blocks inbound TCP by default. Without this, the local test passes and every remote test fails.

Run in an **Administrator** CMD:

```cmd
netsh advfirewall firewall add rule name="Local Web Server 8000" dir=in action=allow protocol=TCP localport=8000
```

If you used port 80:

```cmd
netsh advfirewall firewall add rule name="Local Web Server 80" dir=in action=allow protocol=TCP localport=80
```

Also allow ping, so `ping` is a meaningful diagnostic:

```cmd
netsh advfirewall firewall add rule name="Allow ICMPv4-In" protocol=icmpv4:8,any dir=in action=allow
```

Verify:

```cmd
netsh advfirewall firewall show rule name="Local Web Server 8000"
```

Your network profile should be **Private**. On a Public profile Windows hardens sharing rules:

```cmd
powershell -Command "Get-NetConnectionProfile"
powershell -Command "Set-NetConnectionProfile -InterfaceAlias 'Wi-Fi' -NetworkCategory Private"
```

---

## Part 3 — Reach it from a VirtualBox VM

The VM's network mode decides whether it can see your host's LAN IP at all. Default NAT **cannot** — that is why this usually fails on the first try.

| Mode | VM can reach host's LAN IP? | Use for this lab |
|---|---|---|
| NAT (default) | No — host is `10.0.2.2` only | ✗ |
| **Bridged Adapter** | Yes — VM joins your real LAN | ✓ recommended |
| **Host-only Adapter** | Yes — via `192.168.56.1` | ✓ fallback |
| Internal Network | No — VM-to-VM only | ✗ |

### Option A — Bridged Adapter (recommended)

1. Shut the VM down (full power off, not saved state).
2. **Settings → Network → Adapter 1**
3. *Enable Network Adapter*: checked
4. *Attached to*: **Bridged Adapter**
5. *Name*: the host adapter you actually use — your Wi-Fi or Ethernet card
6. **Advanced → Promiscuous Mode: Allow All** (often required over Wi-Fi)
7. OK, start the VM.

Inside the VM, confirm it landed on the same subnet:

```cmd
ipconfig
```

```text
IPv4 Address. . . . : 192.168.0.157      <-- same 192.168.0.x family as the host
```

Then test:

```cmd
ping 192.168.0.111
curl http://192.168.0.111:8000
```

Open the browser in the VM:

```text
http://192.168.0.111:8000
```

Linux guest equivalents: `ip addr` and `curl -v http://192.168.0.111:8000`.

### Option B — Host-only Adapter (when bridged Wi-Fi is blocked)

Many campus and hotel access points use *client isolation*, which silently kills bridged Wi-Fi. Host-only sidesteps the physical network entirely.

1. VirtualBox → **Tools → Network → Host-only Networks → Create** (gives you `192.168.56.0/24`, DHCP on).
2. VM **Settings → Network → Adapter 2** → Enable → *Attached to*: **Host-only Adapter** → pick that network.
   Keep Adapter 1 on NAT so the VM keeps internet access.
3. Start the VM.

On the **host**, find the host-only address:

```cmd
ipconfig
```

```text
Ethernet adapter VirtualBox Host-Only Network:
   IPv4 Address. . . . : 192.168.56.1
```

The Python server is already bound to `0.0.0.0`, so it listens on this interface too. From the VM:

```text
http://192.168.56.1:8000
```

Add a firewall rule scoped to that subnet if the general rule is too tight:

```cmd
netsh advfirewall firewall add rule name="HostOnly Web 8000" dir=in action=allow protocol=TCP localport=8000 remoteip=192.168.56.0/24
```

### Watch the traffic

Every request the VM makes prints in the server's CMD window:

```text
192.168.0.157 - - [17/Sep/2026 14:22:41] "GET / HTTP/1.1" 200 -
```

That line is proof the client reached you — a useful screenshot for a lab report.

---

## Part 4 — Reach it from another physical PC

Both machines must be on the same LAN (same Wi-Fi/router).

On the **client**:

```cmd
ping 192.168.0.111
```

| Result | Meaning |
|---|---|
| Replies | Layer 3 is fine → go to the browser |
| Request timed out | Different subnet, or ICMP blocked — check `ipconfig` on both |
| Destination host unreachable | Not on the same network at all |

Then browse to `http://192.168.0.111:8000`.

Ping succeeding but the browser hanging almost always means the **firewall rule from Part 2 is missing**, or the server was not bound to `0.0.0.0`.

---

## Part 5 — Custom DNS with the hosts file

Goal: type `http://mysite.local:8000` instead of the raw IP. The `hosts` file is a local name-to-IP table consulted **before** any DNS server.

```text
C:\Windows\System32\drivers\etc\hosts
```

Edit it on whichever machine you want the name to work on. The name is local to that machine — repeat on the client if you want the client to use it.

### 5.1 Edit it

GUI route: right-click Notepad → **Run as administrator** → File → Open → paste the path → change *Text Documents* to **All Files**.

CLI route, in an **Administrator** CMD:

```cmd
notepad C:\Windows\System32\drivers\etc\hosts
```

Append:

```text
192.168.0.111    mysite.local
192.168.0.111    server1
192.168.1.20     computer2
```

Format: IP, whitespace, then one or more names. `#` starts a comment. Save.

One-liner append (Administrator CMD only — a normal prompt gets "Access is denied"):

```cmd
echo 192.168.0.111    mysite.local>> C:\Windows\System32\drivers\etc\hosts
```

Back it up first:

```cmd
copy C:\Windows\System32\drivers\etc\hosts C:\Windows\System32\drivers\etc\hosts.bak
```

### 5.2 Apply and verify

```cmd
ipconfig /flushdns
ping mysite.local
```

```text
Pinging mysite.local [192.168.0.111] with 32 bytes of data:
Reply from 192.168.0.111: bytes=32 time<1ms TTL=128
```

The bracketed IP is the proof the mapping took effect.

> ### `nslookup` will *not* confirm a hosts entry
>
> `nslookup` talks to your configured DNS server directly and bypasses the Windows resolver, so it never reads `hosts`:
>
> ```cmd
> nslookup mysite.local
> ```
> ```text
> *** No internal type for both IPv4 and IPv6 Addresses (A+AAAA) records available for mysite.local
> ```
>
> That output is expected and does **not** mean your entry failed. Use `ping`, `ipconfig /displaydns`, or `Resolve-DnsName` instead.

Windows loads hosts entries into the DNS client cache, so they appear here:

```cmd
ipconfig /displaydns | findstr /i "mysite"
```

PowerShell, which does honour the resolver:

```powershell
Resolve-DnsName mysite.local
```

Reverse lookup, name from IP:

```cmd
ping -a 192.168.0.111
```

Search the hosts file itself:

```cmd
findstr /i "mysite" C:\Windows\System32\drivers\etc\hosts
type C:\Windows\System32\drivers\etc\hosts | findstr /v "^#"
```

### 5.3 Use the name in the browser

On any machine whose hosts file has the entry:

```text
http://mysite.local:8000
```

The server log still shows the client's IP — naming is purely client-side resolution, and the HTTP request itself is unchanged.

### 5.4 Do it inside the VM

Windows guest: same file, same steps.

Linux guest:

```bash
sudo nano /etc/hosts
```

```text
192.168.0.111    mysite.local
```

```bash
ping mysite.local
curl http://mysite.local:8000
```

---

## Part 6 — DNS command cheat sheet

```cmd
:: --- resolution ---
ping hostname                      :: resolves via hosts, then DNS; shows the IP
ping -a 192.168.0.111              :: reverse lookup, IP -> name
ipconfig /displaydns               :: show resolver cache (includes hosts entries)
ipconfig /flushdns                 :: clear cache after editing hosts
nslookup example.com               :: query the DNS server directly (ignores hosts)
nslookup example.com 8.8.8.8       :: query a specific DNS server

:: --- hosts file ---
findstr /i "server1" C:\Windows\System32\drivers\etc\hosts
notepad C:\Windows\System32\drivers\etc\hosts      :: needs Administrator

:: --- which DNS servers am I using ---
ipconfig /all | findstr /i "DNS Servers"
netsh interface ip show dnsservers

:: --- change DNS servers (Administrator) ---
netsh interface ip set dns name="Wi-Fi" static 8.8.8.8 primary
netsh interface ip add dns name="Wi-Fi" 8.8.4.4 index=2
netsh interface ip set dns name="Wi-Fi" dhcp          :: revert to automatic

:: --- connectivity ---
netstat -ano | findstr :8000       :: is the server listening?
curl http://192.168.0.111:8000     :: fetch without a browser
tracert 192.168.0.111              :: path to the host
arp -a                             :: IP-to-MAC table for the local subnet
```

PowerShell extras:

```powershell
Resolve-DnsName mysite.local
Test-NetConnection 192.168.0.111 -Port 8000     # the best single remote-access test
Get-NetFirewallRule -DisplayName "Local Web Server 8000"
```

`Test-NetConnection` returning `TcpTestSucceeded : True` means routing *and* firewall are both clear.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Works on the server, refused elsewhere | Bound to localhost | Restart with `--bind 0.0.0.0` |
| Ping works, browser times out | Firewall | Add the rule in Part 2 |
| VM can't ping the host at all | VM is on NAT | Switch to Bridged or Host-only (Part 3) |
| `Permission denied` on port 80 | Non-admin shell | Run CMD as Administrator, or use 8000 |
| `Address already in use` | Port taken | `netstat -ano \| findstr :8000`, then `taskkill /PID <pid> /F` |
| Hostname won't resolve | Stale cache | `ipconfig /flushdns` |
| `hosts` won't save | Not elevated | Run Notepad as Administrator |
| Saved as `hosts.txt` | Notepad added an extension | Set *Save as type* to **All Files** |
| `nslookup` can't find the name | Expected — it skips `hosts` | Use `ping` or `Resolve-DnsName` |
| Bridged Wi-Fi gets no IP | AP client isolation | Use Host-only (Option B) |
| Browser shows a directory listing | No `index.html` | Confirm the filename and that you `cd`'d into `C:\mywebsite` |
| Edits don't appear | Browser cache | Ctrl + F5 |

---

## Cleanup

```cmd
:: stop the server
Ctrl + C in the server window

:: remove firewall rules (Administrator)
netsh advfirewall firewall delete rule name="Local Web Server 8000"
netsh advfirewall firewall delete rule name="Allow ICMPv4-In"

:: restore the hosts file
copy /y C:\Windows\System32\drivers\etc\hosts.bak C:\Windows\System32\drivers\etc\hosts
ipconfig /flushdns

:: restore automatic DNS
netsh interface ip set dns name="Wi-Fi" dhcp
```

---

## Concepts demonstrated

- **IP addressing** — locating a host on a LAN by its layer-3 address
- **Client/server model** — browser requests, HTTP server responds
- **Ports** — 80 as the implicit HTTP default; 8000 as an explicit alternative
- **Name resolution order** — `hosts` file consulted before any DNS server
- **Packet filtering** — inbound firewall rules as a reachability gate
- **Virtual networking** — NAT vs Bridged vs Host-only adapter behaviour

---

## Serving your GitHub Pages site instead

Replace the contents of `C:\mywebsite` with your site's files, keeping `index.html` at the top level:

```cmd
git clone https://github.com/<username>/<username>.github.io.git C:\mywebsite
cd C:\mywebsite
python -m http.server 8000 --bind 0.0.0.0
```

Static HTML/CSS/JS runs fine this way. If the repo is a Jekyll source (it has `_config.yml` and `_layouts/`), the raw files won't render — build it locally with `bundle exec jekyll build` and serve the generated `_site` folder instead.
