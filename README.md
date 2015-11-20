# Limited Async

An executor service (a.k.a. smart pool of threads) that is backed by a [ArrayLimitedQueue](src/java/lasync/limitq/ArrayLimitedQueue.java) or [LinkedLimitedQueue](src/java/lasync/limitq/LinkedLimitedQueue.java).

The purpose of this tiny library is to be able to block on ".submit" whenever the q task limit is reached. Here is why..

## Why

If a regular [BlockingQueue](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/BlockingQueue.html) is used, 
a ThreadPoolExecutor calls queue's "[offer](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/BlockingQueue.html#offer\(E\))"
method which does not block: inserts a task and returns true, or returns false in case a queue is "capacity-restricted" and its capacity was reached.

While this behavior is useful, there are cases where we do need to _block_ and wait until a ThreadPoolExecutor has 
a thread available to work on the task. One reason could be an off heap storage that is being read and processed by a ThreadPoolExecutor:
e.g. there is no need, and sometimes completely undesired, to use JVM heap for something that is already available off heap.
Another good use is described in ["Creating a NotifyingBlockingThreadPoolExecutor"](https://today.java.net/pub/a/today/2008/10/23/creating-a-notifying-blocking-thread-pool-executor.html).

## How To

### Get it

To get lasync with Leiningen:

```clojure
[lasync "0.1.1"]
```

### Use it

To create a pool with limited number of threads and a backing q limit:

```clojure
(ns sample.project
  (:use lasync.core))

(def pool (limit-pool))
```

That is pretty much it. The pool is a regular [ExecutorService](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ExecutorService.html) that can have tasks submitted to it:

```clojure
(.submit pool #(+ 41 1))
```

By default lasync will create a number of threads and a blocking queue limit that matches the number of available cores:

```clojure
(defonce available-cores 
  (.. Runtime getRuntime availableProcessors))
```

But the number can be changed by:

```clojure
user=> (def pool (limit-pool :nthreads 42))
#'user/pool
user=> (def pool (limit-pool :limit 42))
#'user/pool
user=> (def pool (limit-pool :nthreads 42 :limit 42))
#'user/pool
```

## Show Me

To see lasync in action:

```clojure
lein repl
```

```clojure
user=> (use 'lasync.show)
```

```clojure
user=> (rock-on 69)  ;; Woodstock'69
```

```
INFO: pool q-size: 4, submitted: 1
INFO: pool q-size: 4, submitted: 3
INFO: pool q-size: 4, submitted: 2
INFO: pool q-size: 4, submitted: 0
INFO: pool q-size: 4, submitted: 4
INFO: pool q-size: 4, submitted: 5
INFO: pool q-size: 4, submitted: 6
INFO: pool q-size: 4, submitted: 7
...
...
INFO: pool q-size: 4, submitted: 62
INFO: pool q-size: 3, submitted: 60
INFO: pool q-size: 4, submitted: 63
INFO: pool q-size: 3, submitted: 65
INFO: pool q-size: 3, submitted: 64
INFO: pool q-size: 2, submitted: 66
INFO: pool q-size: 1, submitted: 67
INFO: pool q-size: 0, submitted: 68
```

Here lasync show was rocking on 4 core box (which it picked up on), so regardless of how many tasks are being pushed to it,
the queue max size always stays at 4, and lasync creates that back pressure in case the task q limit is reached. 
In fact the "blocking" can be seen in action, as each task is sleeping for a second, 
so the whole thing can be visually seen being processed by 4, pause, next 4, pause, etc..

Here is [the code](dev/show.clj) behind the show

## Tweaking the knobs

By default the limited blocking queue is ArrayLimitedQueue with 1024 element capacity, here's how to customize it

```clojure
(def lp (limit-pool :nthreads 1 :queue-size 128))
```

What if you want to use your own queue? No problem!

```clojure
(def lp (limit-pool :nthreads 1 :queue (LinkedLimitedQueue. 128)))
```

By default lasync's thread factory tries to have reasonable defaults but if you want to make your it's simply a matter
of reify'ing an interface.

```clojure
(def tpool (reify ThreadFactory
                 (newThread [_ runnable] ...)))

(def lp (limit-pool :nthreads 10 :thread-factory tpool))
```

Custom 'RejectedExecutionHandler' is equally as simple

```clojure

(def reh (reify RejectedExecutionHandler
             (rejectedExecution [_ runnable executor] ...)))

(def lp (limit-pool :nthreads 10 :rejected-handler reh))
```

Customize everything!

```clojure
(def n ...)

(def tpool (reify ThreadFactory
                 (newThread [_ runnable] ...)))

(def reh (reify RejectedExecutionHandler
             (rejectedExecution [_ runnable executor] ...)))

(def lp (limit-pool :nthreads n :thread-factory tpool :queue-size 2000 :rejected-handler reh))
```

## License

Copyright © 2013 tolitius

Distributed under the Eclipse Public License, the same as Clojure.
