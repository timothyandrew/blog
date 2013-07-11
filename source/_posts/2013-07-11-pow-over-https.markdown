---
layout: post
title: "Pow Over HTTPS"
date: 2013-07-11 12:26
comments: true
categories: [pow, rails, ruby, https]
---

I use [Pow](http://pow.cx) to manage web servers on my development machine. It works pretty well.
To start my server, I just hit a URL like `http://surveyweb.dev`, which starts the server (if it isn't running) and spins it down automatically in 5 minutes.

It doesn't work over HTTPS by default; here's how you get that done.

```bash
$ gem install tunnels
```

This gem lets you route traffic from one port to another port.

We need to route traffic from port 443, to port 80 (where the Pow server runs).

```
$ sudo tunnels 443 80
```

While the tunnel is open, I can access `https://surveyweb.dev` just fine.

Pow also has a feature where I can access my server from another machine on the LAN using a URL like `http://surveyweb.192.168.1.10.xip.io/` where `192.168.1.10` is the IP address of my machine. Even with the tunnel open, HTTPS doesn't work for this URL.

We need to start another tunnel:

```
$ sudo tunnels 192.168.1.10:443 127.0.0.1:80 # Replace 192.168.1.10 with your IP address
```

And now, both URLs work over HTTPS.
