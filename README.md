# async_comparisons
Simple servers using different async mechanisms

Writing a single-user server is pretty easy. But real-life servers have to handle multiple users at the same time. A number of different mechanisms have been invented over the years to solve this problem, and Python has libraries (in the stdlib or on PyPI) to help out with most of them. But they're all different enough that the knowledge you gain using one mechanism doesn't always apply smoothly to the others.

Making things worse, most tutorials build servers that avoid all the hard problems. In particular, an echo server that just echoes back whatever the client sends doesn't have to deal with delimiting separate messages on the stream, or with treating bytes as text, or with communicating between different users. But real-life servers often need these. So, I'll provide examples that add them progressively, for each different mechanism.

Requirements
============

To keep things simple, you can use telnet or netcat as a client for any of the servers. You will get different behaviors for the two--Netcat usually sends a line at a time instead of a character at a time, lets you quit quickly with a simple `^C` instead of having to hit `^]` and then typing `exit`, can half-shutdown the socket (by typing `^D` (Unix) or `^Z` (some Windows versions)), etc. You may actually want to try both (and try different options for them) to see how they affect the server.

If you're on Unix (including Mac OS X, Linux, etc.), just start the server in one terminal (e.g., `python3 single.py`), then, in another terminal, run `nc localhost 12345` or `telnet localhost 12345`. 

Windows doesn't come with Netcat, and there are a few different versions. Try [netcat for Windows](https://eternallybored.org/misc/netcat/): extract it, put `nc.exe` somewhere on your `%PATH%`, and then you can just `nc localhost 12345`. Windows does support a builtin telnet, but in recent versions you have to [install it](https://technet.microsoft.com/en-us/library/cc771275(v=ws.10).aspx) before you can use it; then it's just `telnet localhost 12345`. There are also a zillion graphical telnet clients for Windows, if you want to use one of them instead.

All of the servers are written for Python 3.5. The `await` server won't work with anything older; the `asyncio` server requires at least 3.4 (or 3.3, with the `asyncio` backport from PyPI); the rest should be portable back to 3.2 or 2.6, but you may have to make some changes for some.

Because the servers are servers, you may have to configure your firewall to allow them to work. On most Unix boxes, this won't be a problem (they may not allow connections from other machines, but from localhost it'll be fine). On recent Windows, you'll probably get a popup asking for permission the first time you run. Accept it. (You can choose the safer/more restrictive/"public computer" option if you want.)

Each server that requires a third-party package will say so when it comes up. But if you want to get them all in advance:

    pip install gevent

Some of these may not have wheels yet (as of 16 December 2015, there's at least no wheel for `gevent` for 64-bit Python 3.5 on Windows), so you may need a C compiler. On Mac, the easiest way is to install Xcode out of the App Store. On Windows, you can install MSys or Visual Studio Community Edition, but it may be easier to just download the binaries from [Christoph Gohlke's repository] and follow his directions for installing them.

`single.py`
===========

The first server, `single`, just accepts a single user, echoes whatever he types, then quits when he's done.

`single2.py`
============

The `single2` server just adds logging, but that requires two more features: interpreting bytes as text, and error handling.

`single3.py`
============

The next server adds persistence: the server still only handles one client at a time, but after it's done, it goes on to the next client instead of quitting. If another client tries to connect while the first one is still connected, he'll have to wait around, but at least he'll eventually get served.

`single4.py`
============

The next server adds nothing, but refactors to the code to make it easier to add new features. 

`single5.py`
============

This server just takes the refactoring further, using an OO design similar to some actual server frameworks.

`single6.py`
============

For most servers, you don't want a stream of bytes, you want a stream of messages. But you have to take care of that yourself. So, the first question is: how are the messages delimited? The simplest answer is the one we've done here: each ASCII newline means the end of a messages. (Of course this can be tricky if you need to send commands that include newlines in them, but that's not a problem here.) Other ways to do it are to send fixed-size messages, or to send a fixed-size header before every message that tells you how to figure out long the message will be, or to use some kind of self-bounded string format like JSON (each message ends when the initial opening `{` or `[` is closed or the initial simple token ends).

When people talk about "designing network protocols", that's really all there is to it. Of course there are details, and those details can get pretty complex if you want them to. (Go look up HTTP--even before it was split into 6 separate specs, [RFC 2616](https://tools.ietf.org/html/rfc2616) was 176 pages long, and you have to read at least three other RFCs to implement it.) But a simple protocol can be described in one line of English and implemented in a few lines of code.

We're still not doing anything particularly smart with the messages--we just send them back, as-is, with newlines between them. In fact, if you've been using netcat as a client, you may not notice any change (except the log output), because by default netcat already buffers your input line by line. But if you're using telnet, it defaults to character by character, so you will see a big change.

By the way, if you're on Windows, the lines you send end with a carriage return and a newline (`'\r\n'`), not just a newline. Since `'\r\n'` itself ends with `'\n'`, and we don't really care about the content of the messages except to send them right back (to the same Windows telnet program that likes those carriage returns), that's fine for now. But many real servers have to deal with this. Some protocols require `'\r\n'` as a delimiter; some allow either `'\r\n'` or `'\n'` (which can get tricky for some kinds of messages), etc. One of the many ways Windows makes the world more fun.

`single7.py`
============

Now we turn our messages into actual commands, with an IRC-like protocol, and add a dynamic dispatcher. Now, you can add new commands to the server just by defining new methods.

`single8.py`
============

This version adds some simple user-input sanitization. It then shows why simple sanitization is a problem, and why you should look for a more systematic solution to your problem. Since it's probably been at least 4 seconds since someone linked to [Little Bobby Tables](https://xkcd.com/327/), go read that. (The more systematic solution in that case is to use parameterized SQL statements, where you pass the arguments as arguments, instead of building a SQL string out of them.)

`single9.py`
============

It's refactoring time again. A simple step is to split the `Connection` object off from the `Server` object, so it can now store connection-specific state, like the socket and the address. The result is a common design for simple frameworks.

The abstraction between `single8` and `single9` is the same one that the stdlib's [`socketserver`](https://docs.python.org/3/library/socketserver.html) module does. And it's worth reading through that module's documentation to see some of the ways you can build on this design. (Note that `socketserver` doesn't include all of the intermediate steps we did before here; its notion of "request" is "all of the data that comes in for the lifetime of a socket", which can be confusing. But it, and the related modules, later show how to break those requests down into smaller requests, at which point it converges back toward our design.)

`single10.py`
=============

And for our final refactoring: Often you can split up the work to be done as a chain of transformations: First you break the buffers down into lines, then you decode each line, then you process and dispatch each decoded line. And on the way back up, you send a response, which gets encoded, which gets wrapped up as a line and crammed onto the send buffer.

Many of these pieces can be reusable parts of a framework. For example, line splitting and joining is used by a wide range of protocols from IMAP to IRC, and they all do it pretty much the same way, so why not use the same code for all of them?

If we split the work up into a chain of protocol objects, each of them can be simple, and most can be reusable.

This also allows us to override a small piece of functionality by just subclassing one simple class in the chain (e.g., if we want line splitting with `'\r\n'` instead of `'\n'`, we just subclass `LineProtocol` and use our subclass in place of the original in the chain).

It also allows us to stack things that may never have been intended. For example, you can run JSON-RPC directly over a TCP socket, or you can run it over HTTP. If you want the same for your new protocol, just throw an HTTP protocol object on the front of the chain, and none of the rest of your code has to change.

If you've ever wondered why frameworks like Twisted are so big and complicated, it's because they contain tons of pre-build, easily-customizable, chainable protocols. (Some frameworks split things up into protocols and transports, which makes things a little cleaner, but I didn't get into that here.)

Obviously, if you're considering building scaffolding as heavy-weight as this example, you probably want to go look at something like Twisted instead, that's already made all the tricky design decisions, probably a lot better than you or I will on the spur of the moment with only a single application in our head, and provides ready-made pieces that you can slot in so you don't have to build them yourself.

`thread4.py`
============

We can't go any farther with a single-connection server, because the only thing left to deal with is communication between users. The simplest way to handle multiple connections is with a thread per connection. That's literally a one-liner change, and now we can handle hundreds of simultaneous clients. (Of course when we go over the limit and run out of bandwidth, file handles, memory for thread stacks, or some other resource, we're going to fail badly and just kill the server, but let's not worry about that for now.)

One thing: If you want to shut down your server, but people are still connected, what should happen? The simplest thing to do is just kill the threads, which kicks off the users instantly. That's what the `daemon=True` in the `Thread` constructor does. This can be unfriendly to your users, of course; the friendliest thing to do is to stop accepting connections (by exiting the loop around `accept` and closing `ss`), then `join` all the threads you created, which lets them all run until they get closed. But then one uncooperative (or just clueless) user can prevent you from ever shutting down. Ideally, you want to signal all the connections in some way, give them a few seconds to alert their users and start doing a graceful shutdown, and then kill whichever ones didn't finish shutting down, but signaling the connections is the same problem as communicating between connections, which is a big problem we're putting off until a later step.

It should be pretty obvious how to extend the same thing up the chain from `single4` to `single10`, so I won't do all those steps. 

If you read about `socketserver` above (in `single9`), it provides a pretty simple mixin that allows the application developer to create a threaded server in one line, with no change to the request-handling logic in the connection (aka handler) object. But before you go too far within trying to factor out the asynchronocity mechanism, read the rest of this document; most of the alternatives are much harder to just plug in without changing the connection object. And the ones that are easy to plug in (like forking, as seen in `socketserver`), you usually don't want.

`select3.py`
============

There's a completely different way to write a single-connection server. The problem is that your program is spending all of its time blocked on each `ss.accept` (if you're waiting for a connection) or `cs.recv` (if you're servicing a connection), so it can't do anything else. But if you could block on a bunch of sockets at once, instead of a single socket, you could handle listening for a new connection and servicing two connections at the same time.

And of course you can block on a bunch of sockets at once, or I wouldn't have brought it up. The trick is to use non-blocking sockets, and then use a "selector" or "multiplexor" object that takes a list of sockets and blocks until one of them is ready for you.

Of course this requires you to reorganize your code. For one thing, you can't have an outer loop around `ss.accept` and an inner loop around `cs.recv`; they have to be the same loop, with an `if` switching between the two behaviors. But, more importantly, look at all that state we're keeping in local variables, like `cs` and `buf` and so on, knowing that it'll be the same after `cs.recv` as it was before. Now, between one `recv` and the next, there may have been `recv`s on 20 other sockets, and a couple `accept`s, and a few clients closing. So, all those local variables have to be moved into a dictionary that maps sockets to "client states".

Once you read that, you should immediately be thinking about how to adapt the `single4`-`single10` designs to fit the selector model. Which is great, but we'll make some other changes before we start reproducing that work.

There's a serious problem with `send` in this simple server, as explained in the code. But the solution to it is so closely tied to the way we're going to handle protocols that we'll put it off until then.

One last note: I'm using the [`selectors`](https://docs.python.org/3/library/selectors.html) module from the stdlib, which is the easiest thing to use if you're not familiar with this kind of code. But at some point, you should look at the lower-level [`select`](https://docs.python.org/3/library/select.html) module--it's a lot closer to what C and other languages provide, and, even in Python, you're going to find a lot more `select`-based than `selectors`-based code out there.

`select6.py`
============

The steps to extend on from 4-6 are very different for selectors than they were for single-client and threaded servers--although they re-converge pretty nicely later. You may want to try following the same steps yourself, but I'm going to collapse them into this one, where we get to and solve the big issue.

If you look at `single6`, the way we switched from handling a stream of bytes to a stream of protocol messages was by looping around `recv` and appending to a buffer. So, how are we going to do that here? Obviously, we need to store a buffer for each socket. Then, for each `recv`, we handle whatever commands we have on that socket's buffer, and move on to the next socket.

This is where `send` comes in. The socket can try to `send` immediately, but if it fails, or only sends some of the data, it needs to store the rest in a second buffer, and tell the redirector that we now want write notifications as well as read notifications on that socket. Then, once we've drained the write buffer, we turn those write notifications back off (because otherwise, we'll get notified every time through the loop that the socket is ready for sends, even though we have nothing to send, which is pretty wasteful).

We're already storing the friendly address in the selector's per-socket data, so where do we store these read and write buffers? Simple; just store a list holding the friendly address and the buffers. (Of course this should make you think of using a connection object, which gets us on track toward a `single9` equivalent.)
