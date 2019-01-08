What is JavaScript? How does it work?

Most JavaScript developers know what a call stack is. You've probably used Promises to write async code. You've likely heard that JavaScript is single-threaded. Maybe you've even heard about this thing called the "Event Loop".

But do you understand how JavaScript works under the hood? Are you comfortable thinking about JavaScript's [execution model](https://en.wikipedia.org/wiki/Execution_model)? A few weeks ago I would have answered no to both of these questions. But after a lot of research and experimentation, I now have a much better understanding. And I hope that after reaching the end of the post you will as well!

But before we get too abstract, let's write a bit of code. Most of us are familiar with the [prime number sequence](https://en.wikipedia.org/wiki/Prime_number), so let's write a function to compute it:

```js
function isPrime(n) {
  for (let i = 2; i < n; i++) {
    if (n % i === 0) return false;
  }
  return true;
}

function computePrimes(onPrime) {
  for (let currNum = 1; true; currNum++) {
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

Logging the primes is fun and all, but I'd rather see how many primes we've calculated than the values themselves. Let's make a simple website to render a live count for us!

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

Oh, that's too bad. We don't get a live `Total Primes Found` count. Instead the browser warns us that the webpage is running slow, and gives us the opportunity to stop it 🙁.

Our code is definitely running, you can see that by the logs it's printing. That means our `primesCountDiv.textContent = msg;` line is also executing. So why isn't our `primes-count` div updating? And not only is our div stuck, but the button is still rendered in the `pressed` state! It seems like the webpage isn't re-rendering properly for some reason 🤔.

Let's see if we can fix this with some magic 🎩🐇✨. Rewrite the `computePrimes()` function like so:

```js
async function computePrimes(onPrime) {
  for (let currNum = 1; true; currNum++) {
    if (isPrime(currNum)) onPrime(currNum);
    if (currNum % 500 === 0) await new Promise(setTimeout); // Magic‽
  }
}
```

Now what happens when we click the `Start Computing Primes` button?

![Webpage after running with magic](webpage_after_running_with_magic.png)

It works! But why...?

---

Sure, I'd written plenty of JS code for web apps, Node apps, and React Native apps. But I reached the limits of my understanding while working on a
