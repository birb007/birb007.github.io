---
layout: post
title: "Operating a Tor Relay"
date: 2020-06-06
---

# Prelude

After watching society crumble in the comfort of quarantine, I was bored. Therefore, I chose to operate a Tor [relay](https://community.torproject.org/relay/). Initially, I wanted to operate an exit node because it looked more exciting, I would be part of the barrier between the Tor network and the clearnet but the prospect of having my door kicked in by [law enforcement](https://community.torproject.org/relay/community-resources/eff-tor-legal-faq/) or dealing with thousands of [legal notices](https://www.copyright.gov/legislation/dmca.pdf) dissuaded me. Instead, I opted to operate a middle relay node because the risk of any legal implication was low and I can still contribute to the Tor network.

I will discuss the process behind operating a middle relay (to learn about the Tor protocol itself, refer to a brief [overview](https://2019.www.torproject.org/about/overview.html.en) and [comparison to traditional proxies](https://2019.www.torproject.org/docs/faq.html.en#Torisdifferent)). If you want to operate a relay yourself, consult the [official relay guide](https://trac.torproject.org/projects/tor/wiki/TorRelayGuide).

## Terminology

| Term | | Description |
---- | | ----
Middle relay | : | Midpoint node in a Tor circuit.
Middle probability | : | Probability that a tor client uses the relay as a middle relay.
Guard relay | : | Entry point for tor clients to enter the Tor network.
Guard probability | : | Probability that a tor client uses the relay as an entry guard.

## TL;DR

| Date | Event |
------ | -----
**Jun 06** | Relay began unmeasured phase (little traffic).
**Jun 09** | Relay began remote measurement phase (increased traffic).
**Jun 13** | Relay Guard relay ramp up (reduced traffic).

Operating a Tor relay was simpler than expected as a project for data analysis and server administration - I recommend operating your own relay.

# Technical Requirements

## Networking

- 7,000 concurrent connections.
- 16 Mbps upload and download bandwidth.
- 100GB of in/outbound traffic, monthly.
- Public IPv4/IPv6 address (preferably static).
- 24/7 uptime (reboots and daemon restarts are fine).

## Physical

- < 40Mbps non-exit relay requires 512MB of memory, otherwise 1GB.
- 200MB of free disk space.

# Technical Setup

The Tor project maintains a [list](https://trac.torproject.org/projects/tor/wiki/doc/GoodBadISPs) of ISPs detailing permitted Tor related behaviour and any additional comments. I chose to use [OVH](https://us.ovh.com) because it offered acceptable network connectivity for a reasonable price. Once the VPS became available I authenticated then began setting up the relay. After a bit of administrative hassle (e.g. securing the VPS, configuring tor, etc.), I had a working Tor relay.

```terminal
root@vps-e0d4d314:~# systemctl status tor@default
● tor@default.service - Anonymizing overlay network for TCP
   Loaded: loaded (/lib/systemd/system/tor@default.service; static; vendor preset: enabled)
   Active: active (running) since Sat 2020-06-06 19:09:50 UTC; 19min ago
 Main PID: 7542 (tor)
   CGroup: /system.slice/system-tor.slice/tor@default.service
           └─7542 /usr/bin/tor --defaults-torrc /usr/share/tor/tor-service-defaults-torrc -f /etc/tor/torrc --RunAsDaemon 0
```

It is really important to appropriately secure the server because it will be servicing thousands of connections to and from unknown hosts. A relay is a worthwhile target because it can assist in [correlation attacks](https://www.ohmygodel.com/publications/usersrouted-ccs13.pdf).

# Phases
## Unmeasured Phase

**June 06, 2020:**
The relay began its "unmeasured phase" (described in the [lifecycle of a new relay](https://blog.torproject.org/lifecycle-new-relay)) where its bandwidth and stability is verified by the network. The first phase would last several days and it would take several weeks for the relay to be in full use. However, the relay was already recorded in the [Tor Atlas](https://metrics.torproject.org/rs.html#details/730E0D04D90CC0B15F320F6DFD5DD23752AD52E9).

To improve relay maintenance, I installed [Nyx](https://nyx.torproject.org) which allowed for real time observation of various relay properties (bandwidth, number of connections, memory overhead, etc.). After fiddling around with [`CookieAuthentication`](https://2019.www.torproject.org/docs/tor-manual.html.en) in [`torrc`](https://2019.www.torproject.org/docs/faq.html.en#torrc):

```conf
ControlPort 9051
CookieAuthentication 1
CookieAuthFile /var/lib/tor/control_auth_cookie
CookieAuthFileGroupReadable 1
DataDirectoryGroupReadable 1

DisableDebuggerAttachment 0

DataDirectory /var/lib/tor
```

I was then able to view the `Nyx` dashboard which presents several relay metrics and properties.

![Nyx interface](/blog/assets/images/tor-relay-nyx.png)

`Nyx` said the relay was transferring a whopping 420bps! The low bandwidth is the result of a 20KB cap applied during this phase. As an aside, the relay bootstrap phase was recorded in the logs.

```terminal
 │ 19:09:52 [NOTICE] Bootstrapped 100% (done): Done
 │ 19:09:51 [NOTICE] Bootstrapped 95% (circuit_create): Establishing a Tor circuit
 │ 19:09:51 [NOTICE] Bootstrapped 90% (ap_handshake_done): Handshake finished with a relay to build circuits
 │ 19:09:51 [NOTICE] Bootstrapped 75% (enough_dirinfo): Loaded enough directory info to build circuits
 │ 19:09:51 [NOTICE] Bootstrapped 15% (handshake_done): Handshake with a relay done
 │ 19:09:51 [NOTICE] Bootstrapped 14% (handshake): Handshaking with a relay
 │ 19:09:51 [NOTICE] Bootstrapped 10% (conn_done): Connected to a relay
 ...
 │ 19:09:51 [NOTICE] Bootstrapped 5% (conn): Connecting to a relay
 ...
 │ 19:09:47 [NOTICE] Bootstrapped 0% (starting): Starting
 ```

During bootstrap, the relay builds four circuits to estimate bandwidth. Looking at the connections page confirms there were several active connections.

**June 07, 2020:**
During the next day, an increase in bandwidth from the 20KB cap (700KBps down, 500KBps up) was recorded. In the unmeasured phase the relay was compared to other prospective relays, and measured by the bwauths. There was evidence of additional self-testing in the `tor` logs.

```terminal
│ 07:30:56 [NOTICE] Performing bandwidth self-test...done.
```

At time of writing, the relay advertised a bandwidth of 1.98MiB/s and was supporting around 1000, respective, in/outbound active connections. The assignment of the [`Fast`](https://gitweb.torproject.org/torspec.git/tree/dir-spec.txt#n2633) flag was evidence the relay was actively compared to other prospective relays.

> "Fast" -- A router is 'Fast' if it is active, and its bandwidth is either in the top 7/8ths for known active routers or at least 100KB/s.

I was rather perplexed at this stage, the unmeasured phase was supposed to last several days and impose a bandwidth cap of 20KBps. However, the relay was routing above 20KBps with around 24 hours uptime. Rationalising this, I think each individual connection is capped to 20KBps but thousands of bwauths concurrently evaluate the relay (if I'm mistaken, let me know).

**June 08, 2020:**
The relay was supporting around 2000, respective, in/outbound active connections.

## Remote Measurement Phase

**June 10, 2020:**
The unmeasured phase ended on June 9th - evidenced by a removal of the [`Unmeasured`](https://gitweb.torproject.org/torspec.git/tree/dir-spec.txt#n2446) consensus value.

> The "`Unmeasured=1`" value is included in consensus generated with method 17 or later when the '`Bandwidth=`' value is not based on a threshold of 3 or more measurements for this relay.

During the remote measurement stage, the relay began to receive, incrementally, better bandwidth estimates as it was compared to other relays. In order to record better bandwidth estimates, the relay participated in Tor circuits which pushed traffic. The amount of traffic pushed through the relay increased as the estimated bandwidth increased. Consequently, the middle relay probability increased from 0.0074% (June 9th) to 0.0176% with an advertised bandwidth of 4.86 MiB/s.

I opted to add IPv6 support to the relay which would make the relay more accessible (especially if it were to become an [Entry Guard](https://2019.www.torproject.org/docs/faq#EntryGuards)). To enable IPv6 support, the following lines were added to `torrc`:

```conf
ORPort [2001:41d0:801:2000::272e]:443
ExitPolicy reject6 *:*
```

For the changes to take effect, the tor service had to be restarted. Restarting the tor service was an opportune moment to remove the bogus contact information (it was serving no purpose and annoying me). After a couple hours, the relay was assigned the `ReachableIPv6` flag as confirmation that IPv6 support was successfully added.

For longitudinal traffic observation, I scheduled [`vnstati`](https://humdi.net/vnstat/) to produce a weekly summary of the VPS traffic using [`systemd`](https://wiki.archlinux.org/index.php/Systemd/Timers) (note: there will be spikes where [`UnattendedUpgrades`](https://wiki.debian.org/UnattendedUpgrades) executes). The scheduled script is listed below.

```shell
#!/bin/bash

if [[ $(id -u) != 999 ]]; then
	echo "Not vnstati-sched, exiting..."
	exit 1
fi

SCRIPT_NAME=$(basename -- "$0")

output=$(date +%Y-%m-%d)
mkdir $output
cd $output
vnstati -d -i ens3 -o "${output}_daily.png"
vnstati -vs -i ens3 -o "${output}_summary.png"
```

I toyed with the idea of setting up a separate interface for tor to allow more accurate bandwidth statistics but the effort required, for a minor improvement, deterred me. At time of writing, the relay was relaying around 30Mbps across 5000, respective, in/outbound connections.

**June 12, 2020:**
Relay was assigned the [`Stable`](https://gitweb.torproject.org/torspec.git/tree/dir-spec.txt#n2613) flag.

> "Stable" -- A router is 'Stable' if it is active, and either its Weighted MTBF is at least the median for known active routers or its Weighted MTBF corresponds to at least 7 days. Routers are never called `Stable` if they are running a version of Tor known to drop circuits stupidly. (0.1.1.10-alpha through 0.1.1.16-rc are stupid this way.)

The `Stable` flag was assigned using the second definition because the relay had only been online since 19:00, June 6th - 5 days. The middle relay propability increased to 0.0443% as did the volume in traffic. The relay was handling a similar number of connections to June 10th (2368 inbound, 3033 outbound). It was possible to see if any Guard relays were using IPv6, via the command below.

```terminal
root@vps-e0d4d314:~# ss -H6 state established "( sport = :http or sport = :https )" | wc -l
6
```

The relay was engaging over 6, active, IPv6 connections.

## Guard Relay Phase

**June 13, 2020:**
The relay advertised bandwidth increased to 12.12 MiB/s (15.63 MiB/s burst) as the middle relay probability increased to 0.0538%. Additionally, the relay was assigned the [`HSDir`](https://gitweb.torproject.org/torspec.git/tree/dir-spec.txt#n2668) flag which allowed the relay to become a DNS server, of sorts, for [onion services](https://community.torproject.org/onion-services/overview/).

> "HSDir" -- A router is a v2 hidden service directory if it stores and serves v2 hidden service descriptors, has the `Stable` and `Fast` flag, and the authority believes that it's been up for at least 96 hours (or the current value of `MinUptimeHidServDirectoryV2`).

Moreover, the relay was assigned the [`Guard`](https://gitweb.torproject.org/torspec.git/tree/dir-spec.txt#n2383) flag. Guard status allowed tor clients to enter the Tor network through the relay and signaled the start of the guard phase.

> "Guard" -- A router is a possible Guard if all of the following apply:
>
> - It is `Fast`,
> - It is `Stable`,
> - Its Weighted Fractional Uptime is at least the median for "familiar" active routers,
> - It is "familiar",
> - Its bandwidth is at least `AuthDirGuardBWGuarantee` (if set, 2 MB by default), OR its bandwidth is among the 25% fastest relays,
> - It qualifies for the `V2Dir` flag as described below (this constraint was added in 0.3.3.x, because in 0.3.0.x clients started avoiding guards that didn't also have the `V2Dir` flag).

The flag itself is defined by:

> "Guard" if the router is suitable for use as an entry guard.

During the guard relay phase: the middle probability of the relay will drop as the guard probability increases and all clients stop using the relay as a middle relay. Consequently, after the assignment of the `Guard` flag the middle probability decreased to 0.0437% within 8 hours whereas the guard probability increased to 0.0073%. However, the number of connections did not see an immediate drop - I suspect existing connections are unaffected. The volume of traffic through the relay was anticipated to decrease over the next several weeks until the next [guard rotation](https://blog.torproject.org/research-problem-better-guard-rotation-parameters). Furthermore, I expected the number of IPv6 connections to increase because relays are unlikely to be subjugated to restrictive firewalls compared to tor clients'.

However, the relay's exposure to the clearnet increased because clients would connect directly to the relay itself as opposed to by proxy relays. The increased exposure was expected to increase the number of attacks toward the relay - putting my own privacy at risk - so I would regularly check the [`fail2ban`](https://www.fail2ban.org/wiki/index.php/Main_Page) logs and the tor logs for malicious activity. It should be noted that during the initial administrative work, I had setup [persistent](https://packages.debian.org/buster/iptables-persistent) [`iptables`](https://wiki.debian.org/iptables) rules to only allow 22 (ssh), 80 (tor dirport), and 443 (tor orport). If you are operating your own relay, I strongly suggest you take the time to secure your host.

**June 14, 2020:**
Advertised bandwidth increased to 12.13 MiB/s. Further, the middle probability decreased to 0.0236% whereas the guard probability increased to 0.0337%. The number of connections to the relay had not changed significantly since the 13th.

It had been 9 days since the relay first came online and `vnstati` had produced a traffic summary over the last week. The local data and remote consensus data presented an opportunity to inspect traffic behaviour and consensus values as the relay traversed through multiple phases. Firstly, the network consensus values are inspected because they were causal to the relay traffic (with respect to the network).

![atlas network visualisation](/blog/assets/images/tor_relay_atlas_network_visualisation.png)

The rightmost graph, the network consensus visualisation, clearly shows each relay phase. Between Jun 6th (unlisted at the far left) and Jun 9th there is very little traffic, at some point during Jun 8th it appears the relay began the remote measurement phase. The relay is restricted to < 700 KiB/s.

The remote measurement phase shows a significant increase in middle relay probability, reaching a maximum of 0.0538% on Jun 14th. Further, the network consensus weight increases as the relay bandwidth is measured and confirmed against the advertised bandwidth. The increase in middle probability during the remote measurement phase is somewhat linear and positively correlates to the sudden increase in bytes read/written. During the remote measurement phase, the relay increased from 659.2 KiB/s to 5.877 MiB/s (892% increase) over the 4 - 5 day, remote measurement period.

Within the last 24 hours, the relay had began the guard phase. As expected, the guard probability rapidly increased to 0.0334% from 0.0000% within 4 hours. The increase in guard probability negatively correlates with middle probability. The relay traffic did not see a significant change in volume.

The latest local visualisations of relay traffic are shown below.

<p align="center">
<img alt="daily traffic visualisation" src="/blog/assets/images/tor_relay_2020_06_15_summary.png">
<img alt="daily traffic visualisation" src="/blog/assets/images/tor_relay_2020_06_15_daily.png">
</p>

The relay had transferred 3.9TB in total since the 6th of Jun. However, between the 12th and the 14th, 2.76TB had been transferred. The increase in traffic coincides with the rapid increase in middle probability. Looking at the daily graph, the incremental increase in traffic is clear. Hourly traffic measurements show the traffic peaks between 11:00 - 15:00 (UTC), longer analysis is required to build a traffic schedule.

That concludes the interesting parts of running a relay. The next phase will be becoming a steady Guard but that will take a couple months. Within that period the middle probability will drop in favour of guard probability and the relay will receive more direct connections. In the meantime, relay will continue to collect information for my own curiosity, I hope this post was interesting.

<div style="width:100%;height:0px;position:relative;padding-bottom:56.250%;"><iframe src="https://streamable.com/e/shanqw?autoplay=1" frameborder="0" width="100%" height="100%" allowfullscreen style="width:100%;height:100%;position:absolute;left:0px;top:0px;overflow:hidden;"></iframe></div>
