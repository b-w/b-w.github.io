---
layout: post
title: "FastArg: a look at the code"
categories: blog
---

In a previous post I've discussed _FastArg_, a small utility for efficiently handling command-line arguments. Today I'll briefly go over the code behind the tool. I'll also include a full code listing.

FastArg basically consists of two classes: `FastArg`, which is where the logic resides; and `FastArgHandler`, which is just a data structure for storing the handler information. We'll look at the latter first:

```csharp
public struct FastArgHandler : IEquatable<FastArgHandler>
{
    public FastArgHandler(string key, int argumentOrder, int executionPriority, bool required, Delegate handler)
        : this()
    {
        Key = key;
        ArgumentOrder = argumentOrder;
        ExecutionPriority = executionPriority;
        Required = required;
        Handler = handler;
    }

    public string Key { get; private set; }

    public int ArgumentOrder { get; private set; }

    public int ExecutionPriority { get; private set; }

    public bool Required { get; private set; }

    public Delegate Handler { get; private set; }

    #region IEquatable members

    // boring code here

    #endregion

    public override string ToString()
    {
        return String.Format(CultureInfo.InvariantCulture, "{0}({1})", Key, Handler.Method.GetParameters().Length);
    }
}
```

It's little more than a collection of variables, which hold the configuration for this particular handler. The handler itself is stored as a delegate. I've left out the implementation code for the `IEquatable` members, seeing as it suffices to just say that a two handlers are equal if and only if their keys and the number of arguments of their handlers are equal. Of note is the seemingly useless call to the default constructor, which is necessary fields and methods of structs cannot be used until they have been initialized by the default constructor.

The FastArg class itself is a little more interesting. First of all there's a few fields in which the configuration is stored:

```csharp
private readonly string m_switchPrefix;
private readonly string m_bracketOpen;
private readonly string m_bracketClose;
private readonly Dictionary<string, List<FastArgHandler>> m_handlers;
```

All the options discussed in the previous post are here. There's also a function for adding new handlers, which we won't discuss. The interesting stuff is all in the `Handle` function. We first set up a number of variables:

```csharp
var unusedHandlers = new List<FastArgHandler>();
var usedHandlers = new List<Tuple<FastArgHandler, string[]>>();

foreach (var handlerList in m_handlers.Values) {
    unusedHandlers.AddRange(handlerList);
}

var currentKey = String.Empty;
var currentArgs = new List<string>();
var currentMaxArgLength = 0;

var bOpen = false;
```

Throughout the `Handle` function we keep a list of handlers we have not yet used, as well as a list of handlers we have used along with the argument(s) that we have found for this handler. The remaining variables represent the state of the parser: `currentKey` is the last switch we have seen, `currentArgs` is the list of arguments we have encountered so far but which have not yet been used, `currentMaxArgLength` is a helper variable which contains the highest number of arguments among any handler matching the current switch, and `bOpen` is simple flag indicating whether we've currently seen an opening bracket which has not yet been closed.

Next up is the main parser function, which simply loops over the given arguments in a left-to-right fashion:

```csharp
for (int i = 0; i < args.Length; i++) {
    var arg = args[i];
```

This way of "parsing" the input is straightforward to implement, but also has its limits because we can't do any backtracking. Still, I feel that this is sufficient given our current goals. The first thing we do when parsing a single argument (or token) is check if it is an opening or closing bracket:

```csharp
...

    //
    // see if it's an opening bracket

    if (arg.Equals(m_bracketOpen, StringComparison.OrdinalIgnoreCase)) {
        if (!bOpen) {
            bOpen = true;
            continue;
        } else {
            throw new FastArgException("Parse error: unexpected opening bracket.");
        }
    }

    //
    // see if it's a closing bracket

    if (arg.Equals(m_bracketClose, StringComparison.OrdinalIgnoreCase)) {
        if (bOpen) {
            bOpen = false;
            AddNextHandler(unusedHandlers, usedHandlers, ref currentKey, currentArgs);
            continue;
        } else {
            throw new FastArgException("Parse error: unexpected closing bracket.");
        }
    }
```

We simply toggle the bracket flag in these cases. If the token is not a bracket, we next see if it's a switch:

```csharp
...

    //
    // see if it's a switch

    if (!bOpen && arg.StartsWith(m_switchPrefix, StringComparison.OrdinalIgnoreCase)) {
        if (!String.IsNullOrEmpty(currentKey) || currentArgs.Count > 0) {
            AddNextHandler(unusedHandlers, usedHandlers, ref currentKey, currentArgs);
        }

        var key = arg.Substring(m_switchPrefix.Length);
        if (key.Length > 0) {
            if (unusedHandlers.Any(h => h.Key.StartsWith(key, StringComparison.OrdinalIgnoreCase))) {
                currentKey = key;
                currentMaxArgLength = unusedHandlers.Where(h => h.Key.StartsWith(key, StringComparison.OrdinalIgnoreCase))
                                                    .Max(h => h.Handler.Method.GetParameters().Length);
                continue;
            } else {
                throw new FastArgException(String.Format(CultureInfo.InvariantCulture, "Unknown switch \"{0}\".", key));
            }
        } else {
            throw new FastArgException("Parse error: zero-length switch encountered.");
        }
    }
```

Handling the switch is relatively straightforward. First of all, we check if we have already encountered other switches or arguments prior to this one. If we have, we call the `AddNextHandler` function, which goes through our list of unused handlers and finds one which matches the given switch and number of arguments. It then adds this handler/arguments combo to our list of used handlers, removes the handler from the list of unused handlers, and clears the list of arguments. Next, all we basically do here is store the new switch in our `currentKey` variable and recompute the `currentMaxArgLength` variable.

Finally, if the token was not a bracket or a switch, it must be an argument:

```csharp
...

    //
    // if it isn't any of the above, it gets treated as an argument

    if (!String.IsNullOrEmpty(currentKey) && currentArgs.Count == currentMaxArgLength) {
        if (bOpen) {
            throw new FastArgException(String.Format(CultureInfo.InvariantCulture, "Invalid number of arguments for switch \"{0}\".", currentKey));
        } else {
            AddNextHandler(unusedHandlers, usedHandlers, ref currentKey, currentArgs);
        }
    }

    currentArgs.Add(arg);

    if (!bOpen) {
        if (String.IsNullOrEmpty(currentKey) || currentArgs.Count == currentMaxArgLength) {
            AddNextHandler(unusedHandlers, usedHandlers, ref currentKey, currentArgs);
        }
    }
}
```

The first thing we do is check if we have already encountered a switch and we have currently reached the maximum number of arguments among the handlers matching the switch. If so, we make another call to `AddNextHandler`. We then add the argument to the list of arguments. There's also a few checks involving the brackets used for argument grouping. Most importantly, if we are not currently handling a named switch (and are therefor in anonymous arguments mode), then we can only deal with handlers taking a single argument. We also do an additional check after adding the argument to see if we've reached the maximum number of arguments taken by any current switch.

And that's the entire parsing loop. Pretty simple, right? After exiting, there's a few other checks to do but once those have passed all that remains is invoking the handlers.

```csharp
//
// handle trailing zero-argument switch

if (!String.IsNullOrEmpty(currentKey)) {
    AddNextHandler(unusedHandlers, usedHandlers, ref currentKey, currentArgs);
}

if (unusedHandlers.Any(h => h.Required)) {
    throw new FastArgException("One or more required switches are not provided.");
}

var sortedHandlers = from h in usedHandlers
                        orderby h.Item1.ExecutionPriority ascending
                        select h;

foreach (var handler in sortedHandlers) {
    handler.Item1.Handler.DynamicInvoke(handler.Item2);
}
```

And that's been most of what makes FastArg tick. It's nothing terribly complex, but nevertheless nice to have lying around for when you need it. It certainly beats having to spend the additional hour or two to come up with something yourself.

Full code listing:

## FastArgException.cs

```csharp
using System;
using System.Runtime.Serialization;

namespace FastArg
{
    [Serializable]
    public class FastArgException : Exception
    {
        public FastArgException()
        {
        }

        public FastArgException(string message)
            : base(message)
        {
        }

        public FastArgException(string message, Exception inner)
            : base(message, inner)
        {
        }

        protected FastArgException(SerializationInfo info, StreamingContext context)
            : base(info, context)
        {
        }
    }
}
```

## FastArgHandler.cs

```csharp
using System;
using System.Globalization;

namespace FastArg
{
    public struct FastArgHandler : IEquatable<FastArgHandler>
    {
        public FastArgHandler(string key, int argumentOrder, int executionPriority, bool required, Delegate handler)
            : this()
        {
            Key = key;
            ArgumentOrder = argumentOrder;
            ExecutionPriority = executionPriority;
            Required = required;
            Handler = handler;
        }

        public string Key { get; private set; }

        public int ArgumentOrder { get; private set; }

        public int ExecutionPriority { get; private set; }

        public bool Required { get; private set; }

        public Delegate Handler { get; private set; }

        #region IEquatable members

        public bool Equals(FastArgHandler other)
        {
            return (Key == other.Key
                    && Handler.Method.GetParameters().Length == other.Handler.Method.GetParameters().Length);
        }

        public override bool Equals(object obj)
        {
            if (obj == null) {
                return false;
            }
            if (obj is FastArgHandler) {
                return this.Equals((FastArgHandler)obj);
            }

            return false;
        }

        public override int GetHashCode()
        {
            return Key.GetHashCode() ^ Handler.Method.GetParameters().Length;
        }

        public static bool operator ==(FastArgHandler left, FastArgHandler right)
        {
            if (object.ReferenceEquals(left, right)) {
                return true;
            }
            if ((object)left == null || (object)right == null) {
                return false;
            }
            return left.Equals(right);
        }

        public static bool operator !=(FastArgHandler left, FastArgHandler right)
        {
            return !(left == right);
        }

        #endregion

        public override string ToString()
        {
            return String.Format(CultureInfo.InvariantCulture, "{0}({1})", Key, Handler.Method.GetParameters().Length);
        }
    }
}
```

## FastArg.cs

```csharp
using System;
using System.Collections.Generic;
using System.Globalization;
using System.Linq;

namespace FastArg
{
    public class FastArg
    {
        private readonly string m_switchPrefix;
        private readonly string m_bracketOpen;
        private readonly string m_bracketClose;
        private readonly Dictionary<string, List<FastArgHandler>> m_handlers;

        public FastArg(string switchPrefix, string bracketOpen, string bracketClose)
        {
            m_switchPrefix = switchPrefix;
            m_bracketOpen = bracketOpen;
            m_bracketClose = bracketClose;
            m_handlers = new Dictionary<string, List<FastArgHandler>>();
        }

        public bool Add(string key, int argumentOrder, int executionPriority, bool required, Delegate handler)
        {
            if (key == null) {
                throw new ArgumentNullException("key");
            }

            if (String.IsNullOrWhiteSpace(key)) {
                throw new ArgumentOutOfRangeException("key", "Switches must be non-empty strings.");
            }

            if (handler == null) {
                throw new ArgumentNullException("handler");
            }

            if (!m_handlers.ContainsKey(key)) {
                m_handlers.Add(key, new List<FastArgHandler>());
            }

            if (m_handlers[key].Any(h => h.Equals(handler))) {
                return false;
            } else {
                m_handlers[key].Add(new FastArgHandler(key, argumentOrder, executionPriority, required, handler));
                return true;
            }
        }

        public void Handle(string[] args)
        {
            if (args == null) {
                throw new ArgumentNullException("args");
            }

            if (args.Length == 0) {
                return;
            }

            var unusedHandlers = new List<FastArgHandler>();
            var usedHandlers = new List<Tuple<FastArgHandler, string[]>>();

            foreach (var handlerList in m_handlers.Values) {
                unusedHandlers.AddRange(handlerList);
            }

            var currentKey = String.Empty;
            var currentArgs = new List<string>();
            var currentMaxArgLength = 0;

            var bOpen = false;

            for (int i = 0; i < args.Length; i++) {
                var arg = args[i];

                //
                // see if it's an opening bracket

                if (arg.Equals(m_bracketOpen, StringComparison.OrdinalIgnoreCase)) {
                    if (!bOpen) {
                        bOpen = true;
                        continue;
                    } else {
                        throw new FastArgException("Parse error: unexpected opening bracket.");
                    }
                }

                //
                // see if it's a closing bracket

                if (arg.Equals(m_bracketClose, StringComparison.OrdinalIgnoreCase)) {
                    if (bOpen) {
                        bOpen = false;
                        AddNextHandler(unusedHandlers, usedHandlers, ref currentKey, currentArgs);
                        continue;
                    } else {
                        throw new FastArgException("Parse error: unexpected closing bracket.");
                    }
                }

                //
                // see if it's a switch

                if (!bOpen && arg.StartsWith(m_switchPrefix, StringComparison.OrdinalIgnoreCase)) {
                    if (!String.IsNullOrEmpty(currentKey) || currentArgs.Count > 0) {
                        AddNextHandler(unusedHandlers, usedHandlers, ref currentKey, currentArgs);
                    }

                    var key = arg.Substring(m_switchPrefix.Length);
                    if (key.Length > 0) {
                        if (unusedHandlers.Any(h => h.Key.StartsWith(key, StringComparison.OrdinalIgnoreCase))) {
                            currentKey = key;
                            currentMaxArgLength = unusedHandlers.Where(h => h.Key.StartsWith(key, StringComparison.OrdinalIgnoreCase))
                                                                .Max(h => h.Handler.Method.GetParameters().Length);
                            continue;
                        } else {
                            throw new FastArgException(String.Format(CultureInfo.InvariantCulture, "Unknown switch \"{0}\".", key));
                        }
                    } else {
                        throw new FastArgException("Parse error: zero-length switch encountered.");
                    }
                }

                //
                // if it isn't any of the above, it gets treated as an argument

                if (!String.IsNullOrEmpty(currentKey) && currentArgs.Count == currentMaxArgLength) {
                    if (bOpen) {
                        throw new FastArgException(String.Format(CultureInfo.InvariantCulture, "Invalid number of arguments for switch \"{0}\".", currentKey));
                    } else {
                        AddNextHandler(unusedHandlers, usedHandlers, ref currentKey, currentArgs);
                    }
                }

                currentArgs.Add(arg);

                if (!bOpen) {
                    if (String.IsNullOrEmpty(currentKey) || currentArgs.Count == currentMaxArgLength) {
                        AddNextHandler(unusedHandlers, usedHandlers, ref currentKey, currentArgs);
                    }
                }
            }

            if (bOpen) {
                throw new FastArgException("Parse error: expected closing bracket not found.");
            }

            //
            // handle trailing zero-argument switch

            if (!String.IsNullOrEmpty(currentKey)) {
                AddNextHandler(unusedHandlers, usedHandlers, ref currentKey, currentArgs);
            }

            if (unusedHandlers.Any(h => h.Required)) {
                throw new FastArgException("One or more required switches are not provided.");
            }

            var sortedHandlers = from h in usedHandlers
                                    orderby h.Item1.ExecutionPriority ascending
                                    select h;

            foreach (var handler in sortedHandlers) {
                handler.Item1.Handler.DynamicInvoke(handler.Item2);
            }
        }

        protected static void AddNextHandler(List<FastArgHandler> unusedHandlers, List<Tuple<FastArgHandler, string[]>> usedHandlers, ref string currentKey, List<string> currentArgs)
        {
            FastArgHandler nextHandler;

            if (String.IsNullOrEmpty(currentKey)) {
                nextHandler = GetHandlerByArgumentOrder(unusedHandlers, currentArgs.Count);
            } else {
                nextHandler = GetHandlerByKey(unusedHandlers, currentKey, currentArgs.Count);
            }

            usedHandlers.Add(Tuple.Create(nextHandler, currentArgs.ToArray()));
            unusedHandlers.Remove(nextHandler);

            currentKey = String.Empty;
            currentArgs.Clear();
        }

        protected static FastArgHandler GetHandlerByArgumentOrder(IEnumerable<FastArgHandler> handlers, int argCount)
        {
            var candidates = (from h in handlers
                                where h.Handler.Method.GetParameters().Length == argCount
                                orderby h.ArgumentOrder ascending
                                select h).ToList();

            if (candidates.Count == 0) {
                throw new FastArgException("No matching switch found for anonymous argument(s).");
            }
            if (candidates.Count > 1 &&
                candidates[0].ArgumentOrder == candidates[1].ArgumentOrder) {
                throw new FastArgException("Ambiguous switch for anonymous argument(s).");
            }

            return candidates[0];
        }

        protected static FastArgHandler GetHandlerByKey(IEnumerable<FastArgHandler> handlers, string key, int argCount)
        {
            var candidates = (from h in handlers
                                where h.Handler.Method.GetParameters().Length == argCount
                                    && h.Key.StartsWith(key, StringComparison.OrdinalIgnoreCase)
                                select h).ToList();

            if (candidates.Count == 0) {
                throw new FastArgException(String.Format(CultureInfo.InvariantCulture, "Invalid number of arguments or unknown switch \"{0}\".", key));
            }
            if (candidates.Count > 1) {
                throw new FastArgException(String.Format(CultureInfo.InvariantCulture, "Ambiguous switch \"{0}\".", key));
            }

            return candidates[0];
        }
    }
}
```

Enjoy!
