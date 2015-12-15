---
layout: post
title: "Setting up Octopress on Windows"
date: 2013-08-11 14:36
comments: true
categories: [Chocolatey, Octopress]
published: false
---
##TL;DR Summary

Install [Chocolatey](http://chocolatey.org) if you don't already have it.
```
c:\@powershell -NoProfile -ExecutionPolicy unrestricted -Command "iex ((new-object net.webclient).DownloadString('https://chocolatey.org/install.ps1'))" && SET PATH=%PATH%;%systemdrive%\chocolatey\bin - See more at: http://chocolatey.org/#sthash.y5m7Ah7V.dpuf
```

Install the Octopress Chocolatey package.

```
cinst Octopress [destination-folder]
```

Your done.

<!--more-->

##Setting up Octopress
Octopress has some great [documentation](http://octopress.org/docs/setup/) on getting
