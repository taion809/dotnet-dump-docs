Creating and Analyzing a coredump
=================================

# Getting Started

I have prepared a sandbox workspace from which you, the reader, can test out taking and analyzing different coredump scenarios.

To follow along you will need
	- Vagrant
	- Virtualbox

Please take some time to read the following documentation: [diagnostics readme](https://github.com/dotnet/diagnostics/blob/master/README.md), [dotnet-dump instructions](https://github.com/dotnet/diagnostics/blob/master/documentation/dotnet-dump-instructions.md), and [installing sos](https://github.com/dotnet/diagnostics/blob/master/documentation/installing-sos-instructions.md)

TLDR:
	- vagrant up && vagrant ssh
	- mkdir -p $HOME/.dotnet/tools
	- mkdir -p /vagrant_data/dumps
	- echo "\nexport PATH=$PATH:$HOME/.dotnet/tools" >> $HOME/.bashrc && source $HOME/.bashrc
	- dotnet tool install -g dotnet-dump
	- dotnet tool install -g dotnet-symbol
	- dotnet tool install -g dotnet-sos
	- dotnet tool install -g dotnet-counters
	- dotnet-sos install

You should now be able to execute dotnet-dump, give it a shot.

```
> dotnet-dump analyze --help
analyze:
  Starts an interactive shell with debugging commands to explore a dump

Usage:
  dotnet-dump analyze [options] <dump_path>

Arguments:
  <dump_path>    Name of the dump file to analyze.

Options:
  -c, --command <command>    Run the command on start.
```

As an aside, dotnet-dump does not expose all of the functionality of lldb and sos, if you find yourself limited by what is available in dotnet-dump do not hesitate to use `lldb`.  Infact, a section of this document will cover starting and using sos and lldb.

# First Scenario: Hello World

In the first scenario we're going to explore a dotnet coredump for a hello world aspnetcore application.

### Building the application
Our HelloWorld application can be located in /vagrant_data/HelloWorld.  You can build this by executing the associated `build.sh` in that directory.

### Running the application
You can run the newly created application by executing the associated `run.sh` script in that directory.  You can either open multiple ssh sessions or use tmux to manage the background task, how you do it isn't really important.

### Test
You should be able to navigate your web browser to the port opened in your `Vagrantfile`.  The `Vagrantfile` accompanying this document has exposed port 5000.

## Take a dump
Now that things are working you should be able to take a coredump of the running process.  This is outlined in the [diagnostics documentation](https://github.com/dotnet/diagnostics/blob/master/documentation/dotnet-dump-instructions.md#using-dotnet-dump) in more detail.

```bash
> PID=$(pgrep --full "dotnet run") # You can use any form of process listing to fetch the corred pid.
> dotnet-dump collect -p $PID -o /vagrant_data/dumps/HelloWorld.$PID.coredump

Writing minidump with heap to /vagrant_data/dumps/helloworld.1234.coredump
Complete
```

Your coredump should be located in `/vagrant_data/dumps`

## Our First Analysis

```bash
> dotnet-dump analyze /vagrant_data/dumps/helloworld.1234.coredump

Loading core dump: /vagrant_data/dumps/helloworld.1234.coredump ...
Ready to process analysis commands. Type 'help' to list available commands or 'help [command]' to get detailed help on a command.
Type 'quit' or 'exit' to exit the session.
```

For our first analysis lets take a look at the number of threads managed by the CLR.
```bash
> clrthreads
ThreadCount:      14
UnstartedThread:  0
BackgroundThread: 10
PendingThread:    0
DeadThread:       3
Hosted Runtime:   no
                                                                                                        Lock
 DBG   ID OSID ThreadOBJ           State GC Mode     GC Alloc Context                  Domain           Count Apt Exception
   0    1  700 0000000001FC7050  2020020 Preemptive  00007F39ED214198:00007F39ED215FD0 0000000001FDF8F0 0     Ukn
   4    2  704 0000000001FCD4E0    21220 Preemptive  0000000000000000:0000000000000000 0000000001FDF8F0 0     Ukn (Finalizer)
   6    3  706 00007F39E0000C50  1020220 Preemptive  0000000000000000:0000000000000000 0000000001FDF8F0 0     Ukn (Threadpool Worker)
XXXX    4    0 00007F39E0001C10  1031820 Preemptive  0000000000000000:0000000000000000 0000000001FDF8F0 0     Ukn (Threadpool Worker)
XXXX    6    0 00007F39D4001C70  1031820 Preemptive  0000000000000000:0000000000000000 0000000001FDF8F0 0     Ukn (Threadpool Worker)
   7    7  70b 00007F39D4004FD0  2021220 Preemptive  00007F39ED231130:00007F39ED231FD0 0000000001FDF8F0 0     Ukn
   8    5  70c 00007F39D4009390  2021220 Preemptive  00007F39ED22F278:00007F39ED22FFD0 0000000001FDF8F0 0     Ukn
   9    8  70d 00007F39D400A400  2021220 Preemptive  00007F39ED22D3D0:00007F39ED22DFD0 0000000001FDF8F0 0     Ukn
  10    9  70e 00007F39D4017800  2021220 Preemptive  00007F39EC23B3E0:00007F39EC23BA60 0000000001FDF8F0 0     Ukn
XXXX   10    0 00007F39D402B910  1031820 Preemptive  0000000000000000:0000000000000000 0000000001FDF8F0 0     Ukn (Threadpool Worker)
  12   11  718 00007F39D0037C10    21220 Preemptive  00007F39EC2B21B8:00007F39EC2B3FD0 0000000001FDF8F0 0     Ukn
  11   12  70f 00007F39C0000C50    20220 Preemptive  00007F39EC65DB40:00007F39EC65F928 0000000001FDF8F0 0     Ukn
  13   13  737 00007F39B410F160  1020220 Preemptive  0000000000000000:0000000000000000 0000000001FDF8F0 0     Ukn (Threadpool Worker)
  14   14  809 00007F39E0004F60  1021220 Preemptive  00007F39ED2340F0:00007F39ED235FD0 0000000001FDF8F0 0     Ukn (Threadpool Worker)
```

This can get pretty intimidated at first glance.  It was for me, my first thought was "What is DBG, why is everything Preemptive, what exactly am I looking at?"  I have a todo to explain what each of these columns mean, it is pretty low level information that may or may never be useful for everyday work.

### Explore
With the above information you should be able to inspect each thread stack by issuing the `clrstack` command with the thread id you're interested in.

```
> setthread 0
> clrstack
OS Thread Id: 0x700 (0)
        Child SP               IP Call Site
00007FFC23762980 00007f3a8ec619f3 [HelperMethodFrame: 00007ffc23762980] System.Threading.WaitHandle.WaitOneCore(IntPtr, Int32)
00007FFC23762AD0 00007F3A13A6954D System.Threading.WaitHandle.WaitOneNoCheck(Int32) [/_/src/System.Private.CoreLib/shared/System/Threading/WaitHandle.cs @ 161]
00007FFC23762B10 00007F3A13A69434 System.Threading.WaitHandle.WaitOne(Int32) [/_/src/System.Private.CoreLib/shared/System/Threading/WaitHandle.cs @ 133]
00007FFC23762B30 00007F3A1491E8D6 System.Diagnostics.ProcessWaitState.WaitForExit(Int32) [/_/src/System.Diagnostics.Process/src/System/Diagnostics/ProcessWaitState.Unix.cs @ 408]
00007FFC23762BB0 00007F3A1491E7D7 System.Diagnostics.Process.WaitForExitCore(Int32) [/_/src/System.Diagnostics.Process/src/System/Diagnostics/Process.Unix.cs @ 212]
00007FFC23762BE0 00007F3A1491B359 System.Diagnostics.Process.WaitForExit() [/_/src/System.Diagnostics.Process/src/System/Diagnostics/Process.cs @ 1328]
00007FFC23762C00 00007F3A1420A874 Microsoft.DotNet.Cli.Utils.Command.Execute() [/_/src/Microsoft.DotNet.Cli.Utils/Command.cs @ 63]
00007FFC23762C70 00007F3A140D0090 Microsoft.DotNet.Tools.Run.RunCommand.Execute() [/_/src/dotnet/commands/dotnet-run/RunCommand.cs @ 60]
00007FFC23762CE0 00007F3A140CFD56 Microsoft.DotNet.Tools.Run.RunCommand.Run(System.String[]) [/_/src/dotnet/commands/dotnet-run/Program.cs @ 34]
00007FFC23762CF0 00007F3A140F24A9 Microsoft.DotNet.Cli.Program.ProcessArgs(System.String[], Microsoft.DotNet.Cli.Telemetry.ITelemetry) [/_/src/dotnet/Program.cs @ 217]
00007FFC23762E20 00007F3A140F1A4D Microsoft.DotNet.Cli.Program.Main(System.String[]) [/_/src/dotnet/Program.cs @ 47]
00007FFC23763168 00007f3a8d35acaf [GCFrame: 00007ffc23763168]
00007FFC23763650 00007f3a8d35acaf [GCFrame: 00007ffc23763650]
```

#### Help
You can find helpful documentation for a command  by using the `help` command

```
> help clrstack
-------------------------------------------------------------------------------
CLRStack [-a] [-l] [-p] [-n] [-f]
CLRStack [-a] [-l] [-p] [-i] [variable name] [frame]

CLRStack attempts to provide a true stack trace for managed code only. It is
handy for clean, simple traces when debugging straightforward managed
programs. The -p parameter will show arguments to the managed function. The
-l parameter can be used to show information on local variables in a frame.
SOS can't retrieve local names at this time, so the output for locals is in
the format <local address> = <value>. The -a (all) parameter is a short-cut
for -l and -p combined.
...
...
```

#### Dump stack for all threads
It can become incredibly tedious to individually dump threads, maybe try `clrstack -all`.

```
> clrstack -all
OS Thread Id: 0x700
        Child SP               IP Call Site
00007FFC23762980 00007f3a8ec619f3 [HelperMethodFrame: 00007ffc23762980] System.Threading.WaitHandle.WaitOneCore(IntPtr, Int32)
00007FFC23762AD0 00007F3A13A6954D System.Threading.WaitHandle.WaitOneNoCheck(Int32) [/_/src/System.Private.CoreLib/shared/System/Threading/WaitHandle.cs @ 161]
00007FFC23762B10 00007F3A13A69434 System.Threading.WaitHandle.WaitOne(Int32) [/_/src/System.Private.CoreLib/shared/System/Threading/WaitHandle.cs @ 133]
00007FFC23762B30 00007F3A1491E8D6 System.Diagnostics.ProcessWaitState.WaitForExit(Int32) [/_/src/System.Diagnostics.Process/src/System/Diagnostics/ProcessWaitState.Unix.cs @ 408]
OS Thread Id: 0x704
        Child SP               IP Call Site
00007F3A8B17CD60 00007f3a8ec61ed9 [DebuggerU2MCatchHandlerFrame: 00007f3a8b17cd60]
OS Thread Id: 0x706
        Child SP               IP Call Site
OS Thread Id: 0x70b
        Child SP               IP Call Site
...
...
...
```

