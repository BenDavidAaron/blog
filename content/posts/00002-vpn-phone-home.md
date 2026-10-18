---
title: "VPN Phone Home"
date: 2026-10-18
draft: true
description: "How to get started with a wireguard peer"
tags: ["vpn", "selfhosting", "wireguard"]
categories: ["guides"]
---

## What is this about?

This is a guide on how to setup and expose a wireguard peer on the
average home nework.

## Who is this for?

This is a write up for anyone who self-hosts a service on thier home network
and would like to access it via the public internet, without exposing it to the whole world.
If you're already self hosting Immich, Jellyfin, or a similar service, this will give you a big quality of life improvement for not much effort.

## Why should I do all this?

Hypothetically: if you're using Immich to store your photos, you can sync new photos every evening when you get home, connect to your home wifi, and charge your phone.

If you go on vacation for two weeks, your library will get out of sync, and require a big sync when you get home. By exposing your network via a wireguard tunnel, you can access your Immich instance from any network without exposing machines to the internet.

## How do I do this?

#### Hardware

You need one (1) small computer, that can stay on 24/7, with an ethernet interface.

A Single Board Computer like a Rasberry Pi is the typical machine one would use for this. They are common because they are small (easy to hide in the back of a cabinent or closet), and they use relatively little power.

An old laptop is also an excelent platform, older machines typically have an ethernet connection, and USB-Ethernet adapters are cheap and robust.
Additionally, a laptop's battery will function as a built in backup power supply, so your VPN endpoint can persist through short power outages.

I **strongly** reccomend keeping your VPN peer on it's own dedicated machine. This will make it easier to segment from the rest of your services, and prevent you from having to forward any traffic from your router to another machine with sensitive data or content.

Choosing a Server OS is outside the scope of this article.
I reach for [Debian](https://www.debian.org/) for projects like this because it's boring and stable.

#### Software

We'll be using [wireguard](https://www.wireguard.com/#conceptual-overview) for this project. You can read more about it at that link if you're a big ol' nerd, but the practical reasons to use it are:

- Wireguard has easy to use first-party client software on MacOS, Windows, iOS, and Android - Easy for your whole household to access.
- Wireguard has a minimal attack surface - only one port is open, and it will only respond to clients with a known key. Effectively, it is invisible to passive scanners (unlike OpenVPN, or commercial VPNs).
- Wireguard is redistributed as part of the [PiVPN project](https://github.com/pivpn/pivpn), which simplifies administration

##### Server

1. [PiVPN](https://www.pivpn.io/)

##### Client

1. Wireguard Client

- for [MacOS](https://apps.apple.com/us/app/wireguard/id1451685025)
- for [Windows](https://download.wireguard.com/windows-client/)
- for [iOS](https://apps.apple.com/us/app/wireguard/id1441195209)
- for [Android](https://play.google.com/store/apps/details?id=com.wireguard.android&hl=en)

##### Services

1. Registered domain name, with Dynamic DNS Support

### Runbook

1. [Set up your server hardware](#setup-your-server-hardware)
1. [Configure Dynamic DNS](#configure-dynamic-dns)
1. [Configure a static IP address](#configure-a-static-ip-address)
1. [Run the PiVPN installer](#run-the-pivpn-installer)
1. [Configure NAT forwarding](#configure-nat-forwarding)
1. [Install WireGuard on your client device](#install-wireguard-on-your-client-device)
1. [Create a client device profile](#create-a-client-device-profile)
1. [Configure your client device](#configure-your-client-device)

#### Setup your server hardware

Obviously, I can't prescribe how to setup your server hardware (I already wrote this blog post), so I'm just going to give you a handfull of bullet points to keep in mind:

- Install a stable Debian release on it
- Ensure it is plugged into your router via Ethernet (no WiFi)
- Make sure it is dedicated to this purpose
  - this is the lock on your front door
  - do _not_ also make it a window!
- Ensure you have access to the server via `ssh`, so you don't have to work on it from the closet or cabinent where it's isntalled.

#### Configure Dynamic DNS

We'll cover this in a future article, but it's important to touch on here.

Since most consumer internet services assign IP addresses to users dynamically, you need a stable name for your clients to connect to.
Using a dynamic DNS provider ensures that `vpn.notLarpSec.net` always points to your home router's IP address.

#### Configure a static IP address

First things first, you must make sure that the Local IP address of your server is static.

By default, most home internet routers will assign dynamic IP addresses to new hosts.
When a new computer connects to the network, it is assigned a random IP address from it's pool.

The easiest way to set this up is to login to your router's admin pannel,
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

- enable unatended updates
- Use Wireguard (instead of OpenVPN)
- Use your domain name instead of an IP address for clients to connect

Unatended updates allow PiVPN to periodically check for and install new security fixes and updates from your Linux Distribution Maintainers.
This will save you a few minutes of admin work every month.

Using Wireguard is an important choice because the project originially used OpenVPN, and still supports it.
Nothing against OpenVPN, I just find Wiregurard to be easier to use and more performant.

Using your domain name instead of an IP address is important for clients to connect. When your IP address (assigned by your ISP changes),
statically configured client profiles will fall out of date, and your client devices will be unable to connect.
Since dynamic DNS keeps your IP address up to date, you can be reasonably sure that your domain name will consistently point to your home router.

#### Configure NAT port forwarding

Now log back into your router, we have one last change to make:

We're going to add a Port Forwarding Rule for traffic on WireGuard's port.

By default, most home internet routers will ignore all internet traffic coming in.
They use NAT (Network Address Translation) to allow devices inside your home network to reach out and request sites like facbook.com, google.com, or larpsec.net,
from behind your router's IP Address.

In our case we _want_ to allow incoming wireguard requests,
and we want your router to forward them to the wireguard peer on the server we just configured.

Every router is different, but the configuration will ask for three parameters:

- a source port number, this should be the wireguard port you configured when setting up PiVPN above (default is 51820)
- a destination IP address, this should be the static or reserved IP address you set for your server
- a destionation port number, this should be the same wireguard port as above

Create one port forwarding rule for the three of these and then you should be all set.

#### Install WireGuard on your client device

Clients are availible for many systems at https://www.wireguard.com/install/

#### Create a client device profile

Log back into your server, and use the PiVPN CLI to generate a new client profile for your first device:

Use the command `sudo pivpn -a` to add a client.
This client profile will contain:

- A private key for your device
- The server's public key
- The endpoint (domain name and port number) of your wireguard peer
- The address range that wireguard will tunnel connections to
  - you can restrict this to `192.168.0.0/16`, `172.16.0.0/12`, and `10.0.0.0/8` to not tunnel public internet traffic through your VPN
  - the pivpn defaults are probably okay

This profile will be saved on the device, and it will need to be copied onto your client devices to use.

Your VPN peer should be fully usable at this point.

#### Configure your client device

The final step is to setup your client device to connect to your wireguard peer.

You can run `sudo pivpn -qr` to export the profile to a phone.
A QR code containing the profile will be shown in your terminal,
and you can scan it with the wireguard app using the "import tunnel from QR code".

You can import profile files from your VPN server to another computer using `rsync`, `sftp`, or `scp`,
and then import them into wireguard client applications for Windows or MacOS.

### Try it out!

Now you can try connecting to any service hosted on your home network though the VPN tunnel to test it out.

Enjoy, and stay safe out there.
