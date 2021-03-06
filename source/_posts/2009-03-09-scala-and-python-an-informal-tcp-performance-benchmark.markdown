---
layout: post
comments: true
title: "Scala and Python: An informal TCP performance benchmark"
date: 2009-03-09 00:00
categories: [python, scala, java, jvm, performance, tcp, programming]
toc: true
---

# Introduction

I've been using [Python][] in a large-scale, high-throughput,
high-availability network application. Scalability is an issue: We
ultimately have to be able to process a large number of requests a second.
The [JVM][] *seems* easier to scale than CPython, at least for what we're
doing; it has real threads, for instance, instead of the crippled threading
in CPython. But our code base is already large, it's almost entirely
Python, and it makes heavy use of the [Twisted Python][] libraries.

This article describes a small series of benchmarks I ran, to test how many
requests per second I could process using deliberately naive servers
written in Python and [Scala][]. I chose Scala, rather than Java, because:

* Like Java, Scala compiles directly to JVM byte code.
* The Scala language is, in my opinion, a much better language than Java.
  (Having written Java exclusively for nearly nine years, I have some
  experience on which to base that assessment.)
* It's easier and faster to write concurrent programs using the
  [Scala Actor library][] than it is to use the [java.util.concurrent][]
  library.

These test programs are deliberately simple-minded. They accept
incoming socket connections and send back canned HTTP results.

<!-- more -->

# The Programs

I wrote six programs:

* A Scala server that dispatches each incoming connection to a
  pool of actors.
* A simple Python TCP server, written using the popular Twisted
  Python framework's `twisted.internet` API.
* A Twisted-based Python web server written using Twisted's `twisted.web`
  framework, which provides a complete web server framework.
* A Python TCP server that uses a Python Actor library called [dramatis][].
  The dramatis framework is still alpha-quality, so its inclusion here is
  dubious. However, I included it primarily because it allowed for an easy
  translation from the Scala Actor-based server to a Python Actor-based
  server.
* A Python server that uses the standard Python [SocketServer][] API.
* A Python server that uses [fapws3][], a web server framework based on
  [libev][].

The Twisted servers both use a special [ReactorFinder][] module that
attempts to find the best Twisted reactor module for the platform. For
Linux, where I ran my tests, that's the [epoll][] reactor.

The programs are deliberately unoptimized. I'm sure I could get
better performance by profiling and optimizing each one, but my
goal was to see which server worked better *without* tuning. The
pre-tuning benchmarks for these servers provide an interesting
basis for comparing the platforms.

Of course, I'm sure plenty of people will find fault with this
comparison. If you're one of them, then feel free to comment on
this article. (Try to keep it civil.) Or, better yet, do your own
comparisons and publish *your* results.

You can download the programs from the following links:

* [scalaserver.scala][], the Scala Actor-based server.
* [rawserver.py][], the simple Twisted Python TCP server.
* [webserver.py][], the Twisted Python Web-based server.
* [actorserver.py][], the dramatis Actor-based Python server.
* [socketserver.py][], the Socket-Server-based Python server.
* [fapws_server.py][], the *fapws3* Python server.

# The Test Environment

I ran these tests on a Dell Vostro 1700 laptop with two 2.2GHz
Intel CPUs and 3Gb of RAM. The laptop runs
[Ubuntu][] 8.10 (Intrepid Ibex). I used
Python 2.5.2, Scala 2.7.3, and the Sun Java 1.6.0\_10-b33 runtime.

For each server, I used the [ApacheBench][] HTTP benchmarking tool, running
it as follows:

    ab -c 1000 -n 100000 http://localhost:9999/

That is, it issued 100,000 HTTP "GET" requests, 1,000 requests at a
time, using the loopback device. I did not specify `-k`
(KeepAlive), because most of our production servers are using HTTP
between servers, and the servers making the requests aren't using
KeepAlive. (Besides, most of my test servers don't honor
KeepAlive.)

Before running each test, I ran a few thousand requests through the
server *before* running `ab`. For the Scala server, this "priming
of the pump" allows the HotSpot compiler to profile and optimize
the running server. For the Python servers, it likely does nothing
at all.

# The Results

Here are some of the data from the `ab` runs, ranked from most
requests/second to least.

<table border="1">
<tr>
<th rowspan="2">Server</th>
<th rowspan="2">Mean requests per second</th>
<th rowspan="2">Time per request (ms)</th>
<th colspan="4">Percentage of requests served within a certain time (ms)</th>
</tr>
<tr>
<th>50%</th>
<th>75%</th>
<th>90%</th>
<th>100% (longest)</th>
</tr>
<tr>
<td>Scala server</td>
<td>6,220</td>
<td>160.8</td>
<td>81</td>
<td>119</td>
<td>152</td>
<td>9,087</td>
</tr>
<tr>
<td>fapws3-based server</td>
<td>5,733</td>
<td>174.4</td>
<td>20</td>
<td>20</td>
<td>22</td>
<td>16,644</td>
</tr>
<tr>
<td>Python `SocketServer`-based server</td>
<td>4,761</td>
<td>202.2</td>
<td>1</td>
<td>1</td>
<td>2</td>
<td>15,819</td>
</tr>
<tr>
<td>Twisted Python TCP server</td>
<td>3,173</td>
<td>315.1</td>
<td>39</td>
<td>51</td>
<td>53</td>
<td>22,673</td>
</tr>
<tr>
<td>twisted.web` server</td>
<td>1,727</td>
<td>578.9</td>
<td>83</td>
<td>93</td>
<td>94</td>
<td>45,111</td>
</tr>
<tr>
<td>Python actor-based server</td>
<td>1,290</td>
<td>776.3</td>
<td>543</td>
<td>1700</td>
<td>841</td>
<td>93,648</td>
</tr>
</table>

From these results, the JVM seems to be the clear winner. *fapws3*
is the next-fastest server, which is no surprise, since the largest
part of the *fapws3* package is written in C and uses `epoll` and
`libev`. The Twisted-based servers are surprisingly slow in
comparison--which is a shame, since that's what we're currently
using. However, moving to *fapws3* should not be too difficult.

Ultimately, I'd like to be using Scala: It's fast, it's type-safe.
and it runs on the JVM (where Hotspot kicks butt). But given an
already large Python code base, *fapws3* is looking promising.


* * * * *

\[**EDIT**\]

# More Results

## 50 Requests/Second

On the theory that 1,000 requests per second might be introducing
contention, I ran the same tests with 50 requests per second. That
is, I used the following `ab` command line:

    ab -c 50 -n 100000 http://localhost:9999/

The results were interesting:

<table border="1">
<tr>
<th rowspan="2">Server</th>
<th rowspan="2">Mean requests per second</th>
<th rowspan="2">Time per request (ms)</th>
<th colspan="4">Percentage of requests served within a certain time (ms)</th>
</tr>
<tr>
<th>50%</th>
<th>75%</th>
<th>90%</th>
<th>100% (longest)</th>
</tr>
<tr>
<td>Scala server</td>
<td>7,258</td>
<td>6.9</td>
<td>6</td>
<td>7</td>
<td>8</td>
<td>172</td>
</tr>
<tr>
<td>fapws3-based server</td>
<td>6,132</td>
<td>8.2</td>
<td>8</td>
<td>8</td>
<td>12</td>
<td>39</td>
</tr>
<tr>
<td>Python `SocketServer`-based server</td>
<td>4,762</td>
<td>10.5</td>
<td>1</td>
<td>1</td>
<td>2</td>
<td>20,997</td>
</tr>
<tr>
<td>Twisted Python TCP server</td>
<td>3,230</td>
<td>15.5</td>
<td>16</td>
<td>51</td>
<td>16</td>
<td>61</td>
</tr>
<tr>
<td>twisted.web` server</td>
<td>1,732</td>
<td>28.9</td>
<td>30</td>
<td>30</td>
<td>30</td>
<td>77</td>
</tr>
<tr>
<td>Python actor-based server</td>
<td>816</td>
<td>61.2</td>
<td>56</td>
<td>84</td>
<td>93</td>
<td>3,310`</td>
</tr>
</table>

For the Scala and *fapws3* servers, reducing the number of concurrent
connections to 50 increased the total requests/second served. For the
Python Actor-based server, it *reduced* the total. The change had a
negligible effect on the other servers.

[Python]: http://www.python/org/
[JVM]: http://en.wikipedia.org/wiki/Jvm
[Twisted Python]: http://twistedmatrix.com/
[Scala]: http://www.scala-lang.org/
[Scala Actor library]: http://www.scala-lang.org/node/242
[java.util.concurrent]: http://java.sun.com/j2se/1.5.0/docs/api/java/util/concurrent/package-summary.html
[dramatis]: http://dramatis.mischance.net/
[SocketServer]: http://www.python.org/doc/2.5.2/lib/module-SocketServer.html
[fapws3]: http://github.com/william-os4y/fapws3/tree/master
[libev]: http://software.schmorp.de/pkg/libev.html
[ReactorFinder]: ReactorFinder.py
[epoll]: http://linux.die.net/man/4/epoll
[scalaserver.scala]: scalaserver.scala
[rawserver.py]: rawserver.py
[fapws_server.py]: fapws_server.py
[webserver.py]: webserver.py
[actorserver.py]: actorserver.py
[socketserver.py]: socketserver.py
[Ubuntu]: http://www.ubuntu.com/
[ApacheBench]: http://httpd.apache.org/docs/2.0/programs/ab.html
