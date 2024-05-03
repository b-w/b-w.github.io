---
layout: post
title: "Implementing an async producer/consumer queue using BlockingCollection"
categories: blog
---

In this post, I'd like to shine some light on a relatively unknown class from the .NET framework: [BlockingCollection](https://msdn.microsoft.com/en-us/library/dd267312(v=vs.110).aspx). I'll use this powerful class to implement an asynchronous, multi-threaded, in-memory producer/consumer queue using just a few lines of code.

## Use case

First off, let's consider an extremely common real-world use case: logging. Every app or service should have it. In order to minimize the impact that logging has on your performance, a good idea is to implement it in an asynchronous way. For this scenario, the producer/consumer pattern is a natural fit.

If you have a multi-threaded application, like a web service, it should be obvious that there can be multiple producers, in independent threads, all generating logging events. However, you'll likely only want a single consumer, for example to write the events to a file on disk. Something like this:

![Producer/Consumer queue](/assets/img/blog/2017/01/Producer-Consumer.PNG)

The producers all add their logging events to a queue, while the consumer removes them one by one for processing (e.g. writing to disk or to a database). Having only a single consumer is important if we want to ensure the log events are processed in the same order they arrived in (which we do).

We'll implement such a queue using the BlockingCollection class.

## BlockingCollection

The BlockingCollection class is a thread-safe collection that supports the notion of _blocking_. The class has a few different functions for adding and removing items, and these functions can all _block_ (i.e. pause execution of the current thread) until some condition is satisfied.

The Add() functions can block if the collection is full. This can only happen in the case where the collection has a maximum size. We won't be using a bounded collection in our implementation.

The Take() functions block if the collection is empty. This means a call to Take() will wait, potentially indefinitely, until there is something to return.

It should be obvious that these functions are perfect for implementing an async producer/consumer queue.

## A simple producer/consumer queue

The following is my implementation of a simple producer/consumer queue using BlockingCollection.

```csharp
using System;
using System.Collections.Concurrent;
using System.Threading;

public class AsyncProducerConsumerQueue<T> : IDisposable
{
    private readonly Action<T> m_consumer;
    private readonly BlockingCollection<T> m_queue;
    private readonly CancellationTokenSource m_cancelTokenSrc;

    public AsyncProducerConsumerQueue(Action<T> consumer)
    {
        if (consumer == null)
        {
            throw new ArgumentNullException(nameof(consumer));
        }

        m_consumer = consumer;
        m_queue = new BlockingCollection<T>(new ConcurrentQueue<T>());
        m_cancelTokenSrc = new CancellationTokenSource();

        new Thread(() => ConsumeLoop(m_cancelTokenSrc.Token)).Start();
    }

    public void Produce(T value)
    {
        m_queue.Add(value);
    }

    private void ConsumeLoop(CancellationToken cancelToken)
    {
        while (!cancelToken.IsCancellationRequested)
        {
            try
            {
                var item = m_queue.Take(cancelToken);
                m_consumer(item);
            }
            catch (OperationCanceledException)
            {
                break;
            }
            catch (Exception ex)
            {
                Console.Error.WriteLine(ex);
            }
        }
    }

    #region IDisposable

    private bool m_isDisposed;

    protected virtual void Dispose(bool disposing)
    {
        if (!m_isDisposed)
        {
            if (disposing)
            {
                m_cancelTokenSrc.Cancel();
                m_cancelTokenSrc.Dispose();
                m_queue.Dispose();
            }

            m_isDisposed = true;
        }
    }

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    #endregion
}
```

It takes a consumer delegate in its constructor, and exposes a single function: `Produce()`. A call to `Produce()` simply adds the given item to the `BlockingCollection`. Meanwhile, a single consumer thread is started in the constructor. This thread contains an infinite loop that keeps calling `Take()` on the `BlockingCollection`, and processing the items it finds using the consumer delegate. `IDisposable` is implemented to gracefully kill the consumer thread once the queue is disposed.

In the end, we leveraged a powerful data structure from the .NET framework to implement an asynchronous producer/consumer queue in just a few lines of code. Not bad for a Sunday.
