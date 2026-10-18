---
title: "VPN Phone Home"
date: 2026-10-17
draft: false
description: "How to get started with a WireGuard peer"
tags: ["vpn", "selfhosting", "wireguard"]
categories: ["guides"]
---

## What is this about?

This is a guide on how to set up and expose a WireGuard peer on the
average home network.

## Who is this for?

This is a write-up for anyone who self-hosts a service on their home network
and would like to access it via the public internet, without exposing it to the whole world.
If you're already self hosting Immich, Jellyfin, or a similar service, this will give you a big quality of life improvement for not much effort.

## Why should I do all this?

Hypothetically: if you're using Immich to store your photos, you can sync new photos every evening when you get home, connect to your home wifi, and charge your phone.

If you go on vacation for two weeks, your library will get out of sync, and require a big sync when you get home. By exposing a WireGuard endpoint, you can access your Immich instance from any network without exposing the service directly to the internet.

## How do I do this?

#### Hardware

You need one (1) small computer, that can stay on 24/7, with an ethernet interface.

A Single Board Computer like a Raspberry Pi is the typical machine one would use for this. They are common because they are small (easy to hide in the back of a cabinet or closet), and they use relatively little power.

An old laptop is also an excellent platform, older machines typically have an ethernet connection, and USB-Ethernet adapters are cheap and robust.
Additionally, a laptop's battery will function as a built-in backup power supply, so your VPN endpoint can persist through short power outages.

I **strongly** recommend keeping your VPN peer on its own dedicated machine. This will make it easier to segment from the rest of your services, and prevent you from having to forward any traffic from your router to another machine with sensitive data or content.

Choosing a Server OS is outside the scope of this article.
I reach for [Debian](https://www.debian.org/) for projects like this because it's boring and stable.

#### Software

We'll be using [WireGuard](https://www.wireguard.com/#conceptual-overview) for this project. You can read more about it at that link if you're a big ol' nerd, but the practical reasons to use it are:

- WireGuard has easy to use first-party client software on macOS, Windows, iOS, and Android - Easy for your whole household to access.
- WireGuard has a small attack surface. It uses one configured UDP port and ignores handshakes that do not use a valid key. This does not make the server invisible or secure other services on the host.
- [PiVPN](https://github.com/pivpn/pivpn) installs and manages WireGuard, which simplifies administration.

##### Server

1. [PiVPN](https://www.pivpn.io/)

##### Client

1. WireGuard Client

- for [macOS](https://apps.apple.com/us/app/wireguard/id1451685025)
- for [Windows](https://download.wireguard.com/windows-client/)
- for [iOS](https://apps.apple.com/us/app/wireguard/id1441195209)
- for [Android](https://play.google.com/store/apps/details?id=com.wireguard.android&hl=en)

##### Services

1. Registered domain name and a Dynamic DNS provider

### Runbook

1. [Set up your server hardware](#set-up-your-server-hardware)
1. [Configure Dynamic DNS](#configure-dynamic-dns)
1. [Configure a static IP address](#configure-a-static-ip-address)
1. [Run the PiVPN installer](#run-the-pivpn-installer)
1. [Configure NAT forwarding](#configure-nat-port-forwarding)
1. [Install WireGuard on your client device](#install-wireguard-on-your-client-device)
1. [Create a client device profile](#create-a-client-device-profile)
1. [Configure your client device](#configure-your-client-device)

#### Set up your server hardware

Obviously, I can't prescribe how to set up your server hardware (I already wrote this blog post), so I'm just going to give you a handful of bullet points to keep in mind:

- Install a stable Debian release on it
- Ensure it is plugged into your router via Ethernet (no WiFi)
- Make sure it is dedicated to this purpose
  - this is the lock on your front door
  - do _not_ also make it a window!
- Ensure you have access to the server via `ssh`, so you don't have to work on it from the closet or cabinet where it's installed.

#### Configure Dynamic DNS

We'll cover this in a future article, but it's important to touch on it here.

Since most consumer internet services assign public IP addresses to users dynamically, you need a stable name for your clients to connect to.
After you configure a Dynamic DNS updater, `vpn.larpsec.net` will point to your current home router IP address.
If your ISP uses carrier-grade NAT, port forwarding will not work unless your ISP provides a public address or IPv6 service.

#### Configure a static IP address

First things first, you must make sure that the Local IP address of your server is static.

By default, most home internet routers will assign dynamic IP addresses to new hosts.
When a new computer connects to the network, it is assigned a random IP address from its pool.

The easiest way to set this up is to log in to your router's admin panel,
which is typically at [http://192.168.1.1](http://192.168.1.1) or [http://192.168.1.254](http://192.168.1.254),
find the `DHCP` tab in the admin panel,
then plug in your new server, and refresh the page.
The new entry in the DHCP table will be your new server.
Then you should be able to click into it and either assign a static IP to the device's MAC address (hardware identifier),
or create a DHCP Reservation so that that device will always get that same IP address.
Either will suffice in this case.

#### Run the PiVPN installer

Now you can log in to your new server and run the [PiVPN](https://www.pivpn.io/) installer.
The installer will run you through a wizard that helps you configure the system. Make sure to:

- enable unattended updates
- Use WireGuard (instead of OpenVPN)
- Use your domain name instead of an IP address for clients to connect

Unattended updates allow PiVPN to periodically check for and install new security fixes and updates from your Linux distribution maintainers.
This will save you a few minutes of admin work every month.

Using WireGuard is an important choice because the project originally used OpenVPN, and still supports it.
Nothing against OpenVPN, I just find WireGuard to be easier to use and more performant.

Using your domain name instead of an IP address is important for clients to connect. When the public IP address assigned by your ISP changes,
statically configured client profiles will fall out of date, and your client devices will be unable to connect.
Since dynamic DNS keeps your IP address up to date, you can be reasonably sure that your domain name will consistently point to your home router.

#### Configure NAT port forwarding

Now log back into your router, we have one last change to make:

We're going to add a Port Forwarding Rule for traffic on WireGuard's port.

By default, most home routers block unsolicited incoming internet traffic with a firewall.
They use NAT (Network Address Translation) to allow devices inside your home network to reach sites like facebook.com, google.com, or larpsec.net,
from behind your router's public IP address.

In our case we _want_ to allow incoming WireGuard requests,
and we want your router to forward them to the WireGuard peer on the server we just configured.

Every router is different, but the configuration will ask for four values:

- a protocol, set this to UDP
- an external port number, this should be the WireGuard port you configured when setting up PiVPN above (default is 51820)
- a destination IP address, this should be the static or reserved IP address you set for your server
- a destination port number, this should be the same WireGuard port as above

Create one port forwarding rule with these four values.

#### Install WireGuard on your client device

Clients are available for many systems at https://www.wireguard.com/install/

#### Create a client device profile

Log back into your server, and use the PiVPN CLI to generate a new client profile for your first device:

Use the command `sudo pivpn -a` to add a client.
This client profile will contain:

- A private key for your device
- The server's public key
- The endpoint (domain name and port number) of your WireGuard peer
- The address ranges that WireGuard will tunnel connections to
  - for split tunneling, use your actual home LAN CIDR and WireGuard network CIDR
  - check the generated values before you use the profile

This profile will be saved on the server, and it will need to be copied onto your client devices.
Protect the profile because it contains the client's private key.

Your VPN peer should be fully usable at this point.

#### Configure your client device

The final step is to set up your client device to connect to your WireGuard peer.

You can run `sudo pivpn -qr` to export the profile to a phone.
A QR code containing the profile will be shown in your terminal,
and you can scan it with the WireGuard app using the "import tunnel from QR code".

You can import profile files from your VPN server to another computer using `rsync`, `sftp`, or `scp`,
and then import them into WireGuard client applications for Windows or macOS.

### Try it out!

Now you can try connecting to any service hosted on your home network through the VPN tunnel to test it out.

Enjoy, and stay safe out there.
