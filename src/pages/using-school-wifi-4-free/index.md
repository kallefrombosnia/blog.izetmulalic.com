---
title: Bypassing school WiFi restrictions
date: '2021-01-10'
spoiler: Old school wifi bypass
---

Back in the day when I was in middle school around 2014 / 2015 school didn't allow us WiFi usage, only heavy nerds with score A+ could gain access to that network.

The network was unlocked with no password but there was a MAC whitelist what means you can connect but if your MAC is not added to wl you couldn't use that network.

At that time I loved challenges so I decided to bypass that restriction.

With a few searches, I found a tool called Busybox (fckn awesome tool) and a terminal for Android phones.

At that time I had Samsung Galaxy S2 (broke ass) which was rooted (tools above don't work without root access)


```sh
# Become root!

$ su 
```


See what MAC address telephone currently uses: 

```sh
$ busybox iplink show eth0
```


How to get the victim MAC address?

Thats was the hardest part but with some reverse engineering I succeeded to get the MAC address which was whitelisted (invited nerd on coffee so I could scan coffee bar wifi network for his MAC address lmao).

So with that information, I could bypass that restriction where XX-es are 12 digit victim MAC address.


```sh
$ busybox ifconfig eth0 hw ether XX:XX:XX:XX:XX:XX
```

After that, with the first command, you can check if spoofing was successful.


```sh
$ busybox iplink show eth0
```


If a new MAC address is equal to the victim's MAC address you can access to the network.

Note: you can't use this if there are 2 same MAC address connected and after a restart of the phone you need to repeat the same process.

This is for educational purposes only, no harm intended.

