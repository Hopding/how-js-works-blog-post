Potential titles:

- How JS Works: The Event Loop
- How JS Works: `setTimeout`
- How Async JS Works
- Async JS In Depth
- JS `setTimeout` In Depth
- JavaScript's Execution Model

# &lt; Insert Title Here &gt;

...This is the first in a series of two posts about JavaScript's execution model. This post focuses on the Call Stack and Event Loop. The second post builds on this by discussing how Promises fit into the picture...

I've been writing JavaScript for awhile. I've written web services with Express, web apps with React, mobile apps with React Native, and I've authored libraries, such as [`pdf-lib`](https://github.com/Hopding/pdf-lib).

Recently, while working on `pdf-lib`, I found myself dealing with some long running synchronous code. I had optimized this code to run as fast as possible. Yet, when this code ran it would sometimes freeze the web page and cause the browser to warn about a script slowing down the browser.

I realized this function needed to be broken apart somehow. It needed to be asynchronous in order to allow other work to be done before it finished. But the function itself wasn't doing any fundamentally asynchronous work (it didn't make any HTTP requests or perform any file I/O).

How does one take a fundamentally synchronous task and make it asynchronous? I didn't know how to do this. To figure it out, I spent some time learning the intricacies of `setTimeout()`, `Promise.resolve()`, the Event Loop, and JavaScript's [execution model](https://en.wikipedia.org/wiki/Execution_model) in general.

My goal in writing this series of posts is to share what I learned. After reading through them, I hope that you will have a much better understanding of JavaScript's execution model, and how async JavaScript actually works under the hood.

# A Working Example

Let's start by writing some code to reproduce the problem I encountered while working on `pdf-lib`. Most of us are familiar with the [prime number sequence](https://en.wikipedia.org/wiki/Prime_number), so let's write a function to compute it:

```js
function isPrime(n) {
  for (let i = 2; i < n; i++) {
    if (n % i === 0) return false;
  }
  return true;
}

function computePrimes(onPrime, startAt = 1) {
  let currNum;
  for (currNum = startAt; true; currNum++) {
    if (isPrime(currNum)) onPrime(currNum);
  }
}
```

- The `isPrime()` function is very simple. It returns `true` if a number is prime, and `false` if it isn't.

- The `computePrimes()` function runs indefinitely. It checks every number from 1 to infinity to see if it's prime. Whenever one is found, it's passed to the `onPrime` callback.

Let's try it out:

```js
computePrimes(prime => {
  console.log(prime);
});
// => 1 2 3 5 7 11 13 ...
```

(If you run this in a Node REPL, you'll have to ctrl-c to stop it. In a browser, you can just close the tab)

Logging the primes is fun and all, but I'd rather see how many primes we've calculated instead of the values themselves. Let's make a simple website to render a live count for us!

<!-- prettier-ignore -->
```html
<!DOCTYPE html>
<html>
  <head><meta charset="utf-8" /></head>

  <body>
    <button onclick="startComputingPrimes()">Start Computing Primes</button>
    <div id="primes-count"></div>
  </body>

  <script type="text/javascript">
    /* Insert isPrime() and computePrimes() here... */

    let primesCount = 0;
    const primesCountDiv = document.getElementById('primes-count');

    function startComputingPrimes() {
      computePrimes((prime) => {
        primesCount += 1;
        if (primesCount % 500 === 0) {
          const msg = `Total Primes Found: ${primesCount}`;
          primesCountDiv.textContent = msg;
          console.log(msg);
        }
      });
    }
  </script>
</html>
```

When you load this webpage in a browser, you'll see the following:

![Webpage after first loading](webpage_after_loading.png)

Let's click the `Start Computing Primes` button and see what happens:

![Webpage after running for a short time](webpage_after_running.png)

Oh, that's too bad. We don't get a live `Total Primes Found` count. Instead the browser warns us that the webpage is running slow, and gives us the opportunity to stop it.

Our code is definitely running, you can see that by the logs it's printing. That means our `primesCountDiv.textContent = msg;` line is also executing. So why isn't our `primes-count` div updating? And not only is our div stuck, but the button is still rendered in the `pressed` state! It seems like the webpage isn't re-rendering for some reason...

Let's see if we can fix this with some magic 🎩🐇✨. Rewrite the `computePrimes()` function like so:

```js
function computePrimes(onPrime, startAt = 1) {
  let currNum;
  for (currNum = startAt; currNum % 500 !== 0; currNum++) {
    if (isPrime(currNum)) onPrime(currNum);
  }
  setTimeout(() => computePrimes(onPrime, currNum + 1), 0); // Magic‽
}
```

Now what happens when we click the `Start Computing Primes` button?

![Webpage after running with magic](webpage_after_running_with_magic.png)

It works! But why...? To answer this question, we need to talk about JavaScript's **Call Stack** and **Event Loop**. Let's start with the Call Stack.

# The Call Stack

The Call Stack is a fundamental part of the JavaScript language. It is a record-keeping structure that allows us to perform function calls. Each function call is represented as a **frame** on the call stack. This is how the JavaScript engine keeps track of which functions have been called and in what order. The JS engine uses this information to ensure execution picks back up in the right spot after a function returns.

When a JavaScript program first starts executing, the Call Stack is empty. When the first function call is made, a new frame is pushed onto the top of the Call Stack. When that function returns, its frame is popped off of the Call Stack. Let's look at an example.

Consider the following code snippet:

```js
function main() {
  doStuff('baz');
}

function doStuff(x) {
  doThings(x);
  foo();
}

function doThings(y) {
  console.log('Things done', y);
}

function foo() {
  return 'bar';
}

main();
```

Let's look at each transition made in the Call Stack while executing the above snippet:

```
| State 1 |   | State 2 |   |      State 3     |   |      State 4     |
|---------|   | main()  |   | doStuff(x='baz') |   | doThings(y=x)    |
              |---------|   | main()           |   | doStuff(x='baz') |
                            |------------------|   | main()           |
                                                   |------------------|

|      State 5     |   |      State 6     |   | State 7 |   | State 8 |
| foo()            |   | doStuff(x='baz') |   | main()  |   |---------|
| doStuff(x='baz') |   | main()           |   |---------|
| main()           |   |------------------|
|------------------|
```

This visualization of the Call Stack is familiar to most of us. We all have an intuitive feel for what's going on here. However, the Call Stack is only part of JavaScript's execution model. It doesn't tell the full story. Consider this snippet:

```js
const logA = () => console.log('A');
const logB = () => console.log('B');
const logC = () => console.log('C');

logA();
setTimeout(logB, 100);
logC();

// => A C B
```

How is it that `B` is logged last? The Call Stack always works in order. But what we see here is happening out of order.

`setTimeout` is responsible for the out-of-order logging we're seeing. What we've done is told JavaScript to call `logB` in 100 milliseconds. And because computers are fast, logC() will have been called long before 100 milliseconds have expired.

But what if we pass a timeout of 0 milliseconds?

```js
logA();
setTimeout(logB, 0);
logC();
// => A C B
```

Interesting! The same thing still happens. To understand why, we need to understand what `setTimeout` is actually doing under the hood. Clearly, it's not your typical function. Its behavior cannot be explained with the Call Stack alone. To explain how `setTimeout` works, we need to talk about the **Event Loop**.

# The Event Loop

If the Call Stack keeps track of the functions that are executing right now, then the Event Loop keeps track of functions that are going to be executed in the future. The Event Loop consists of queues of functions. When the Call Stack is empty, the JavaScript engine picks a queue and removes the oldest function from it. The engine then invokes this function, which fills up the Call Stack. When this function finishes running and the Call Stack is empty again, another function is taken from a queue, then it is invoked, and so on.

Let's visualize our last code snippet with a Call Stack and an Event Queue:

```
=========== State 1 =================
|  Call Stack  |   | Function Queue |
|--------------|   |----------------|

=========== State 2 =================
|  Call Stack  |   | Function Queue |
| logA()       |   |----------------|
|--------------|

=========== State 3 =================
|  Call Stack  |   | Function Queue |
| setTimeout() |   | logB           |
|--------------|   |----------------|

=========== State 4 =================
|  Call Stack  |   | Function Queue |
| logC()       |   | logB           |
|--------------|   |----------------|

=========== State 5 =================
|  Call Stack  |   | Function Queue |
|--------------|   | logB           |
                   |----------------|

=========== State 6 =================
|  Call Stack  |   | Function Queue |
| logB()       |   |----------------|
|--------------|
```

In this example, we're only visualizing a single callback queue. However, most JavaScript engines use multiple queues to manage events. For example, a browser might have one queue for timer callbacks, and another queue for DOM event callbacks.

The JavaScript language specification doesn't dictate the order in which these queues are to be serviced. This is left up to the designers of the JavaScript engine to decide. An engine might choose to handle all the events in its timer queue first, and only move onto the DOM event queue when the timer queue is empty. Or, the engine might interweave events from both queues.

It is also left to the engine designers to decide what happens when all event queues are empty. An engine might choose to exit (like NodeJS) or continue running and wait for some outside source to enqueue a new event (like web browsers).

# One Event at a Time

JavaScript has a single Call Stack. Because of this, the Event Loop is only allowed to process one event at a time. This makes for a relatively simple execution model that saves JavaScript from a host of concurrency issues.

Consider [reentrancy](<https://en.wikipedia.org/wiki/Reentrancy_(computing)>), defined by Wikipedia as follows:

> ... a computer program or subroutine is called reentrant if it can be interrupted in the middle of its execution and then safely be called again ("re-entered") before its previous invocation's complete execution. The interruption could be caused by an internal action such as a jump or call, or by an external action such as an interrupt or signal. Once the reentered invocation completes, the previous invocations will resume correct execution.

JavaScript programmers don't have to worry about making their functions reentrant, because they can never be interrupted[^1]! JS functions always run to completion.

[^1]: This isn't strictly true. If a function is recursive it can be entered a second time before the initial invocation is complete. However, this is not a concurrency issue forced upon the developer by the runtime. This scenario isn't representative of the context in which reentrancy is typically a concern for developers.

However, this simple execution model doesn't come without risks.

Suppose a "rogue" callback makes its way from an event queue onto the Call Stack. This rogue callback never finishes running, and occupies the Call Stack indefinitely. If this happens, the rogue callback is able to prevent all other events from ever being processed. This includes critical events, such as rendering callbacks!

For this reason, web browsers monitor how much time callbacks spend on the Call Stack. If a callback takes too long to finish, the browser alerts the user and gives them the option to "Stop It" - removing the callback from the Call Stack and allowing other events to be processed.

# How our Magic Works

Let's return to our example. Before we added magic to it, our code was causing the webpage to freeze. And after a bit of time, the browser gave us the option to stop it. This happened because our `computePrimes()` function went rogue. After it was placed on the Call Stack it never finished running. It blocked the Call Stack and prevented rendering events from being processed.

We were able to fix this by adding some magic. Of course, it wasn't _really_ magic. And now that we've talked about the Call Stack and Event Loop, we can understand how it actually works.

We started with a single infinitely long function call. And our magic broke it up into a series of short running callbacks. Each callback would compute 500 primes and then enqueue an event to compute the next 500 primes (using `setTimeout(fn, 0)`). This allows the JS engine to handle other events that have been enqueued in-between prime calculations.

Without Magic:

```
|    Call Stack   |   | Event Queue |
| computePrimes() |   | rerender    |
|-----------------|   | rerender    |
                      | rerender    |
                      | ...         |
                      |-------------|
```

With Magic:

```
|    Call Stack   |   |  Event Queue  |
| computePrimes() |   | rerender      |
|-----------------|   | computePrimes |
                      | rerender      |
                      | computePrimes |
                      | rerender      |
                      | ...           |
                      |---------------|
```
