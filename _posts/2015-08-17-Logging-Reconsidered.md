---
layout: post
title: Logging Reconsidered
---
Hi, I'm David and in this post I'm going to re-examine my [previous post]({% post_url 2015-08-14-Logging-Considered-Harmful %}) in which I concluded that logging applied carelessly can disturb the natural operation of a program to the point where it could even hide race conditions.

As a reminder, here's the code in question:

```csharp
	var result = 100;
	var expected = 93;
	var workerCount = 15;
	var workers = Enumerable
		.Range (0, workerCount)
		.Select ((x) => {
			return Task.Run (() => {
				Console.WriteLine("worker {0}", x);  // blink and you'll miss it!
				result += x % 2 == 0 ? -x : x;
			});
		})
		.ToList ();
		await Task.WhenAll (workers)
			.ContinueWith ((t) => {
				if (result != expected) {
					// race detected!
				}
			});
```

I'm fortunate to have some extremely smart friends who are also very good programmers. After I advertised my previous post to them and asked for comments, one came back with the very good observation that the working part of my racing code consumed much less CPU time than the log statements used to record information about it.

I have to agree. The line

```csharp
	result += x % 2 == 0 ? -x : x;
```

is over very quickly.

The line

```csharp
	Console.WriteLine("worker {0}", x);  // blink and you'll miss it!
```

takes a good bit longer to execute, largely because there are several subroutine calls involved all the way down to the depths of the .NET console, where the system then does [this](https://github.com/dotnet/corefx/blob/master/src/System.Console/src/System/IO/SyncTextWriter.cs#L287):

```csharp
	public override void WriteLine(String format, Object[] arg)
	{
		lock (_methodLock)
		{
			_out.WriteLine(format, arg);
		}
	}
```

Taking the lock and composing the output string are clearly expensive activities (in terms of CPU time) compared to adding or subtracting one number from another. 

My friend then quite reasonably asked: "What happens if you run the add/subtract calculation lots of times during the `Task`?" The implication being that the task may then take long enough to balance the effect of taking a lock during logging, and the lock will no longer hide the race.

So of course in the name of Scientific Research I had to try it. I modified my program to loop around the add/subtract calculation, and increased the number of iterations until the system began to race again while logging was still enabled. If you're inclined to try for yourself, your results will inevitably be different than the following, because everything about threads and multi-tasking is heavily dependent on the hardware the program is running on. But here's what I found for the PC-based command line version:

| Iterations | Result |
| -------------------: | ------ |
| 1 | No race |
| 10 | No race |
| 1000 | No race |
| 10000 | Race |

So on this test system (desktop PC with 8 cores and lots of RAM) I had to have a `Task` whose overall duration was on average about 10000 times that of the original `Task` to prevent the logs from hiding the race.

On the face of it this looks reasonable. In normal applications, multi-threaded code is usually quite long-running - that's often the driver for putting it into another thread in the first place, after all. Think of downloading a an image, updating a database, calculating the new position of some object in 3-D space - these are all tasks that will on the whole take much longer than adding an entry in a log.

Should I be concerned, then? I think the answer to that question, as in so much of computing, is "it depends". If my threaded tasks are relatively long-running, I'll probably get away with simple logging to debug them. 

But logs have a way of growing - once you start, there's much seemingly-useful information to be extracted from a program while it's running - and there's a very real danger that the CPU time involved in logging will begin to outweigh the CPU time used by working code, like barnacles on a ship. At that point, I'm once again in danger of masking out the races in my code by all the logging that's going on.

To my mind, the safest way to do logging in a program is to use a non-blocking logger. This offers the best guarantee that logging code won't interfere with working code. 

Of course, there's no such thing as a free lunch - and in the world of logging, a non-blocking logger can mean the runtime has to create at least one extra thread for the logger to run in. This in and of itself can affect the operation of an application, especially if the hardware has fewer cores than the number of threads needed to run it. So while it's possible that the C# `async`/`await` construct could eliminate the requirement for an extra thread to manage the logging, the risk still exists and must be accounted for.

Happily, in C# there are several nice ways to create a non-blocking logger that all work well in a [Xamarin](www.xamarin.com) app.

I'll explore one of them in my next post.
