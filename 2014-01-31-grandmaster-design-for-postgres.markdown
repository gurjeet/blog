---
layout: post
title: "Grandmaster Design for Postgres"
date: 2014-01-31 03:34:46 +0000
comments: true
published: false
categories: 
---

Have a parent process of Postmaster, called Grandmaster (as in grandparent of
backend processes), that forks the Postmaster, and upon minor version upgrade,
signals the Postmaster to kill itself without killing its children. Then, the
Grandmaster launchesa new Postmaster using newer binaries. The new Postmaster
handles new connection requests, while the orphaned children continue to serve
the connections from before their Postmaster died.

The challenges will be in letting the orphaned children take their commands from
the Grandmaster, and waiting on Grandmaster's exit (not on their Postmaster, as
backends usually do), to die before killing themselves.

PS: Idea conceived in bath tub, when thinking about restart-less major version upgrades.


