# Using Caddy with OpenWrt

> [!NOTE]
> If you are seeing this note, this article is INCOMPLETE.

Using Caddy by itself is pretty straight forward. However, when mixing dynamic DNS and publically recognized self-signed SSL certificates (like from Let's Encrypt or Zero SSL), it becomes less so. `xcaddy` is an option to include the modules so configuration can be easier. However, I am used to recent versions of OPNsense requiring Caddy to be ran separate from dynamic DNS and automatic SSL certificates (at least once I figured out what needed to be done, [following the removal of plugins from the OPNsense community plugin build of `caddy`](https://forum.opnsense.org/index.php?topic=47606.0).)

OpenWrt has packages for [acme.sh](https://acme.sh), and dynamic DNS, with sufficient documentation about configuring it, (A little less so for `acme.sh` if you're doing `DNS-01` challenges with certain dynamic DNS providers, but having it already set up on OPNsense previously allows me to essentially just copy and paste the configured options).

OpenWrt however, lacks a package (at least through the default repositories) for Caddy. So you're either to rely on `xcaddy`, or to download their pre-compiled binary releases (which however, do not include the Layer4 proxy that the OPNsense community repository has).

At the time of writing this guide, I do not need the Layer4 proxy features, so I can rely on pre-compiled binaries. Should that change in the future, I will try to leave a note here (really though, using `xcaddy` is quite straightforward).

An additional, albeit, somewhat manually configured feature from OPNsense that seems missing from OpenWrt documentation, is integrating [Crowdsec](https://www.crowdsec.net/) with Caddy's access logs.

> [!WARNING]
> I am NOT a cybersecurity specialist or professional. To this day, I am still continuing to learn, and further build out how Crowdsec integrates with running services. This will not be a guide of how to secure, but rather how to configure it.

## What is needed

1. Device running OpenWrt (though these instructions could be used to build something on another platform, adjusting instructions as necessary for init systems and configuration file locations.)
2. ~500MB of storage available (This could more or less. The largest users of storage is Crowdsec (not the firewall bouncer) and Caddy. Caddy will use much more storage if you decide to use `xcaddy` to compile on-device.)
3. A supported architecture for Caddy (and/or `xcaddy`)

## Install packages

1. `acme-acmesh` for SSL certificate generation
    - [OpenWrt documentation](https://openwrt.org/docs/guide-user/services/tls/acmesh#using_gui)
    - I installed `acme-acmesh-dnsapi` to do `DNS-01` challenges. (CAs typically require this if you want a wildcard certificate)
    - I installed `luci-app-acme` because I wanted the GUI. I feel it lacks a couple items I am used to from OPNsense, but it has still proved itself valuable for me.
2. `ddns-scripts`
    - Really, just follow [the documentation](https://openwrt.org/docs/guide-user/services/ddns/client#web_interface_instructions).
    - I opted to install the Luci app.
3. `crowdsec` and `crowdsec-firewall-bouncer`
    - At the time of writing, [OpenWrt's documentation for Crowdsec](https://openwrt.org/docs/guide-user/services/crowdsec) is considered still in progress "of being transferred from the community forum."

I will be using this setup on a [Google Wifi](https://openwrt.org/toh/google/wifi), which has plenty of storage (4GB eMMC) and enough RAM (512MB) to run this stack. No, it will not be the fastest, but the internet speeds where this is deployed is not fast enough to overload the system.

> [!NOTE]
> If you are seeing this note, that means this stack has not yet actually been deployed to a Google Wifi yet. I do not actually know if the Google Wifi has enough RAM for this stack. Testing on a x86_64 system, Luci shows 1GB+ of RAM usage, but checking the true used in `btop++` did not show more than 300MB. `Zram` as swap is a thing, worst case scenario.

## Setup Dynamic DNS

> [!NOTE]
> You should already be familar with the requirements for setting up dynamic DNS (eg. Public, internet-routable IPv4 and/or IPv6). My setup consists of a CGNAT IPv4, and a single GUA IPv6 /64 prefix, so I will configure dynamic DNS only for IPv6.

I was going to include a decent amount of configuration info, but there is an OpenWrt wiki page, [specifically for DuckDNS](https://openwrt.org/docs/guide-user/services/ddns/duckdns), with a section specifically for IPv6 setup.

> [!WARNING]
> At the time of writing, DuckDNS does **NOT** support [updating IP addresses over IPv6](https://www.duckdns.org/faqs.jsp#:~:text=Q%3A%20why%20can%27t%20you%20detect%20IPv6%20addresses%3F). To clarify, it does support updating your personal IPv6 addresses, but not communicated over IPv6. So despite what the OpenWrt wiki says, you cannot have `use_ipv6` set to `1`. It will timeout trying to update because it cannot reach DuckDNS over IPv6.

## Setup acme.sh

> [!NOTE]
> Depending on your dynamic DNS provider, propagation of your newly registered sub/domain can take a substantial amount of time. This also includes the TXT record used for the `DNS-01` challenge. If you are impatient, `HTTP-01` exists, just be aware of the drawbacks and/or limitations.

I still recommend referring to OpenWrt's documentation, but there many paths to take. I've included my configuration both as a reminder to myself, but perhaps inspiration to another.

> [!NOTE]
> I am configuring a wildcard certificate. There is a quirk with DuckDNS and `acme.sh` such that the wildcard domain(s) should be the **ONLY** domain name(s) in the configuration. Typically, the root sub/domain would be the primary, with a wildcard as an `alt` names. For example, instead of `mysubdomain.duckdns.org, *.mysubdomain.duckdns.org`, only the wildcard domain is included, `*.mysubdomain.duckdns.org`. Otherwise `acme.sh` will assign two TXT records to the same `mysubdomain.duckdns.org`, overwriting the first with the second. However, the CA is expecting the first TXT record, and verification will fail. Inspecting the `acme.sh` logs with ``--debug`` will prove it.

### `Luci` -> `Services` -> `ACME certificates`

1. Add your account email. Maybe use a real email?
2. Enable debug logging (or not.)

### `Certificate config` -> `Text box`

1. Type the name you want to use. This is stored in the local configuration so it must be a valid UCI identifier.
2. Add it.

### `General Settings`

1. Enable it (of course).
2. Add **ONLY** the wildcard domain to the `Domain names` field.
3. Change `Validation Method` to `DNS`.

### `DNS Challenge Validation`

1. Set `DNS API` to `DuckDNS`
2. Paste your DuckDNS token into the `DuckDNS Token` field. Pressing `Enter` or clearing the focus from the textbox will automatically add it to the `DNS API credentials` field.
3. You know who you are if you need to change the challenge or domain aliases.

### `Advanced Settings`

1. It wouldn't hurt to use the staging server until the configuration is correct. Rate limits are less so there are more retries in case the configuration is improper.
2. Choose your key type. This is not an article about the art of cryptography. Make your own informed choice.
3. You know who you are if you want to change the `ACME server URL`
4. I am not totally sure if the `Days until renewal` is actually respected by the service. I will eventually edit this if/when I find out.

### Now what?

Either wait until midnight in your configured timezone, for the `cron` script to try to acquire/renew your certificate, or do it yourself:

`/etc/init.d/acme renew`

Hopefully it succeeds. If not, diagnose, search, troubleshoot, and good luck. That is why the staging server is set, to be able to retry frequently. You *did* set the staging server, *right*?

The issued certificates can be found under /etc/acme/`mysubdomain`.duckdns.org_`key_type`.

## Setup Caddy

1. Downloaded latest build, of your choosing, of the [Caddy server](https://github.com/caddyserver/caddy/releases).


> [!NOTE]
> So that it will be already in `$PATH`, I copied the Caddy binary to `/usr/bin/`.

2. Create the `Caddyfile` configuration file

> [!NOTE]
> The configuration file can be placed wherever you want. Caddy allows to be passed any arbitrary path.

```
mkdir -p /etc/caddy/
nano /etc/caddy/Caddyfile
```

> [!NOTE]
> In this simple configuration, Caddy will use its internal file browser on the `files` subdomain, while using the SSL certificate on the wildcard domain. This configuration file was copied and adapted from an OPNsense install, so there could be options configured that are not optimal, or even doing anything (such as the Layer4 proxy section).

```
# Global Options
{
        log {
                format json {
                        time_format rfc3339
                }
        }

        http_port 80
        https_port 443

        servers {
                protocols h1 h2 h3
                log_credentials
        }

        auto_https off
        grace_period 10s
}

# Reverse Proxy Configuration

# Layer4 default HTTP port
:80 {
}
# Layer4 default HTTPS port
:443 {
}

*.mysubdomain.duckdns.org {
        log {
                output file /var/log/caddy/access/wildcard-mysubdomain.log {
                        roll_keep_for 7d
                }
        }
        tls /etc/acme/*.mysubdomain.duckdns.org_ecc/fullchain.cer /etc/acme/*.mysubdomain.duckdns.org_ecc/*.mysubdomain.duckdns.org.key {
        }
}

https://files.mysubdomain.duckdns.org {
        log {
                output file /var/log/caddy/access/files-mysubdomain.log {
                        roll_keep_for 7d
                }
        }

        handle {
                root * /srv/caddy
                file_server browse
        }
}
```

> [!WARNING]
> Consider running Caddy as a user other than `root`. Doing this requires changing the listening HTTP/S port(s), but that is why port forwarding exists. That is outside the scope of this documentation.

3. `init.d` script for Caddy

I was lazy, and found an `init.d` script that someone else wrote.

[@wangqs_eclipse - Install caddy as reverse proxy on OpenWrt for my home lab](https://medium.com/@wangqs_eclipse/install-caddy-as-reverse-proxy-on-openwrt-for-my-home-lab-b8ee7d01fea9)

```bash
nano /etc/init.d/caddy
```

```bash
#!/bin/sh /etc/rc.common

START=95
STOP=5

BIN=/usr/bin/caddy
CONFIG=/etc/caddy/Caddyfile
PIDFILE=/var/run/caddy.pid
LOGFILE=/var/log/caddy.log

start() {
    echo "Starting Caddy..."
    start-stop-daemon -S -b -m -p $PIDFILE -x $BIN -- run --config $CONFIG --adapter caddyfile >> $LOGFILE 2>&1
}

stop() {
    echo "Stopping Caddy..."
    start-stop-daemon -K -p $PIDFILE
}

restart() {
    stop
    sleep 1
    start
}
```

```
chmod +x /etc/init.d/caddy
/etc/init.d/caddy enable
/etc/init.d/caddy start
```

## Firewall rules (at least for IPv6)

1. Change the ports that `uhttpd` is listening on, for Luci, and optionally remove the HTTP listener.

```bash
config uhttpd 'main'
        list listen_https '0.0.0.0:8443'
        list listen_https '[::]:8443'
        option redirect_https '1'
        option home '/www'

# blah blah blah
```

2. If within a firewall zone that by default allows `INPUT` traffic (such as `LAN`), assuming a relatively default firewall zone configuration, simply start Caddy, and the service on the subdomain should be accessible.

> [!NOTE]
> yeah idk if true but sure sounds like it

3. If within a firewall zone that by default does not allow `INPUT` traffic (such as `WAN`), create a traffic rule to allow it.

`Luci -> Network -> Firewall -> Traffic Rules -> Add`

Name the rule, (and if you're using HTTP/3, UDP may need to be enabled for QUIC, dependent on what is behind Caddy, otherwise only TCP will suffice), set `Destination zone` to `Device (input)`, and `Destination port` to `80`. (This is our HTTP rule, but there is no need for worry since you'll use HTTPS redirect with Caddy. *Right*?) Once saved, clone the rule, changing the `Destination port` to `443`.

## Crowdsec Integration

1. Follow OpenWrt's [documentation](https://openwrt.org/docs/guide-user/services/crowdsec). Seriously.

2. Then go familarize yourself with OPNsense's [documentation](https://docs.opnsense.org/manual/how-tos/caddy.html#crowdsec-integration).

```bash
cscli collections install crowdsecurity/caddy
nano /etc/crowdsec/acquis.yaml
```

```yaml
#filenames:
#  - /var/log/nginx/*.log
#  - ./tests/nginx/nginx.log
#labels:
#  type: nginx
#---
#filenames:
# - /var/log/auth.log
# - /var/log/syslog
#labels:
#  type: syslog
#---
#filenames:
# - /var/log/httpd-access.log
# - /var/log/httpd-error.log
#labels:
#  type: apache2
---
filenames:
  - /var/log/caddy/access/*.log
force_inotify: true
poll_without_inotify: true
labels:
  type: caddy
```

```bash
service crowdsec restart
cscli metrics
```

```
╭───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
│ Acquisition Metrics                                                                                                                                           │
├─────────────────────────────────────────────────────────────────────┬────────────┬──────────────┬────────────────┬────────────────────────┬───────────────────┤
│ Source                                                              │ Lines read │ Lines parsed │ Lines unparsed │ Lines poured to bucket │ Lines whitelisted │
├─────────────────────────────────────────────────────────────────────┼────────────┼──────────────┼────────────────┼────────────────────────┼───────────────────┤
│ file:/var/log/caddy/access/files-mysubdomain.log                    │ 8.10k      │ 8.10k        │ -              │ 444                    │ -                 │
╰─────────────────────────────────────────────────────────────────────┴────────────┴──────────────┴────────────────┴────────────────────────┴───────────────────╯
╭───────────────────────────────────────────────────────────────────╮
│ Parser Metrics                                                    │
├────────────────────────────────────┬─────────┬─────────┬──────────┤
│ Parsers                            │ Hits    │ Parsed  │ Unparsed │
├────────────────────────────────────┼─────────┼─────────┼──────────┤
│ crowdsecurity/caddy-logs           │ 8.10k   │ 8.10k   │ -        │
╰────────────────────────────────────┴─────────┴─────────┴──────────╯
```

> [!WARNING]
> I am pretty sure that you still need to feed `crowdsec` logs from the proxied applications. That is your responsibility to figure out.