# .NET Asynchronous

## Best Practices
* async void will cause application to crash if an exception is thrown. Exceptions occuring in an async void method cannot be caught.
* Always return Task or Task<T>
* Don't introduce async/await if there is no continuation code. This creates a state machine. just return the Task. An exception to this rule is if you need to do exception handling.
* Use async and await all the way up the chain
* Dangerous to wrap sync calls in Task.Run. Better to copy, paste and update to be async.
* Use thread safe collections such as ConcurrentBag.
* Interlocked.Add(ref variableName, 1) allows thread safe integer updating or lock(object) for non integers. Optimize code so the least amount of locks are used.
* [Getting Started with Asynchronous Programming in .NET](https://app.pluralsight.com/library/courses/getting-started-with-asynchronous-programming-dotnet/table-of-contents)
* [Async Antipatterns](https://markheath.net/post/async-antipatterns)

## Async/Await

```
await Task.Run(() => {

	Dispatchter.Invoke(() => {
		//Update UI
	});
});

await Task.Run(async () => {

	Dispatchter.Invoke(() => {
		//Update UI
	});
});
```

## Task Parellel Library
* Task.Run queues the work passed as the action to run on a different thread by using the thread pool
* A big difference between using await vs continueWith is that it doesn't continue with the same context/thread. 
* A continueWith will always occur regardless if successful or exception is thrown!
* a continueWith will return an AggregateException so need to look at the InnerException property.

### ContinueWith

```
var t1 = Task.Run(() => {

	Dispatchter.Invoke(() => {
		//Update UI
	});
});

t1.ContinueWith(t => {

});
```

### ContinueWith OnSuccess

```
var t1 = Task.Run(() => {

	Dispatchter.Invoke(() => {
		//Update UI
	});
});

t1.ContinueWith(t => {

}, TaskContinuationOptions.OnlyOnRanToCompletion);
```

### ContinueWith Exception

```
var t1 = Task.Run(() => {

	Dispatchter.Invoke(() => {
		//Update UI
	});
});

t1.ContinueWith(t => {

}, TaskContinuationOptions.OnlyOnFaulteds);
```

## Cancellation Tokens
* Cancelling a task does not magically stop the task. You have to listen for the cancellation and handle it.
* The cancellationToken passed into Task.Run just ensures the task is not run if the cancellation has already been requested.

```
var cts = new CancellationTokenSource();
cts.Cancel();

cts.Token.Register(() => {
	//cancellation requested callback
});

Task.Run(() => {
	cts.Token.ThrowIfCancellationRequested();
	
}, cts.Token);
```

```
var cts = new CancellationTokenSource();
cts.Cancel();

try
{
	var client = new HttpClient();
	await client.GetAsync(@"http://www.google.com", cts.Token);
}
catch(CancellationException ex)
{

}
```

## Task.WhenAll and Task.WhenAny

```
var cts = new CancellationTokenSource();

var timeoutTask = Task.Delay(2000);
var resultsTask = Task.WhenAll(tasks);

var completedTask = await Task.WhenAny(timeoutTask, resultsTask);

if(completedTask == timeoutTask)
{
	cts.Cancel();
	throw new Exceotion("Timeout!");
}

var concatResults = resultsTask.Result.SelectManay(r => r);
```

## Task.FromResult
* Allows mocking of async operation


## Execution Context
* When writing library code always add ConfigureAwait(false) to allow execution to continue on a different thread. Performance improvement as doesn't need to use a different thread for continuation.
* ConfigureAwait only affects continuation in the current method.
* ASP.NET Core doesn't leverage synchronization so ConfigureAwait(false) is useless.

```
	var client = new HttpClient();
	await client.GetAsync(@"http://www.google.com").ConfigureAwait(false);
```

## Parallel Extensions
* Allows to break down a large problem and compute each piece independently.
* Independent chunks of data that are CPU bound.
* Calculates most efficient way to divide tasks among CPU cores.
* Calling anything on the Parallel Extensions is a blocking operation. Wrap in Task.Run for ASP.NET.
* Never call Dispatcher.Invoke from Parallel Extensions without Task.Run wrapper as UI thread will be blocked.
* Stop() and Break() will not stop any currently executing iterations. 
* Interlocked.Add(ref variableName, 1) allows thread safe integer updating or lock(object) for non integers. Optimize code so the least amount of locks are used.

```
Task.Run(() => 
{
	Parellel.Invoke(new ParallelOptions { MaxDegreeOfParallelism = 2 },() => {
		//Task 1
	},
	() => {
		//Task 2
	},
	() => {
		//Task 3
	},
	() => {
		//Task 4
	});
}
);
```

```
static object syncRoot = new object();
int total = 0;
var results = new ConcurrentBag<Class>();
Task.Run(() => 
{
	Parellel.ForEach(list, (item, state) => {
		results.Add(new Class());
		
		state.Break();
		state.Stop();
		Interlocked.Add(ref total, 1);
		lock(syncRoot)
		{
			total = total + 1;
		}
	});
}
);
```

## TaskCompletionSource
* Useful for when working with libraries that don't expose tasks but do have callbacks events.

```
await WorkInNotepad();

public Task WorkInNotepad()
{
	var source = new TaskCompletionSource<object>();
	var process = new Process
	{
		EnableRaisingEvents = true,
		StartInfo = new ProcessStartInfo("Notepad.exe")
		{
			RedirectStandardError = true,
			UseShellExecute = false
		}
	};
	
	process.Exited += (sender, e) => {
		source.SetResult(null);
	};
	
	process.Start();
	return source.Task;
}
```

## TaskFactory
* Task.Run is shorthand for Task.Factory.StartNew. By default tasks spawned are not attached to parent task.

```
Task.Factory.StartNew(() => {},
	CancellationToken.None,
	TaskCreationOptions.DenyChildAttach,
	TaskScheduler.Default
	);
```

* For long running tasks use Task.Factory.StartNew with TaskCreationOptions.LongRunning.
* Prevent parent from completing until all child tasks are completed by using TaskCreationOption.AttachedToParent. 

```
Task.Factory.StartNew(() => {
	Task.Factory.StartNew(() => {
	
	}, TaskCreationOption.AttachedToParent);
	
	Task.Factory.StartNew(() => {
	
	}, TaskCreationOption.AttachedToParent);
	
	Task.Factory.StartNew(() => {
	
	}, TaskCreationOption.AttachedToParent);	
});
```

* When using TaskFactory need to use await await Task to get result otherwise use .Unwrap()

```
var service = new StockService();
var operation = Task.Factory.StartNew(async (obj) => {

	var stockService = obj as StockService();
	return await StockService).GetPrices();
}, service);
var result = await await operation;
```

```
var service = new StockService();
var operation = Task.Factory.StartNew(async (obj) => {
	var stockService = obj as StockService();
	return await StockService).GetPrices();
}, service).Unwrap();
var result = await operation;
```

## Async Streams
* using await foreach with IAsyncEnumerable allows results to be processed as soon as they are available rather than once all results are available.

```
foreach(var stock in await LoadStocks())
{
	Console.WriteLine($"{stock.Ticker} {stock.Change}");
}

await foreach(var stock in LoadStocks())
{
	Console.WriteLine($"{stock.Ticker} {stock.Change}");
}

public async IAsyncEnumerable<StockPrice> LoadStocks()
{
	using(var stream = new StreamReader("file.csv"))
	{
		await stream.ReadLineAsync();
		string line;
		while(line = await stream.ReadLineAsync() != null)
		{
			var segments = line.Split(',');
			yield new StockPrice();
			await Task.Delay(200);
		}
	}
}
```

## Data Processing
```
Parallel.ForEach(items, new ParellelOptions {MaxDegressOfParallelism = 2,}, (item, state) => {
	state.Break() //complete all iterations on all threads that are prior to the current iteration on the current thread 
	state.Stop() //stop all iterations as soon as convenient 
});
```