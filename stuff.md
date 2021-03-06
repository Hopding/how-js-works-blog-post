The JavaScript language specification doesn't dictate the order in which these queues are to be serviced. This is left up to the designers of the JavaScript engine to decide. An engine might choose to handle all the events in its timer queue first, and only move onto the DOM event queue when the timer queue is empty. Or, the engine might interweave events from both queues.

It is also left to the engine designers to decide what happens when all event queues are empty. An engine might choose to exit (like NodeJS) or continue running and wait for some outside source to enqueue a new event (like web browsers).

# Simulating the Event Loop

To help solidify our discussion about the Event Loop, let's write some code to simulate it:

```
while (EventLoop.waitForTask()) {
  EventLoop.processNextTask()
}
```

But this code doesn't account for the fact that the Event Loop consists of multiple task queues. Let's add that:

```
while (EventLoop.waitForTask()) {
  const taskQueue = EventLoop.selectTaskQueue()
  if (taskQueue.hasNextTask()) {
    taskQueue.processNextTask()
  }
}
```

# Primary Sources

- https://www.ecma-international.org/ecma-262/9.0/index.html
- https://www.w3.org/TR/html52/webappapis.html

# Secondary Sources

- https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop
- https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/
- https://www.quora.com/Does-JavaScript-in-the-browser-have-the-equivalent-of-process-nextTick-or-setImmediate-in-node-js-or-do-we-just-have-setTimeout
- https://gist.github.com/Matt-Deacalion/4220997
- https://stackoverflow.com/questions/26615966/how-to-make-non-blocking-javascript-code
- http://latentflip.com/loupe/
- https://www.youtube.com/watch?v=8aGhZQkoFbQ
- https://stackoverflow.com/questions/38752620/promise-vs-settimeout
- https://blog.bitsrc.io/microtask-and-macrotask-a-hands-on-approach-5d77050e2168
- https://blog.risingstack.com/writing-a-javascript-framework-execution-timing-beyond-settimeout/
- https://stackoverflow.com/questions/24117267/nodejs-settimeoutfn-0-vs-setimmediatefn
- https://stackoverflow.com/questions/779379/why-is-settimeoutfn-0-sometimes-useful
- https://developer.mozilla.org/en-US/docs/Tools/Performance/Scenarios/Intensive_JavaScript
- https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/
- https://esdiscuss.org/topic/the-initialization-steps-for-web-browsers#content-16
- https://blog.sessionstack.com/how-javascript-works-the-building-blocks-of-web-workers-5-cases-when-you-should-use-them-a547c0757f6a
- https://blog.sessionstack.com/how-javascript-works-event-loop-and-the-rise-of-async-programming-5-ways-to-better-coding-with-2f077c4438b5
- https://stackoverflow.com/questions/2734025/is-javascript-guaranteed-to-be-single-threaded/2734311#2734311
- https://en.m.wikipedia.org/wiki/Reentrancy_(computing)
- https://stackoverflow.com/a/19699970

# Footnotes Example:

And now I'm going to describe[^1] something... Something about the JS Spec[^2].

[^1]: Yum yum yum!
[^2]: https://www.ecma-international.org/ecma-262/9.0/index.html
