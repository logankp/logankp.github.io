---
title: Writing your first .Net Core 2.0 systemd service
layout: single
categories: [linux,systemd,dotnet]
comments: true
---

This tutorial will cover the basics of writing a systemd daemon using .Net Core 2.0 on Linux. For background on writing "new style" (systemd compatible) daemons, see the [documentation](https://www.freedesktop.org/software/systemd/man/daemon.html#New-Style%20Daemons). This tutorial assumes you already have a working .Net Core installation. If not, follow the instructions [here](https://www.microsoft.com/net/core) to install .Net.

## Getting started

Run `dotnet new console -o myfirstdaemon`. This will generate a `myfirstdaemon` directory containing the following files:

```
service.csproj
Program.cs
```

Open the `Program.cs` file with the text editor of your choice. We will now begin working on the logic of our daemon.

1. First we need to set up signal handlers for SIGINT (Ctrl-C) and SIGTERM (sent by systemd upon daemon termination). Add the following lines in Main:

   ```cs
   AssemblyLoadContext.Default.Unloading += SigTermEventHandler; //register sigterm event handler. Don't forget to import System.Runtime.Loader!
   Console.CancelKeyPress += new ConsoleCancelEventHandler(CancelHandler); //register sigint event handler
   ```

2. Then add the event handler methods in the Program class:

   ```cs
   private static void SigTermEventHandler(AssemblyLoadContext obj)
   {
   	System.Console.WriteLine("Unloading...");
   }

   private static void CancelHandler(object sender, ConsoleCancelEventArgs e)
   {	     
   	System.Console.WriteLine("Exiting...");
   }
   ```

3. Finally, add the main daemon logic in Main:

   ```cs
   while(true)
   {
   	Console.WriteLine("Hello World!");
   	Thread.Sleep(2000); //Sleep for 2 seconds. Don't forget to import System.Threading!
   }
   ```

4. Publish your app: `dotnet publish -o /opt/daemons/myfirstdaemon/`

5. Now we need to tell systemd about our service. We're going to use [systemd user units](https://wiki.archlinux.org/index.php/Systemd/User).
	Run `systemctl edit --user --full --force myfirstdaemon.service` and add the following contents:

   ```ini
   [Unit]
   Description=Test .Net service

   [Service]
   ExecStart=/usr/local/bin/dotnet /opt/daemons/myfirstdaemon/myfirstdaemon.dll #Tell systemd which program to start
   ```

6. Run your service: `systemctl start --user myfirstdaemon.service`. You should be able to see the output by running `systemctl status --user myfirstdaemon.service` or `journalctl --user -u myfirstdaemon.service`

That's it! You now know how to write a very simple .Net Core daemon managed by systemd. To view the full source code, please visit my [Github](https://github.com/logankp/systemd.net-tutorial/tree/master/Project1).
