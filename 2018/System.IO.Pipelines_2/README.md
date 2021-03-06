# System.IO.Pipelines Review: Core APIs

Status: **Approved with Feedback** | 
[API](System.IO.Pipelines2.md) |
[Presentation](System.IO.Pipelines2.pptx) |
[Video](https://www.youtube.com/watch?v=1t_a9fbB3jY)

# System.IO.Pipelines 2

## Minor review items ( and String like Extension Methods)

* [Array-like extension methods for `Span<T>`](https://github.com/dotnet/corefx/issues/25850)
* [String-like extension methods for `Span<char>`](https://github.com/dotnet/corefx/issues/21395)

## Pipelines

### Write Loop

```csharp
while (true)
{
    try
    {
        var memory = pipe.Writer.GetMemory();
        // ... write to memory
        pipe.Writer.Advance(bytesWritten);

        if (/* data completion condition */)
        {
            pipe.Writer.Complete();
            break;
        }
    }
    finally
    {
        await pipe.Writer.FlushAsync();
    }
}
```

* Three modes for sending data to the reader
    1. `FlushAsync`. Will send the data to the reader and the reader will wake
       up from the `ReadAsync` call
    2. Commit.
    3. Partial flush. Currently not supported. Do we need this? Ben and Tim
       seemed to believe it's useful. Pavel thinks it can added a later. The
       workaround would be a side channel (variable + lock).

### Read Loop

```csharp
while (true)
{
    var result = await reader.ReadAsync();
    try
    {
        if (result.IsCompleted && result.Buffer.IsEmpty)
        {
            break;
        }
        // ... process data in result.Buffer
    }
    finally
    {
        reader.Advance(result.Buffer.End);
    }
}
```

* We should rename `Advance` to `AdvanceTo`. `Advance(value)` implies a relative
  increment, rather than a specific position.
* `ReadAsync` needs to throw if it's called twice with no `Advance` in-between
  otherwise users will have infinite loop and have no idea what's going on.
* `Advance(consumed)` shouldn't dead-lock or cause inifinite loops
    - Calls `Advance(consumed, consumed)` unless `consume` is `0`, in which case
      it will call `Advance(0, reader.Buffer.End)`
* `IPipeConnection`
    - It seems folks would like to have *duplex* in the name, but that would
      make the name long (`IDuplexPipeConnection`)
    - Sounds like we've settled on `IPipeConnection`
* `PipeOptions`
    - Should expose a static readonly property `Default` that uses the default memory pool
    - `maximumSizeLow` -> `pauseWriterThreshold`
    - `maximumSizeHigh` -> `resumeWriterThreshold`
* `Pipe`
    - Offer an constructor with notarguments that passes in
      `PipeOptions.Default`
* `IOutput`
    - In order to settle on `IOutput` as the key constraint for writers, we need
      a way get a `Span<T>` as this is often more efficient.
    - We could split into `IOutput` into `ISpanOutput` `IMemoryOutput`

## Decisions

* Rename `Position` to `SequencePosition`
* Rename `IOutput` to `IBufferWriter`
* Rename `IPipeConnection` to `IDuplexPipe`
* Make `Pipe` non-abstract
    - Add `Reset` method
    - Has an internal ctor
    - Customers building their own pipes should create their own type
* Rename `Scheduler` to `PipeScheduler` and move to `System.IO.Pipelines`
* We should try to align the namespaces between channels and pipes. Ideally,
  they should be peers under the same common root.
* Remove `PipelineExtensions.WriteAsync(this IPipeWriter output, byte[] source)`