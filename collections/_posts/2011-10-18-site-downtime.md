---
layout: post
title: "Some brief downtime"
categories: blog
---

My blog was off the air for a short while as I struggled with the new router's settings. I sincerely apologize to all my readers (bored search engine crawlers). The router in question (a D-LINK DIR-655, hardware version B1, firmware 2.00) does a fine job router-wise, but I've seen some strange quirks that I want to shine a light on.

The first is the webinterface, which appears to roll a dice to decide whether it'll correctly log you in or not, and then provides no way of logging back out. In fact, it appears to bind your session to you IP or MAC address _only_, as clearing cookies or even switching browsers has no effect on your session state. The only way to "log out" is to do nothing and let the session expire. I'm not a security expert, but this seems downright stupid to me.

The second weirdness comes from port forwarding and is the reason for my downtime. You see, the DIR-655 provides two ways of forwarding ports:

*   "Virtual Server", which lets you forward single external ports to different specified internal ports.
*   "Port Forwarding", which lets you forward ranges of external ports to identical internal ports.

Naturally, for my webserver which runs this blog, I went into Port Forwarding to forward port 80\. This didn't work, unfortunately, and I couldn't figure out why. I did a portscan from an internet service and it said port 80 was open. So why wouldn't my website load? I was beginning to think the problem was with my server, and maybe I needed to do something with IIS because the switch in routers had caused our external IP address to change. Or something. I'm no expert in this area. Anyway, after some more fruitless research (desperate googling), I decided to see if I could remote-desktop into my server. And strangely enough, this _did_ work. But how could this be? Well, I'm forwarding a random external port to port 3389 so I can remote into my server when I'm at school. On this new router, I set this up using the Virtual Server option. Wait a minute... I quickly went into Virtual Server again, forwarded port 80 to port 80, and lo and behold: one website back in the air! I still don't know why the regular Port Forwarding option doesn't work, but for now this workaround is good enough.
