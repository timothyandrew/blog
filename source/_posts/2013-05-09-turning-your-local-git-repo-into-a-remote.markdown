---
layout: post
title: "Turning Your Local Git Repo Into a Remote"
date: 2013-05-09 20:18
comments: true
categories: [git, howto]
---

Need to pull some changes from a friend's local Git repo without having to push to `origin`? This post will show you how to do that.

You can access a local Git repo using SSH, but setting up keys and such will probably take some time. For a quick-and-dirty solution, HTTP is *much* easier.

On the machine you want to use as the server, navigate to your project and then into the `.git` directory.

```bash
$ cd /path/to/project
$ cd .git
```

Stand up a HTTP server using Python's `SimpleHTTPServer` module. You can use any port number you like.

```bash
$ python -m SimpleHTTPServer 5000
```

You'll need the IP address of this machine as well. (Use `ifconfig`)

Make sure you can access the python server from a browser on the client machine. You should be able to see something like this at `http://ip.address:5000/`

![Python Server Browser Screenshot](images/2013-05-09-python-server.png)

On the client, you should be now able to access the git repo over HTTP as though it were a normal git remote.

```bash
$ git ls-remote http://ip.address:5000
$ git pull http://ip.address:5000 master
```

Add it as a remote to avoid typing out the entire IP each time.

```bash
$ git remote add http://ip.address:5000 local-foo
$ git pull local-foo master
```

