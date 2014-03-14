Bulk Writer
===========

# Table of Contents #

1. Introduction
1. DecoratedModel assembly
1. Nhibernate assembly 
1. Examples
 1. Relative distances between an entity and all zip codes
 2. Pipelining
 3. Advanced Pipelining

# Introduction #

We've all had reasons to write ETL jobs in C# rather than with Integration Services in SQL Server. Sometimes it's because the transform logic is easier to reason about in C#, but for whatever reason, it's a perfectly acceptable way to to do things.

When writing an ETL process in C#, we use the tools available to us like NHibernate, Entity Framework or PetaPoco to read and write data. Unfortunately those tools help us stream from source data pretty easily, but they don't make it easy to stream data into our target data stores. Instead, they typically leave us writing to target data stores with `INSERT` statements, which is not acceptable for transforms that generate very large data sets.

> **What we need for transforms that generate very large data sets is a way to stream data into a target data store, just as we're able to stream from source data.**

Such a technique would allow writing to target data stores as fast as our transforms would allow, compared to relying on our ORM to generate `INSERT` statements. In most cases, it would also use significantly less memory.

## SqlBulkCopy ##

`SqlBulkCopy` (or the Oracle or MySql equivalents) is the most likely candidate to stream into a target data store, but its `WriteToServer()` methods take a `DataRow[]`, a `DataTable` or an `IDataReader`.

The methods that take a `DataRow[]` or a `DataTable` aren't really useful when transforms produce very large data sets because they force us to load the entire data set into memory before being streamed into the target data store.

Which leaves us to examine how to leverage the `WriteToServer(IDataReader)` method. If you think about how `IDataReader` works, users of an `IDataReader` instance must call the `Read()` method before examining a current record. A user advances through the result set until `Read()` returns false, at which point the stream is finished and there is no longer a current record. It has no concept of a previous record. In this way, `IDataReader` is a *non-caching forward-only pull reader*.

There are other non-caching forward-only pull readers in .NET, which are used every day by most developers. The most used example of this type of reader is `IEnumerator`, which works similarly to `IDataReader`. However, instead of a `Read():bool` method, `IEnumerator` has a `MoveNext():bool` method. In every other way, `IDataReader` and `IEnumerator` are similar.

## Using SqlBulkCopy ##

The Bulk Writer core assembly has an `IDataReader` implementation that wraps an `IEnumerator`. You can give this `IDataReader` implementation to an instance of `SqlBulkCopy`, so that when it calls for the next record to write, the implementation retrieves the next record from the underlying `IEnumerator`.

It is conceivable that `IEnumerator.MoveNext()` and `IEnumerator.Current` are proffering records from any type of data source, but typically you retrieve an instance of `IEnumerator` from calling `IEnumerable.GetEnumerator()`. So, you can think of the `IDataReader` implementation in this way:

> **You can give the `IDataReader` implementation to `SqlBulkCopy`, and it'll stream your `IEnumerable` into your target data store.**

Most of the other code in the core assembly is for mapping properties on source data to columns in the target data store. 

This technique does require you to reason differently about your ETL jobs. Most jobs *push* data into the target data store. This technique requires you to think about how to structure your transforms so that data is *pulled* from your source data through your transforms instead.

Any number of rows - since it is technically possible to produce infinite data sets with `IEnumerable` - can be pulled into a SQL Server table as soon as your `IEnumerable` can produce them and using very little memory.

## DecoratedModel assembly ##

*Work in progress...*

## Nhibernate assembly ##

*Work in progress...*

# Examples #

By itself, Bulk Writer is a pretty simple concept and the code itself isn't really all that complicated. However, we believe even simple implementations can enable very complex scenarios. The rest of this document shows examples of what you can do with Bulk Writer.

All of these examples use the DecoratedModel assembly.

## Relative distances between an entity and all zip codes ##

Suppose that for performance reasons you wanted to cache the distances between an entity (such as a store, house or distribution center) and the centroid of every zip code in the U.S. Depending on the number of entities, you could easily produce a very large data set from these calculations. But sure, entity locations and zip code centroids aren't likely to change often enough to warrant computing this result set on every ETL job run, but the real point of this example is to show

1. Sometimes our transforms can produce exponentially larger data sets than our source data
2. That those very large data sets can be written to our target data store much faster than by generating an `INSERT` statement for each row.

We start off with this LINQ query that serves as our transform and will produce our large data set.

    var q =
      from entity in GetAllEntities()
      where entity.IsActive && SomeOtherPredicate(entity)
      from zipCode in GetAllZipCodes()
      where zipCode.IsInContiguousStates && SomeOtherPredicate(zipCode)
      let distance = GetDistance(entity, zipCode)
      let arbitraryData = CreateSomeArbitraryData(entity, zipCode)
      where distance > 0
      select new EntityToZipCodeDistance {
         EntityId = entity.Id,
         ZipCode = zipCode.Zip,
         Distance = distance,
         ArbitraryData = arbitraryData
      };

Note that this LINQ query does not execute until its `IEnumerator.MoveNext()` method is called, which will ultimately be called by `SqlBulkCopy`.

Next, all there is to do is let Bulk Writer write the results to your database table.

    var bulkCopyFactory = CreateBulkCopyFactory();
    var dataWriter = new EnumerableDataWriter();
    dataWriter.WriteToDatabase(q, bulkCopyFactory);

## Pipelining ##

The example above shows a transform that's pretty simple. But some ETL jobs require transforms that are so complex (and the code is so complicated), it's easier to reason about and implement the transforms in steps. You can create these steps using a single LINQ query (which are actually pipelines themselves), or you can implement a Pipeline with Stages.

Typical pipelines push data from one stage to the next.

            =========         =========         =========         ==========
            |       |         |       |         |       |         |        |
    Push -> | Stage | Push -> | Stage | Push -> | Stage | Push -> |  Sink  |
            |       |         |       |         |       |         |        |
            =========         =========         =========         ==========

Using Bulk Writer, data is *pulled* through the Pipeline by the `EnumerableDataWriter` class.

    =========         =========         =========         ==========
    |       |         |       |         |       |         |        |
    | Stage | <- Pull | Stage | <- Pull | Stage | <- Pull |  Sink  |
    |       |         |       |         |       |         |        |
    =========         =========         =========         ==========

Take the following example:

	var begin = DoStepBegin();
    var step1 = DoStep1(begin);
    var step2 = DoStep2(step1);
    var step3 = DoStep3(step2);
    DoStepEnd(step3);
      
    // Or a one-liner!!
    // DoStepEnd(DoStep3(DoStep2(DoStep1(DoStepBegin()))));

    private static IEnumerable<BeginStepResult> DoStepBegin() {
       foreach (var item in Enumerable.Range(0, 1000000)) {
          yield return new BeginStepResult();
       }
    }

    private static IEnumerable<Step1Result> DoStep1(IEnumerable<BeginStepResult> input) {
        // Or using LINQ, which does the same thing as below
        return input.Where(x => SomePredicate(x)).Select(x => new Step1Result(x));
    }

    private static IEnumerable<Step2Result> DoStep2(IEnumerable<Step1Result> input) {
       foreach (var item in input) {
	      if (SomePredicate(item)) {
             yield return new Step2Result(item);
          }
       }
    }

    private static IEnumerable<Step3Result> DoStep3(IEnumerable<Step2Result> input) {
       foreach (var item in input) {
          foreach(var many in GetManyItems(item)) {
             yield return Step3Result(many);
          }
       }
    }

    private static void DoStepEnd(IEnumerable<Step3Result> input) {
       var bulkCopyFactory = CreateBulkCopyFactory();
       var dataWriter = new EnumerableDataWriter();
       dataWriter.WriteToDatabase(input, bulkCopyFactory);
    }

## Advanced Pipelining ##

The example above is fine, but we're only processing one source data item at a time. If one pipeline stage takes longer to produce output than other stages, all stage processing suffers. We argue that there are times when you'd like any pipeline stage to continue to process available items even if other stages in the pipeline are blocked.

For example, suppose Stage 1 was IO-bound because it queried and produced a result set for each of its input items. In other words, Stage 1 is producing a larger data set than its input.  Next, supposed Stage 2 was CPU bound because it performed hundreds of calculations on each row produced by Stage 1. In this example, there's no reason why Stage 2 shouldn't be able to perform its calculations while Stage 1 is querying and producing input for Stage 2.

We form a pipeline like this by running each pipeline stage on its own thread and by introducing an input and output buffer between each stage. Now, instead of a pipeline stage pulling directly from the previous stage, each pipeline stage pushes to and pulls from its input and output buffer, respectively.

Such a pipeline would look like this:

            ===                           ===                           ===
            | |         =========         | |         =========         | |         ==========
            ===         |       |         ===         |       |         ===         |        |
    Push -> | | <- Pull | Stage | Push -> | | <- Pull | Stage | Push -> | | <- Pull |  Sink  |
            ===         |       |         ===         |       |         ===         |        |
            | |         =========         | |         =========         | |         ==========
            ===                           ===                           ===
           Buffer                        Buffer                        Buffer

Since each stage is running on its own thread, we need to be careful so that the stage's thread doesn't end before all the items the pipeline needs to process have been pushed through. We also want to block `SqlBulkCopy` until an item is ready to write. In essence, what we need is a buffer that is thread-safe and blocks the current thread until a new item is available. .NET already has such a buffer, `BlockingCollection<T>`.

To implement a pipeline like this, you would do the following:

    var taskFactory = new TaskFactory(TaskCreationOptions.LongRunning, TaskContinuationOptions.None);
    
    var stage1Input = new BlockingCollection<Stage1Input>();
    var stage2Input = new BlockingCollection<Stage2Input>();
    var finalStageInput = new BlockingCollection<FinalStageInput>();
    
    var stage1 = taskFactory.StartNew(() => {
       var enumerable = stage1Input.GetConsumingEnumerable();
       
       try {
          foreach(var item in enumerable) {
             var outputs = GetOutputsFromIO(item);
             foreach(var output in outputs) {
                stage2Input.Add(output);
             }
          }
       } finally {
          // Let's the task managing Stage 2 know that it can end
          stage2Input.CompleteAdding();
       }
    });
    
    var stage2 = taskFactory.StartNew(() => {
       var enumerable = stage2Input.GetConsumingEnumerable();
       
       try {
          foreach(var item in enumerable) {
             var outputs = DoLotsOfCalculations(item);
             foreach(var output in outputs) {
                finalStageInput.Add(output);
             }
          }
       } finally {
          finalStageInput.CompleteAdding();
       }
    });
    
    var finalStage = taskFactory.StartNew(() => {
       var enumerable = finalStageInput.GetConsumingEnumerable();
       
       var bulkCopyFactory = CreateBulkCopyFactory();
       var dataWriter = new EnumerableDataWriter();
       dataWriter.WriteToDatabase(enumerable, bulkCopyFactory);
    });

    // All stages are started and waiting for work at this point
    
    // Populate your first stage here
    foreach (var item in Enumerable.Range(0, 1000000)) {
       stage1Input.add(new Stage1Input());
    }

    stage1Input.CompleteAdding();
    
    await Task.WhenAll(new[] { stage1, stage2, finalStage });