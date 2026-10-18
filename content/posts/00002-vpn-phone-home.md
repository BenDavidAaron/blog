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

### Hardware

You need one (1) small computer, that can stay on 24/7, with an ethernet interface.

A Single Board Computer like a Rasberry Pi is the typical machine one would use for this. They are common because they are small (easy to hide in the back of a cabinent or closet), and they use relatively little power.

An old laptop is also an excelent platform, older machines typically have an ethernet connection, and USB-Ethernet adapters are cheap and robust.
Additionally, a laptop's battery will function as a built in backup power supply, so your VPN endpoint can persist through short power outages.

I **strongly** reccomend keeping your VPN peer on it's own dedicated machine. This will make it easier to segment from the rest of your services, and prevent you from having to forward any traffic from your router to another machine with sensitive data or content.

Choosing a Server OS is outside the scope of this article.
I reach for [Debian](https://www.debian.org/) for projects like this because it's boring and stable.

### Software

We'll be using [wireguard](https://www.wireguard.com/#conceptual-overview) for this project. You can read more about it at that link if you're a big ol' nerd, but the practical reasons to use it are:

- Wireguard has easy to use first-party client software on MacOS, Windows, iOS, and Android - Easy for your whole household to access.
- Wireguard has a minimal attack surface - only one port is open, and it will only respond to clients with a known key. Effectively, it is invisible to passive scanners (unlike OpenVPN, or commercial VPNs).
- Wireguard is redistributed as part of the [PiVPN project](https://github.com/pivpn/pivpn), which simplifies administration

#### Server

1. [PiVPN](https://www.pivpn.io/)

#### Client

1. Wireguard Client

- for [MacOS](https://apps.apple.com/us/app/wireguard/id1451685025)
- for [Windows](https://download.wireguard.com/windows-client/)
- for [iOS](https://apps.apple.com/us/app/wireguard/id1441195209)
- for [Android](https://play.google.com/store/apps/details?id=com.wireguard.android&hl=en)
