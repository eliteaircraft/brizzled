---
layout: post
comments: true
title: "Interpreting Java"
date: 2009-01-08 00:00
categories: [java, python, jython, groovy, programming, scala]
---

In nearly nine years of programming Java, it never really occurred
to me to use it in an interpretive fashion. If I needed to test an
API, I typically wrote a quick and dirty throwaway tester, compiled
it, and ran it. That was my mindset at the time.

However, for the last year, I've been programming Python almost
exclusively, and I find I've grown accustomed to running quick
tests in the [ipython][] command-line
interpreter. Now, with my mindset suitably changed, I want the same
capability when I'm doing Java work.

There are, of course, many Java-based scripting languages, and many
are perfectly suitable for this kind of thing.

For example, I wanted to test a Java JSON library (the one at
[json.org][]). What better way than with
[Jython][]?

    $ jython
    Jython 2.5b0+ (trunk:5882, Jan 8 2009, 12:18:58) 
    [Java HotSpot(TM) Server VM (Sun Microsystems Inc.)] on java1.6.0_03-p3
    Type "help", "copyright", "credits" or "license" for more information.
    >>> from org.json import *
    >>> j = JSONObject({'a' : 1, 'b': [1, 2, 3], 'c': {'x': 0, 'y': 'a'}})
    >>> j
    {"a":1,"b":[1,2,3],"c":{"x":0,"y":"a"}}

That sure beats writing and compiling a tester, just to verify how
something works. Plus, Jython/Python's brevity of syntax is great
for this kind of thing. The {} dictionary in Python translates to a
HashMap. The last line ("`j`") just tells the interpreter to call
Python's `str()` method on "j" -- which Jython translates to a call
to `JSONObject.toString()`, which is exactly what I want.

[Groovy][] also has both power and
syntactic brevity:

    $ groovysh
    Groovy Shell (1.6-RC-1, JVM: 1.6.0_03-p3)
    Type 'help' or '\h' for help.
    -------------------------------------------------------------------------------
    groovy:000> j = new JSONObject(["a" : 1, "b": [1, 2, 3], "c": ["x":0, "y": "a"]]
    ===> {"a":1,"b":[1,2,3],"c":{"x":0,"y":"a"}}
    groovy:000> j
    ===> {"a":1,"b":[1,2,3],"c":{"x":0,"y":"a"}}

And, of course, you can do the same things with [Scala][], my new favorite
language on the JVM.

Frankly, I don't know why I didn't think of using these tools more
when I was programming Java full-time. However, I'll sure use them
*now*, when I find myself working in Java. The ability to run quick
tests without writing a full-blown tester is just too useful to
pass up.

[ipython]: http://ipython.scipy.org/
[json.org]: http://www.json.org/
[Jython]: http://www.jython.org/
[Groovy]: http://groovy.codehaus.org/
[Scala]: http://www.scala-lang.org/
