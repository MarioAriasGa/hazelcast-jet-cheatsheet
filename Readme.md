# Hazelcast Jet Cheatsheet

Useful after reading [Hazelcast Jet Get Started](https://jet-start.sh/docs/get-started/intro)

## Dependency

[Maven Central Link](https://search.maven.org/artifact/com.hazelcast.jet/hazelcast-jet)

Maven:
```xml
<dependency>
    <groupId>com.hazelcast.jet</groupId>
    <artifactId>hazelcast-jet</artifactId>
    <version>4.3</version>
</dependency>
```

Gradle:
```
dependencies {
    compile 'com.hazelcast.jet:hazelcast-jet:4.3'
}
```
## Launching an app in a cluster

1. [Download hazelcast jet distributtion](https://jet-start.sh/download)
2. Start cluster node `$ bin/jet-start`
3. Submit job `$ bin/jet submit path/to/job.jar`

You can start several nodes in the network and they will find each other using Multicast to form a cluster. If any node dies, Hazelcast will heal itself and resume running jobs.

To list and stop jobs:
* `bin/jet list-jobs`
* `bin/jet cancel <job> # Can use name or UUID` 

## Hello World

```java
JetInstance jet = Jet.bootstrappedInstance(); // Connect or create instance
Pipeline p = Pipeline.create();
  p.readFrom(TestSources.itemStream(10))
   .withoutTimestamps()
   .filter(event -> event.sequence() % 2 == 0)
   .setName("filter out odd numbers")
   .writeTo(Sinks.logger());
jet.newJob(p).join();   // Submit job
```
## Pipeline
```java
Pipeline p = Pipeline.create();
p.readFrom(source)
/* Apply some stages */
.writeTo(sink);
```

You can fork a pipeline to continue processing same data differently:
```java
Pipeline lines = Pipeline.create().readFrom(Sources.files("file.txt"));
lines.someStagesA().writeTo(sinkA);
lines.someStagesB().writeTo(sinkB);
```

You can do union of pipelines. They are interleaved in arbitrary order.
```java
BatchStage<String> left = p.readFrom(leftSource);
BatchStage<String> right = p.readFrom(rightSource);
left.merge(right)
.writeTo(sink);
```

## Sources
Where the data comes from.

### Batch Sources (Finite elements like an existing list)
They return a `BatchSource<T>`
```java
Sources.files(dir);  // Each file in dir
Sources.jdbc(query, resultLambda);  // DB connection
Sources.map(imap);  // From Hazelcast IMap, using name string or actual IMap object
```

### Streaming sources (Unbound)
They return a `StreamSource<T>`
```java
Sources.mapJournal(imap,initPosition);  // Changes to IMap as they come
Sources.fileWatcher(dir);               // Lines added to  files in dir
Sources.socket(host,port,charset);      // Text received from socket
Sources.jmsQueue(); Sources.jmsTopic(); // JMS Queue
KafkaSources.kafka(kafkaConnectionProperties,"topic");
TestSources.items(arrayOrIterable);     // Given data for testing
TestSources.itemStream(perSecond, generatorLambda); // Stream items returned by lamda
```

## Sinks
```java
Sinks.noop();           // Discard
Sinks.logger();         // To cluster log
Sinks.logger(format);   // With custom format
Sinks.map(name);        // To Hazelcast IMap (by name or java object)
Sinks.map(name,keyLambda,valueLambda);  // Applying a transform
Sinks.mapWithMerging(); Sinks.mapWithUpdating();    // Combine to existing data
Sinks.observable(observable);   // To notify an observable (Useful to get notifications in your app)
Sinks.list(list);       // To IList
Sinks.socket(host,port,stringLambda);   // Write to socket
Sinks.jmsQueue(queue, connectionFactory);
Sinks.jmsTopic(topic, connectionFactory);
Sinks.jdbc(query,db);   // Update query to SQL database
Sinks.json(dir);        // Each item appended as a json line.
Sinks.cache(cache);     // To Hazelcast ICache
```

## Timestamps
To characterize a `StreamStage`

* `withNativeTimestamps()`: They come from source (e.g. Kafka) 
* `withTimestamps(lambda)`: Get from the event object using lambda.
* `withIngestionTimestamps()`: When the item was received for processind, no matter when it was generated.
* `withoutTimestamps()`: Don't care, will not use window aggregation or they will be assigned at a later stage with `addTimestamps(lambda)`.

## Stateless Stages
Functional, same input=same output without keeping internal state. Easily parallelizable.
```java
.map(lambda);   // Convert one object to another
.filter(predicate); // Keep only elements where predicate returns true
.flatMap(lamda);    // You can return several items for each input using a Traverser(similar to iterator)
.mapUsingIMap(imap,keyLambda,outLambda);    // Look up each item in IMap
.mapUsingReplicatedMap(rmap,keyFn,outFn);   // Look up each item in ReplicatedMap (that is duplicated fully on each node)
.mapUsingService(service,outFn);  // Lookup each item in a service, fn receives original item and service output to produce stage output
.mapUsingServiceAsync();   // If service is async (returns CompletableFuture) 
.mapUsingServiceBatched();  // Service receives a list of input and returns a list of results
.mapUsingPython();  // Call python code
.merge(other);  // Do union to results of other source, interleaved.
.hashJoin(batchStage,JoinClause.onKeys(leftKeyFn,rightKeyFn), outFN);    // Build hash of results of other pipeline, then join elements to current using condition, pass both sides to outFN to generate output
.innerHashJoin(...);    // Same with inner semantic, id appeared in both sides
.hashJoin2(...);    // Join to two other sources at once.
```

## Stateful Stages
They accumulate/transform data or depend on what came before.

### Aggregations
```java
pipeline
.aggregate(aggregateOp);    // aggregate whole input (one output)
.rollingAggregate(aggOp);   // One output per item received (incremental)
.groupingKey(User::getId).aggregate(aggOp);  // Group items with same property, one output per group
.window(windowDefinition).aggregate(aggOp);   // One output per window
.window(w).groupingKey(keyFn).aggregate(agg);    // One output per window/group
.setEarlyResultsPeriod(period);     // For windowed, start producing results earlier even if window is bigger
.aggregate2(aggOp, secondStage, secondAggOp)    // Join to other stream and aggregate at the same time. 
```

`WindowDefinition`
Note: Apply to timestamped pipelines
```java
WindowDefinition
.tumbling(timeunit)         // Exclusive windows. [1,2,3] [4,5,6]
.sliding(winsize,slide)     // E.g. 30 seconds window, slide by 1 second: 
                            // [0,30), [1,31), [2,32) ...
.sessionWindow(idleTime)    // Assume window is closed when didn't receive anything for some time
```


`AggregateOperations`. [Javadoc](https://jet-start.sh/javadoc/4.3/com/hazelcast/jet/aggregate/AggregateOperations.html)
```java
AggregateOperations
.counting()             // Count elements on each group
.averagingLong()        // Average
.averagingDouble()
.summingLong()          // Sum
.summingDouble()
.minBy(comparator)      // Minimum of group using comparator
.maxBy(comparator)
.bottomN(n, comparator) // Keep several minimum
.topN(n, comparator)    // Keep several top
.linearTrend(xFn,yFn)   // Slope of change (linear regresssion of items within window)
.concatenating(delim)   // Build a string with an optional delimiter
.toList() .toSet() toMap(keyFn,valFn)  // Gather to collection
.allOf(aggOpA,AggOpB)   // Calculate several AggregateOperations in one stage
.pickAny()              // Choose a random specimen
.reducing(emtpy, get, combine, /*optional*/ deduct)  // Custom reduction
```

### Other Stateful Operations

```java
.distinct()     // Remove duplicates, can be applied by window
.sort(/*optional*/comparator)         // Sort input using natural order or explicit comparator
.mapStateful(createFn,modifyFn);   // Like map, but keep mutable state between calls. Good for finding patterns.
```

## Service
```java
ServiceFactories.
.sharedService(ctx->obj);   // Create a Thread-safe instance, once per node
.nonSharedService(ctx->obj);    // Create an object, once per parallel jet processor (can be per node)

ServiceFactory
.toNonCooperative();   // Declare that service does blocking calls, jet uses separate thread.
.withAttachedFile(id,file);    // Attach file to context so service knows where to read from
.withAttachedDireectory(id,dir);    // Attach file to context so service knows where to read from
.withDestroyContextFn(destroyFn);   // Attach cleanup lambda
```

## Helper classes

`Traverser` Similar to iterator that implements `next()` but returns null to finish. Easier to implement using lambdas.

```java
Traversers
.empty()
.singleton(value)
.elements(a,b ...)
.iterator(it)
.stream(stream)
```

Many functions that end with `Ex` are equivalent to Java ones but serializable

