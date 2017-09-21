---
title: Adding logging to systemd service
layout: single
categories: [linux,systemd,dotnet]
comments: true
---

In the [first tutorial]({{ site.baseurl }}{% post_url 2017-09-18-systemd-dotnet-1 %}) you learned how to create a basic systemd service. In this tutorial, we are going to expand on that and learn how to log to the [journal](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html). Systemd makes this incredibly easy since all daemon output on Standard Output and Standard Error get captured and sent to the journal automatically.

## Getting started

Clone Project1 from [GitHub](https://github.com/logankp/systemd.net-tutorial/tree/master/Project1) since we're going to be adding to it.

Create a Logger.cs file. This class will be used to create our logging logic that allows us to specify logging levels. You can specify logging levels by prefixing your messages with &lt;level&gt;.

1. In Logger.cs:

   1. Define an enum containing logging levels:

   ```cs
   enum LogLevel
   {
        Emergency,
        Alert,
        Critical,
        Error,
        Warning,
        Notice,
        Info,
        Debug
   }
   ```

   2. Add a static class containing your logging methods. We define one method for logging to StdOut and another for logging to StdErr:

   ```cs
   static class Logger
    {
        //Writes to Standard Output
        public static void Log(string message, LogLevel level)
        {
            Console.WriteLine(GetLogMessage(message, level));
        }

        //Writes to Standard Error
        public static void LogError(string message, LogLevel level)
        {
            Console.Error.WriteLine(GetLogMessage(message, level));
        }

        private static string GetLogMessage(string message, LogLevel level)
        {
            string strLevel = $"<{(int)level}>";
            return $"{strLevel}{message}";
        }
}
   ```

2. Add the main logging logic to Main() in Program.cs:
  
   1. Before the while loop:

   ```cs
   Logger.Log("This is an emergency!", LogLevel.Emergency); //only do this once
   ```
   We only do this once otherwise we'll spam our console with emergency notifications in the loop.   
  
   2. Inside the while loop we simply iterate over the loglevels:

   ```cs
   foreach(LogLevel level in Enum.GetValues(typeof(LogLevel)).Cast<LogLevel>().Where(i => i != LogLevel.Emergency)) //skip emergency
   {
	string strLevel = Enum.GetName(typeof(LogLevel), level);
	Logger.Log($"LogLevel: {strLevel}", level);
   }
   Logger.LogError("This will print to StdErr", LogLevel.Error);
   ```

4. Build then run your program using systemd-run: `systemd-run --user --unit logging-service.service dotnet &lt;absolute path to logging.dll&gt;`

5. Check the status using `systemctl status --user logging-service.service`

Notice how the logged messages appear in the output. You should be able to filter on the levels using [journalctl](https://www.freedesktop.org/software/systemd/man/journalctl.html)

6. Make sure to stop the service: `systemctl --user stop logging-service.service`

All done! We've now learned how to log to the journal. There was nothing special that had to be done like there is for syslog becuase systemd handles it all for us. You can clone this project on [GitHub](https://github.com/logankp/systemd.net-tutorial/tree/master/Project2). There's a few other goodies there as well such as an NLog layout renderer that appends the log levels to messages. Stay tuned for the next articles on service start notification and socket activation!
