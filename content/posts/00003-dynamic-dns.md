---
title: "Dynamic DNS"
date: 2026-10-25
draft: false
description: "An introduction to Domain Name Systems Dynamic DNS"
tags: ["dns"]
categories: ["explainers"]
---

## What is Dynamic DNS?

I touched on this in my [post about WireGuard](/posts/vpn-phone-home/), and thought it was important enough to deserve its own explainer.

### What is DNS?

DNS stands for Domain Name System. It's basically a big key-value store that maps structured _domain names_ to _IP addresses_.

```
larpsec.net -> 185.199.108.153
```

#### Names and Registrars

You can buy a domain name from a domain registrar (companies like namecheap.com, bluehost.com, godaddy.com, etc.), and then you pay a recurring fee every year to keep the name.
Generally for an unowned domain name, shorter names are more expensive, and each top-level domain has its own pricing.
If a name is already owned by someone (eg facebook.com or larpsec.net), they're allowed to set the price of it akin to a piece of real estate.

All domains are composed of a few parts, and they're read right to left:

```
subdomain.domain.top-level-domain.
```

The first component is _the root_: `.`, all domains descend from the root.
There's a historical reason for this, but you don't need to know it.
All modern web browsers will autofill the root if you leave it off,
but you can navigate to `larpsec.net.` to try it out.

The root is followed by the Top Level Domain: `com`, `net`, `org` and `gov` are the best known.
Some countries have top-level domains, like `.jp` for Japan, `.io` for the British Indian Ocean Territory, or `.ai` for Anguilla.
Many of those countries will offer subdomains as top-level domains, like `co.uk` or `gov.uk`, respectively for UK-based commercial and governmental organizations.

The `domain` is the component you can purchase and rent. `larpsec` in the case of this site.
For a home user, looking to expose something behind a VPN,
I recommend choosing something short and easy to type on a cheap top-level domain.
Check what your registrar offers to find one that suits your use case.

#### Subdomains and Records

Once you have a registered domain, you can attach various types of records to it.
There's many types of records, but the three we need to know are `A`, `AAAA`, and `CNAME`:

`A` records take a `subdomain` name and point to an IPv4 address

```
A, subdomain, 192.168.100.102
```

`AAAA` records take a `subdomain` name and point to an IPv6 address

```
AAAA, subdomain, 2001:0db8:85a3:0000:0000:8a2e:0370:7334
```

`CNAME` records take a `subdomain` name and point it to another domain name

```
CNAME, subdomain, other-domain.com.
```

You can assign a record to your registered domain, without a subdomain, by specifying `@` as the subdomain.

Your registrar should have an admin panel per domain, that allows you to create DNS records on each domain you have registered.

### What makes it Dynamic DNS?

Some DNS Registrars support dynamic records,
which means that the registrar offers an API for programmatically updating A and AAAA records.

Third-party services are available to manage this, but Namecheap (and probably some other registrars) support it directly.

Your dynamic DNS service should provide an API with an endpoint, password.
It will take request with a list of subdomains to update and an IPv4 or IPv6 Address to create or update records to.
Many systems will optionally take an IP address, and will default to using the request's source address as a target IP for the record.
This is a nice convenience, since a record originating from within your home network will appear to the Dynamic DNS service as the correct target.

## Why should I care about this?

In the big picture, there are 4,294,967,296 IPv4 addresses available. Less than one per person alive today.

The system was initially designed so that an ISP would buy a block of IP addresses.
Users would connect via a dial-up modem only when they wanted to go online.
These connections blocked phone service, so users would disconnect their modem and return their IP address to the pool after a few minutes or hours.

Because of these constraints (and many others I don't have time to get into here), consumer facing internet services and hardware assume that your IP address is temporary and subject to change.

By using a Dynamic DNS record, you can have a stable name for connections to services running on your home internet.
You don't need to remember what your IP address is, and to update every reference to it every time your modem is restarted.
Instead, you can configure a domain name that will auto update to your IP address, whenever it changes.

## How do I configure this?

### Ingredients

1.  A Domain Name, hosted at a Registrar that supports Dynamic DNS
1.  A server running on your home network

### Runbook

1. Buy a domain name
1. Add a dynamic DNS record to it
1. Write a script to update your dynamic DNS record
1. Setup a scheduler to run your script at regular intervals

#### Buy a domain name

I'm partial to namecheap.com, because I'm cheap, but any domain registrar that supports dynamic DNS records will do.
The examples here will use Namecheap's UI and API for convenience, but the concepts don't change between providers.

Buy a domain name that is cheap to renew, and short and easy to type.
For illustrative purposes, I'm going to use `larpsec.net`.

#### Add a dynamic dns record to it

You can create a new Dynamic DNS record, the same way you'd create a new `A`, `AAAA`, `CNAME`, or `TXT` record manually.

![This is what Dynamic Records Look like on Namecheap](/images/dynamic-dns/dynamic-dns.png)

It'll look something like this.

Namecheap requires you to authenticate yourself before updating domain records (otherwise I could just guess the domain name, and then divert all your traffic back to my blog hahaha). That's what the Dynamic DNS Password is for.

**DO NOT EVER SHARE IT WITH ANYONE!**

I included one in the below screenshot for illustrative purposes, but I rotated it, deleted the dynamic record, and disabled dynamic dns on this domain before publishing.
If you ever think you exposed this password somewhere, immediately click the pinwheel icon next to it to invalidate it, and generate a new one.
You'll have to update your scripts after or your dns will break, it's really annoying.

**...so don't leak your secrets!**

![Dynamic DNS status with cleared password](/images/dynamic-dns/dynamic-dns-password-cleared.png)

#### Write a script to update your dynamic DNS record:

Namecheap accepts Dynamic DNS updates via a GET request. The script below uses curl to hit this API with your password, hostname (subdomain), and an optional IP address. Namecheap defaults to using the request sender's IP address if you do not provide one.
Set `host` to `@` for the root domain or to the subdomain name. We used `vpn` as a subdomain above.

```sh
#!/bin/sh
set -eu

: "${NAMECHEAP_DDNS_PASSWORD:?Set NAMECHEAP_DDNS_PASSWORD first}"

domain="${1:?Usage: NAMECHEAP_DDNS_PASSWORD=... $0 DOMAIN HOST [IP]}"
host="${2:?Usage: NAMECHEAP_DDNS_PASSWORD=... $0 DOMAIN HOST [IP]}"
ip="${3-}"

update_dns() {
  curl \
    --fail \
    --silent \
    --show-error \
    --get \
    --data-urlencode "host=$host" \
    --data-urlencode "domain=$domain" \
    --data-urlencode "password=$NAMECHEAP_DDNS_PASSWORD" \
    "$@" \
    "https://dynamicdns.park-your-domain.com/update"
}

if [ -n "$ip" ]; then
  update_dns --data-urlencode "ip=$ip"
else
  update_dns
fi
```

#### Run the script from cron

Cron doesn't load your shell configuration. If we want the script to see the password, we need to put it somewhere first. Store it in a file that only your user can read.

```sh
umask 077
mkdir -p "$HOME/.config"
printf '%s\n' 'NAMECHEAP_DDNS_PASSWORD="replace-with-your-dynamic-dns-password"' \
  > "$HOME/.config/namecheap-ddns.env"
chmod 600 "$HOME/.config/namecheap-ddns.env"
```

Now create a small wrapper script at `/Users/you/bin/namecheap-ddns-cron`:

```sh
#!/bin/sh
set -eu

. "$HOME/.config/namecheap-ddns.env"
exec /Users/you/bin/update-namecheap-ddns.sh larpsec.net vpn
```

Make the wrapper executable, then open your crontab:

```sh
chmod 700 /Users/you/bin/namecheap-ddns-cron
crontab -e
```

Add this line to run the update every 15 minutes:

```cron
*/15 * * * * /Users/you/bin/namecheap-ddns-cron >> /tmp/namecheap-ddns.log 2>&1
```

Replace the example paths, domain, host, and password before you run these commands. Keep the password out of the crontab and source files.

### All done!

Now is a good time to check if the record has updated. Once the script has run, you should see your home IP address in the Namecheap UI.

Now that you have a reliable way to reach your home infrastructure from the wider internet, you have a base to do all kinds of useful projects, like [setting up a home VPN](/posts/vpn-phone-home/)!
