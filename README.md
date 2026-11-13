# QNAP Transmission through Proton WireGuard

Reusable manual-start kit • 13 November 2026

**Start with START-HERE.html:** open it in your browser for the shorter, color-coded instructions. This file is the detailed reference.

## What this delivers

Run a native Transmission QPKG daemon inside its own Linux network namespace, with a WireGuard default route and a small TCP relay for the LAN web UI. Other NAS applications retain their existing routing. This is for a QNAP administrator comfortable with SSH; it is not an App Center installer.

The underlying approach was demonstrated on Rob's QNAP: Transmission 4.1.3 reached an external torrent tracker as the Proton exit IP, and the web UI worked after setting its asset directory. **This consolidated kit is a new implementation, locally syntax/relay tested but not run on a QNAP.** It adds explicit namespace firewall rules, permission-aware startup, a controlled tunnel-drop probe, and guarded cleanup. QTS versions, QPKGs and kernel capabilities differ. Complete the acceptance checks on each target NAS.

No VPN credentials, private keys, torrent content, or personal public IP addresses are included. The recipient must generate their own Proton WireGuard profile with P2P support.

## Network layout

- NAS LAN IP, TCP 9091 → Python TCP relay → 172.31.191.2:9091 in namespace `qtrns`.
- Transmission → `qtrwg0` → Proton → peers/trackers.
- WireGuard is created on the NAS host and then moved into the namespace. Its encrypted transport retains the host network access required to reach Proton.
- The private web link is 172.31.191.0/30. The script refuses conflicting host routes.
- Only replies to LAN-proxy RPC connections may leave through the private link. Other IPv4 output must use WireGuard. Non-loopback IPv6 is blocked inside the namespace.
- No NAS-wide default-route change, forwarding/NAT rule, or host firewall flush is made.
- This isolates networking, not the filesystem. A privileged Transmission process can still access NAS files.

## 1. Prepare the target NAS

1. Install a compatible native Transmission QPKG from a source you trust. Verify it works normally first.
2. Set a Transmission RPC/web username and strong password. Confirm the intended download and incomplete-download folders. The kit requires existing authenticated settings.
3. Obtain a Proton WireGuard profile for a P2P-capable server. This kit supports a standard single-peer profile with an IPv4 address, IPv4 DNS, endpoint, and `AllowedIPs = 0.0.0.0/0` (an additional `::/0` in the source is fine; the kit runs IPv4 only).
4. Copy the profile to a private NAS location. It contains a private key: do not send it along with this kit. Restrict its access. A general Download share is useful for transfer, but move the profile out of a broadly shared folder for permanent use.
5. Install/verify Python 3.6+, the full `ip` command with network-namespace support, `wg`, `iptables`, `ip6tables`, and `curl`. Kernel support for WireGuard, veth, namespaces, and namespace firewall/conntrack rules is mandatory. Do not substitute guessed QTS package versions. If unsupported, stop and use a supported container/VPN deployment instead.
6. Enable SSH on the NAS and connect using Terminal or an SSH console. Run `id -u`. If it prints `0`, you already have root privileges. Otherwise, run the following command and enter your administrator password when prompted (typing is not echoed):

```sh
sudo -i
```

Run `id -u` again and confirm `0`. If sudo is unavailable or denied, obtain root access using the NAS's supported administrator method before continuing. A username called admin does not by itself establish privilege.

**All remaining SSH commands assume this same root shell.** No extra sudo prefix is needed. Elevate before changing into the kit folder: `sudo -i` may change the current directory and PATH. Check the required tools after elevation. Repeat the privilege check each time you reconnect.

Now check the tools:

```sh
id -u
command -v python3
command -v ip
command -v wg
command -v iptables
command -v ip6tables
command -v curl
ip netns list
```

The kit does not use `su`, `nohup`, `systemd`, or `wg-quick`. It discovers command paths from PATH, with common QNAP fallbacks. Start it from the same SSH environment in which those commands work.

## 2. Discover the real paths

QNAP volumes are not interchangeable. Package files, downloads and VPN profiles can all be on different CACHEDEV volumes.

```sh
readlink -f /share/Download
readlink -f /share/PLEX2
ls -ld /share/Download/ /share/PLEX2/
```

For the installed package path, if the package section is named Transmission:

```sh
/sbin/getcfg Transmission Install_Path -f /etc/config/qpkg.conf
```

Use the path actually returned. Locate the web directory containing `index.html` inside that package. On the demonstrated installation it was:

```text
/share/CACHEDEV1_DATA/.qpkg/Transmission/share/transmission/public_html
```

The demonstrated settings directory was:

```text
/share/CACHEDEV1_DATA/Transmission/config
```

And the profile was found on a different volume:

```text
/share/CACHEDEV2_DATA/Download/Transmission-US-CA-802.conf
```

These are examples from one NAS, not required paths. Do not change CACHEDEV1 to CACHEDEV2 everywhere. The data directory must be the existing directory with `settings.json`, torrent files and resume state. Confirm it from the normal daemon's startup log or process command line.

## 3. Choose a persistent kit directory and configure it

Extract the ZIP and copy the entire `qnap-transmission-vpn` folder onto a persistent data volume, for example `/share/CACHEDEV1_DATA/qnap-transmission-vpn`. Do not install under `/tmp`.

From SSH, enter that actual directory:

```sh
cd /share/CACHEDEV1_DATA/qnap-transmission-vpn
chmod 700 .
cp config.example.json config.json
chmod 600 config.json
```

Edit `config.json` with a text editor. It is JSON, not shell syntax: no comments or trailing commas.

| Field | What to set |
|---|---|
| `binary` | Full path to the installed `transmission-daemon` executable |
| `data` | Existing settings/resume directory; not the downloads directory |
| `web_home` | Folder containing the installed web UI's `index.html` |
| `wireguard_config` | Full path to the recipient's own Proton `.conf` |
| `lan_ip` | This NAS's actual LAN IPv4 address; reserve it in DHCP |
| `web_port` | Available TCP port on that LAN address, normally 9091 |
| `run_as` | Existing account under which Transmission should run |
| `allow_root` | Leave false for a non-root account |
| `mtu` | 1420 initially; change only if your VPN/network needs it |

The default `transmission` account must exist. Check with `id transmission`. It needs read access to the executable/web files, write access to its settings directory, and appropriate QNAP share/ACL access to downloads and incomplete folders. The kit does not recursively chown shares or alter ACLs.

If no suitable service account exists, create one using your NAS's supported account tools and assign only the required access. Do not invent a UID/GID. As an explicit compatibility option, an administrator can set `run_as` to the existing UID-0 account and `allow_root` to true, reproducing the earlier privileged launch. That grants the daemon broad NAS filesystem access; the namespace is not a filesystem sandbox.

## 4. Stop normal Transmission before starting this kit

Stop and disable the normal Transmission QPKG in App Center. Verify the existing daemon exits. If migrating from the earlier chat installer, stop that setup too using its own stop script:

```sh
/bin/sh /share/CACHEDEV1_DATA/.transmission-proton-ns/stop.sh
```

That command applies only to an existing installation at that exact path. The new kit uses different network-object names and will not delete the old setup for you.

```sh
pidof transmission-daemon
```

No PID should be returned. The new launcher refuses to start while a Transmission-named process already exists. Do not use broad `killall python3` commands; other QNAP applications use Python.

Back up the Transmission settings/resume directory using your normal backup process before migration. The kit additionally keeps the first `settings.json` as `settings.original.json`. Subsequent starts do not overwrite that original backup.

## 5. Start

From the kit directory:

```sh
python3 qnap-vpn.py start
```

The launcher validates paths/config/authentication, creates the namespace, installs its resolver and firewall rules, establishes an IPv4 route through WireGuard, checks external connectivity, starts Transmission, and starts the LAN relay. Python detaches the processes and redirects stdin/logs, so `nohup` is unnecessary.

Startup changes these settings while Transmission is stopped: RPC enabled on namespace port 9091, bind address 0.0.0.0, RPC path `/transmission/`, IP whitelist limited to loopback and the private relay address, host whitelist enabled for the LAN address/localhost, explicit announce-IP disabled, and automatic peer-port forwarding disabled. Existing username, password, peer port and download paths are retained. The relay preserves HTTP bytes and authentication; it does not add encryption.

Open the printed URL:

```text
http://YOUR-NAS-LAN-IP:9091/transmission/web/
```

Use the existing Transmission credentials. A different configured web port changes only the host URL; namespace RPC remains on 9091. The relay binds to the configured LAN address, so `127.0.0.1:9091` is not the host test URL in this kit.

## 6. Acceptance tests — required on the recipient's NAS

### A. Process, route and tunnel checks

```sh
python3 qnap-vpn.py status
ip netns exec qtrns curl -4fsS --noproxy '*' --max-time 20 https://api.ipify.org
```

Check that every Transmission PID has the same network namespace ID as `Expected`, the default route uses `qtrwg0`, and WireGuard has a recent handshake after traffic. Compare the reported IP with the recipient's actual home ISP IP. A namespace curl test alone does not establish the torrent daemon's external identity.

### B. External torrent test

Use https://torguard.net/check-my-torrent-ip-address/ and add its test torrent/magnet through this kit's LAN UI. Confirm that the external result identifies Transmission and reports the VPN exit IP, not the ISP IP. TorGuard may return `Tracker returned an error: IP: ...`; that is how its diagnostic address appears in the client. The fake torrent will not complete.

Alternative: https://ipleak.net/ → Torrent Address detection. Use a fresh test and keep its originating browser session open. No result is inconclusive. Look at Inspector → Trackers for the specific announce error. Reaching the IPLeak website with HTTP 200 does not prove its tracker works.

### C. Controlled tunnel-drop probe

Pause ordinary torrents first. This command temporarily lowers only this namespace's WireGuard interface and restores it in a `finally` block:

```sh
python3 qnap-vpn.py drop-test
```

It first requires a successful external IPv4 baseline, then expects the same request to fail with the interface down. If terminated by a signal or power loss before restoration, recover with:

```sh
ip -n qtrns link set qtrwg0 up
```

A successful drop probe supports the intended fail-closed routing for that IPv4 request; it is not exhaustive DNS/IPv6 or hostile-process certification. Repeat the external torrent check after restoration. The firewall blocks non-loopback IPv6 and only permits DNS output through WireGuard, but resolver compatibility still needs checking on the target QTS build:

```sh
ip netns exec qtrns cat /etc/resolv.conf
ip netns exec qtrns iptables -S
ip netns exec qtrns ip6tables -S
```

The namespace resolver must match the IPv4 DNS from the Proton profile. Do not replace it with a LAN router address just to bypass a DNS error.

### D. Reboot expectations

**Autostart is deliberately not installed by this kit.** After a NAS reboot, network objects and processes disappear; normal Transmission must remain disabled. From the persistent kit directory, run `stop` to clear any saved ownership marker, then `start` and repeat the external torrent check. If stop reports no ownership marker, proceed to start; investigate any other error. Confirm App Center did not re-enable the normal QPKG during an update.

For unattended boot, an administrator must integrate this start command with the model/firmware's supported boot mechanism after storage/network services are ready. Do not paste generic `/etc/rc.local` edits onto QNAP. A startup failure must leave Transmission stopped. This package makes no unattended reboot guarantee.

## 7. Normal commands and rollback

```sh
python3 qnap-vpn.py status
python3 qnap-vpn.py stop
python3 qnap-vpn.py start
```

`stop` signals processes in the owned namespace, waits for graceful exit, removes its network objects and resolver, and stops its identified relay. It does not restart the normal QPKG or delete downloads. If graceful exit takes longer than 30 seconds, wait and run `stop` again rather than force-killing the daemon.

To revert configuration, first stop this kit and ensure no Transmission instance is running:

```sh
python3 qnap-vpn.py stop
python3 qnap-vpn.py restore-settings
```

If already stopped, skip the stop command. Restore returns to the first saved settings, so later settings changes are also reverted. Then, only if you intentionally want normal non-namespace routing, re-enable the original QPKG in App Center. **Normal QPKG startup does not inherit this VPN isolation.**

## Troubleshooting

| Symptom | Meaning / action |
|---|---|
| `su` / `nohup` missing | Neither is needed by this kit. Do not reuse the old launcher. |
| Long Proton `.conf` filename | Supported: kit parses the file and uses `wg setconf`, not `wg-quick strip` with a filename-derived interface. |
| Web UI says files missing | Confirm `web_home/index.html` exists and the selected account can read it. |
| HTTP 401 | Server reached; credentials required. Not yet proof that web assets work. |
| HTTP 403 / host rejection | Check whitelist settings and use the numeric NAS LAN URL. Do not broadly disable authentication. |
| HEAD (`curl -I`) returns 501 | Use a normal GET/browser test; HEAD support is not required. |
| Proxy connection reset | Inspect `transmission.log` and `proxy.log`; daemon may have exited or RPC may not be listening. |
| Permission denied | Correct the selected account's access to data/download/web directories; avoid global chmod/chown. |
| `queue.json` absent at first startup | Earlier daemon continued running despite this; inspect subsequent log lines rather than treating it alone as fatal. |
| Peer port closed | Incoming Proton peer forwarding is a separate feature, not provided by this kit; outgoing peer connections can still work. |
| No tracker IP result | Inconclusive. Try the alternate service and inspect tracker error details. |
| Public URL differs from LAN UI | Inspect actual router destination and browser caching; do not assume both URLs reach the same daemon. |
| Command failure / unsupported firewall | Setup fails rather than starting an unfiltered daemon. Check kernel tools/support; do not remove firewall checks to get past the error. |
| Namespace exists but ownership marker missing | Do not delete it blindly. Inspect namespace/process ownership; this may be another installation or manual change. |

Logs are in the kit directory. Before sharing logs, remove usernames, torrent names, trackers, paths and any credentials. Never share a WireGuard profile or private runtime file.

When finished, use `exit` to leave the root shell. If you elevated with `sudo -i`, a second `exit` closes the original SSH session. The detached processes continue running.

## Remote access and limits

Use a trusted remote-access VPN into your LAN or a properly configured HTTPS reverse proxy. **Do not expose the raw HTTP Transmission UI directly with a router port-forward.** Earlier chat advice to forward a public port directly to this unencrypted UI is not carried forward: Basic authentication over HTTP does not protect credentials in transit. Router web-access ports and Proton peer-forwarding ports are different concerns.

This package does not configure Proton NAT-PMP port forwarding, automatic VPN endpoint failover, log rotation, autostart, or app updates. Log files should be monitored for growth. Download locations are normal NAS filesystem paths; the namespace does not require SMB mounts. A VPN does not change which filesystem permissions the daemon has.

## Sources and validation scope

- WireGuard's official namespace design: https://www.wireguard.com/netns/
- Transmission environment variables (web assets and proxy variables): https://github.com/transmission/transmission/blob/main/docs/Environment-Variables.md
- Transmission settings: https://github.com/transmission/transmission/blob/main/docs/Editing-Configuration-Files.md
- TorGuard test: https://torguard.net/check-my-torrent-ip-address/
- IPLeak test explanation: https://ipleak.net/

Local validation: Python compilation, JSON parsing, and a real local-socket relay test including a client half-close. Privileged WireGuard/network-namespace/QNAP operations cannot be validated in this build environment. Treat the recipient's acceptance checks as mandatory before relying on the new kit.
