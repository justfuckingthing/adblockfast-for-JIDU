Step-by-step guide to install and configure AdBlock-Fast with HaGeZi Pro DNS-based ad blocking on JIDU 6J11 (6J11) and JIDU 6401 (6J01) routers running OpenWrt.
# JIDU 6J11 OpenWrt – AdBlock-Fast Installation Guide

A step-by-step guide to install and configure AdBlock-Fast on the JIDU 6J11 router running OpenWrt SNAPSHOT.

This setup uses:

* AdBlock-Fast for DNS-based ad blocking
* HaGeZi Pro blocklist
* dnsmasq as the DNS resolver
* LuCI Web UI for configuration and monitoring

## 1. Requirements

* JIDU 6J11 router with OpenWrt installed
* SSH access as `root`
* Working internet connection on the router
* OpenWrt package feeds configured for the installed firmware

> **Note:** These instructions were tested on JIDU 6J11 running OpenWrt SNAPSHOT with the `apk` package manager. Commands may differ on older OpenWrt releases that use `opkg`.

## 2. Update package lists

Connect to the router over SSH and run:

```sh
apk update
```

## 3. Install AdBlock-Fast and LuCI Web UI

Run:

```sh
apk add adblock-fast luci-app-adblock-fast
```

Install the recommended `gawk` dependency:

```sh
apk add gawk
```

Verify installation:

```sh
apk info | grep -i adblock
```

Expected packages:

```text
adblock-fast
luci-app-adblock-fast
```

## 4. Enable AdBlock-Fast

Enable the service in its configuration:

```sh
uci set adblock-fast.config.enabled='1'
uci commit adblock-fast
```

Enable automatic startup:

```sh
/etc/init.d/adblock-fast enable
```

## 5. Enable the HaGeZi Pro blocklist

First, inspect the available blocklists:

```sh
uci show adblock-fast | grep -E 'file_url|hagezi|enabled|url'
```

Open the LuCI interface:

**Services → AdBlock-Fast → Configuration → List Updates Schedule**

Find the HaGeZi Pro (domains) list and enable it.

The list URL used in this setup is:

```text
https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/wildcard/pro-onlydomains.txt
```

Save and Apply the configuration.

Alternatively, on the tested configuration where HaGeZi Pro is the first `file_url` section, enable it using SSH:

```sh
uci set adblock-fast.@file_url[0].enabled='1'
uci commit adblock-fast
```

> The section index `[0]` is specific to the configuration used in this guide. Check `uci show adblock-fast` before using this command on a different installation.

## 6. Start AdBlock-Fast

Run:

```sh
/etc/init.d/adblock-fast restart
```

Wait for the blocklist download and processing to finish.

Check the service log:

```sh
logread | grep -i 'adblock-fast' | tail -20
```

A successful setup should show messages indicating:

* Blocklist downloaded successfully
* Blocklist processed and formatted
* dnsmasq configuration updated
* dnsmasq restarted
* AdBlock-Fast is blocking domains

## 7. Verify DNS blocking

Check that the generated blocklist exists:

```sh
ls -lh /var/run/adblock-fast/dnsmasq.servers
```

Check whether a known ad domain is present:

```sh
grep -i 'ad-apac.doubleclick.net' /var/run/adblock-fast/dnsmasq.servers
```

Expected:

```text
server=/ad-apac.doubleclick.net/
```

Now test DNS resolution through the router's local DNS resolver:

```sh
nslookup ad-apac.doubleclick.net 127.0.0.1
```

Expected result:

```text
** server can't find ad-apac.doubleclick.net: NXDOMAIN
```

If the domain returns NXDOMAIN, the tested ad domain is being blocked by the router's DNS resolver.

Check the active dnsmasq process:

```sh
ps w | grep '[d]nsmasq'
```

Check the service status and blocking count in the LuCI interface.

## 8. Access the Web UI

Open your router's LuCI Web UI in a browser.

Navigate to:

**Services → AdBlock-Fast**

The Status page should show the service version, active status, number of blocked domains, and DNS resolver mode.

In the tested setup, the status showed:

```text
AdBlock-Fast 1.2.4-r4
Active
Blocking 228217 domains (with dnsmasq.servers)
```

The exact blocking count may change when the blocklist is updated.

## 9. Troubleshooting

### A. AdBlock-Fast is not visible in LuCI

Verify that both packages are installed:

```sh
apk info | grep -i adblock
```

Restart the LuCI backend and web server:

```sh
/etc/init.d/rpcd restart
/etc/init.d/uhttpd restart
rm -f /tmp/luci-indexcache
```

Then hard-refresh the browser with `Ctrl + F5` and log in again.

### B. Blocklist is not downloading

Check the log:

```sh
logread | grep -i 'adblock-fast' | tail -30
```

If `gawk` is missing:

```sh
apk update
apk add gawk
```

Ensure the router has internet access and that at least one blocklist is enabled.

### C. DNS blocking is not working

Check the generated DNS file:

```sh
ls -lh /var/run/adblock-fast/dnsmasq.servers
```

Verify that dnsmasq is running:

```sh
ps w | grep '[d]nsmasq'
```

Test a known blocked subdomain:

```sh
nslookup ad-apac.doubleclick.net 127.0.0.1
```

Do not assume that resolving the root domain `doubleclick.net` means all its ad-serving subdomains are unblocked. Test a domain that is explicitly present in the generated blocklist.

## 10. Useful commands

| Action              | Command                                        |
| ------------------- | ---------------------------------------------- |
| Start / restart     | `/etc/init.d/adblock-fast restart`             |
| Stop                | `/etc/init.d/adblock-fast stop`                |
| Enable startup      | `/etc/init.d/adblock-fast enable`              |
| Disable startup     | `/etc/init.d/adblock-fast disable`             |
| View logs           | `logread \| grep -i adblock-fast`              |
| View blocklist size | `ls -lh /var/run/adblock-fast/dnsmasq.servers` |

## Important notes

* AdBlock-Fast is DNS-based filtering. It does not guarantee removal of every advertisement, particularly ads served from the same domains as normal content.
* Apps or browsers using their own encrypted DNS may bypass ordinary router DNS filtering unless DNS traffic is redirected or otherwise controlled.
* Do not install multiple DNS-blocking services simultaneously without checking their resolver and port configuration.
* On OpenWrt SNAPSHOT, package availability depends on the matching firmware feeds. Avoid mixing packages from incompatible firmware revisions.

Official documentation: https://docs.mossdef.org/adblock-fast/

OpenWrt Ad blocking documentation: https://openwrt.org/docs/guide-user/services/ad-blocking
