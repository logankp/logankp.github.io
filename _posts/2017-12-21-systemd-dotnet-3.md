---
title: Creating a socket-activated systemd service
layout: single
categories: [linux,systemd,dotnet]
comments: true
---

For the third tutorial in [this]({{ site.baseurl }}{% post_url 2017-09-18-systemd-dotnet-1 %}) [series]({{ site.baseurl }}{% post_url 2017-09-20-systemd-dotnet-2 %}), you will learn how to create a socket-activated daemon in C# using .NET Core 2.0. Additionally, we will add systemd notify support to our first program.  [Socket activation](http://0pointer.de/blog/projects/socket-activation.html) allows a daemon to start only when it is requested (connected to). In this instance, systemd sets up and listens on the socket and passes the socket into the daemon when a connection comes in.

In this tutorial, I go over three different ways of using socket activation:
1. The listening socket is passed in and the daemon accepts it and writes a message.
2. The connection is accepted by systemd and the connection socket is passed in to the daemon. The daemon simply writes to the file descriptor of the socket (3 in this instance).
3. The same as 2, but the socket is passed in on stdin/stdout.

This project utilizes an excellent third-party library called [Tmds.Systemd](https://github.com/tmds/Tmds.Systemd).

## Prerequisites

This tutorial assumes you have basic knowledge on creating, building, and deploying .NET Core projects.

## Getting started

Clone the project from [my GitHub](https://github.com/logankp/systemd.net-tutorial.git). Open ```Project3/SocketActivation/SocketService/src/Program.cs```. Let's walk through the new parts:

```cs
Socket[] sockets = ServiceManager.GetListenSockets(); //get all sockets passed in from systemd
Socket socket = sockets[0]; //we only care about the first
```

We must then accept the socket:

```cs
Socket acceptSocket = socket.Accept();
```

Signal to systemd that we're ready:

```cs
ServiceManager.Notify(ServiceState.Ready, ServiceState.Status("Socket accepted"));
```

Then open a stream and write a message:

```cs
NetworkStream stream = new NetworkStream(acceptSocket);
using (var writer = new StreamWriter(stream))
{            
    writer.WriteLine("This is a test");
}
```

Now build and run the daemon using ```systemd-socket-activate -l 3000 dotnet <path to daemon>``` and connect to localhost:3000 using telnet or nc. You should see "This is a test" in your telnet/nc session. You can also deploy and use service/socket files to run the daemon. See the [README](https://github.com/logankp/systemd.net-tutorial/blob/project3/Project3/SocketActivation/README.md) for details.

Now open ```Project3/SocketActivation/SocketService-Accepted/src/Program.cs```. As you can see it requires even less work to interact with sockets in this one since systemd has already accepted the socket and passed it in on file descriptor 3:

```cs
FileStream fs = new FileStream(new SafeFileHandle((IntPtr)3, true), FileAccess.ReadWrite);
using (StreamWriter writer = new StreamWriter(fs))
{
    writer.WriteLine("This socket was already accepted");
}
```
Why fd 3? Systemd uses the first available file descriptor (after stdin, stdout, and stderr). For more low-level details, see https://www.freedesktop.org/software/systemd/man/sd_listen_fds.html.

Run using ```systemd-socket-activate -a -l 3000 dotnet <path to daemon>``` and again use telnet/nc to connect to it.

For the last example it's even easier. Open ```Project3/SocketActivation/SocketService-Inet/src/Program.cs```

This one consists of only three lines of code:

```cs
var stream = Console.OpenStandardOutput();
using (var writer = new StreamWriter(stream))
{
    writer.WriteLine("This socket was passed on stdin/stdout");
}
```

Run using ```systemd-socket-activate -a --inet -l 3000 dotnet <path to daemon>``` and connect to it like usual.


### Wrapping up

That's it! These examples were extremely simple and were meant to show the basics of using socket activation with .NET. For more details on deployment and running using systemd, please see the [Project 3 Readme](https://github.com/logankp/systemd.net-tutorial/tree/project3/Project3/SocketActivation).
