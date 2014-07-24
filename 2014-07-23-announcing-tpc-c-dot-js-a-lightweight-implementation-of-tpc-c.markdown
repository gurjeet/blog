---
layout: post
title: "Announcing TPC-C.js; A Lightweight Implementation of TPC-C"
date: 2014-07-23 22:53:08 +0000
comments: true
categories:
- Postgres
- Performance
- Database
- Benchmark
---

I am glad to announce the [beta] release of TPC-C.js, which implements one of
the most popular database benchmarks, [TPC-C]. It's not a coincidence that today
is also the [22nd anniversary] of the TPC-C benchmark.

It currently supports [Postgres] database, but can be easily extended to test
other database systems.

[beta]: https://github.com/gurjeet/DBYardstick/tree/v0.1.0/TPC-C
[TPC-C]: http://www.tpc.org/tpcc/default.asp
[22nd anniversary]: http://www.tpc.org/information/sessions/sigmod/sld007.htm
[Postgres]: http://www.postgresql.org/

You might ask *"Why another TPC-C implementation when we already have so many of
them?""*

Short answer: This one is very light on system resources, so you can

1. Run the benchmark strictly adhering to the specification, and
2. Invest more in database hardware, rather than client hardware.

Long answer: It's covered in the [Motivation] section of TPC-C.js, which I'll
quote here:

[Motivation]: https://github.com/gurjeet/DBYardstick/tree/master/TPC-C#motivation

> Motivation
> ==========
>
> The TPC-C benchmark drivers currently available to us, like TPCC-UVa, DBT2,
> HammerDB, BenchmarkSQL, etc., all run one process (or thread) per simulated
> client. Because the TPC-C benchmark specification limits the max tpmC metric
> (transactions per minute of benchmark-C) from any single client to be 1.286 tpmC,
> this means that to get a result of, say, 1 million tpmC we have to run about
> 833,000 clients. Even for a decent number as low as 100,000 tpmC, one has to run
> 83,000 clients.
>
> Given that running a process/thread, even on modern operating systems, is a bit
> expensive, it requires a big upfront investment in hardware to run the thousands
> of clients required for driving a decent tpmC number. For example, the current
> TPC-C record holder had to run 6.8 million clients to achieve 8.55 million tpmC,
> and they used 16 high-end servers to run these clients, which cost them about
> $ 220,000 (plus $ 550,000 in client-side software).
>
> So, to avoid those high costs, these existing open-source implementations of
> TPC-C compromise on the one of the core requirements of the TPC-C benchmark:
> keying and thinking times. These implementations resort to just hammering the
> SUT (system under test) with a constant barrage of transactions from a few
> clients (ranging from 10-50).
>
> So you can see that even though a decent modern database (running on a single
> machine) can serve a few hundred clients simultaneously, it ends up serving
> very few (10-50) clients. I strongly believe that this way the database is
> not being tested to its full capacity; at least not as the TPC-C specification
> intended.
>
> The web-servers of yesteryears also suffer from the same problem; using one
> process for each client request prohibits them from scaling, because the
> underlying operating system cannot run thousands of processes efficiently. The
> web-servers solved this problem (known as [c10k problem]) by using event-driven
> architecture which is capable of handling thousands of clients using a single
> process, and with minimal effort on the operating system's part.
>
> So this implementation of TPC-C uses a similar architecture and uses [NodeJS],
> the event-driven architecture, to run thousands of clients against a database.
>
[c10k problem]: http://en.wikipedia.org/wiki/C10k_problem
[NodeJS]: http://nodejs.org/

