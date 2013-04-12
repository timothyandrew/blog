---
layout: post
title: "Changing the Server Timeout on EngineYard"
date: 2013-04-12 22:29
comments: true
categories: [engineyard, rails, unicorn, timeout, ruby, server, nginx]
---

While working on [survey-web](http://github.com/c42/survey-web) today, we were stuck for a really long time trying to figure out this problem.

Unless otherwise specified, image uploads while adding a response are capped at 5MB per image.
Adding a larger image (like this 20MB image) should result in a validation error showing up.

![Validation Error](/images/2013-04-12-image-too-big.png)

On production, we'd see this.

![Production](/images/2013-04-12-502.png)

After a *lot* of digging, including looking at Carrierwave (and [Backgrounder](https://github.com/lardawge/carrierwave_backgrounder)), delayed_job server logs, and our controller logic pretty closely, we noticed in `production.log` that Rails was sending down a `200`, but the browser was recieving a `502`.

`unicorn.log` showed that a worker process was being killed with a `SIGIOP` whenever the error page showed up.

Only then did we realise that the worker was being killed around 60s every time. It had to be a timeout issue.

On EngineYard, the unicorn config already had:

```ruby
# sets the timeout of worker processes to +seconds+.  Workers
# handling the request/app.call/response cycle taking longer than
# this time period will be forcibly killed (via SIGKILL).  This
# timeout is enforced by the master process itself and not subject
# to the scheduling limitations by the worker process.  Due the
# low-complexity, low-overhead implementation, timeouts of less
# than 3.0 seconds can be considered inaccurate and unsafe.
timeout 180
```
The server didn't seem to be following this configuration.

After a fair bit of googling and help from the `#engineyard` IRC channel, this is what we did to fix it.  
Add the following lines to `/data/nginx/nginx.conf` inside the `http{}` block (replacing 300 with the timeout you need).

```nginx
client_header_timeout 300;
client_body_timeout 300;
send_timeout 300;
```

And restart nginx/unicorn with

```bash
$ sudo /etc/init.d/nginx reload
$ /engineyard/bin/app_<app_name> reload
```
