FuturesJS
====

FuturesJS is a JavaScript library which (when used as directed) simplifies handling Callbacks, Errbacks, Promises, Subscriptions, Joins, Chains, Sequences, Asynchronous Method Queues, Synchronization of asynchronous data, and Eventually Consistent data.

Installing / Loading
====

Download the file `lib/futures.all.js` and include it in your application.

In a browser:

    <script src='vendor/persevere/global-es5.js'></script>
    <script src='lib/futures.all.js'></script>
    var Futures = require('futures'); // comes with thin require wrapper

Consider http://jscompress.com/ for minify-ing (7K packed, 3.6K gzipped).

In Node.js:

    npm install futures
    node> var Futures = require('futures');

For Rhino you will need `env.js` as Futures utilizes `setTimeout` and its friends.

FYI: FuturesJS does pass JSLint regularly (but not every single commit)

Getting Started
----

Look in the `examples` directory and in `test.html` for simple examples.

Post questions, bugs, and stuff you want to share on the [(Google Groups) Mailing List](http://groups.google.com/group/futures-javascript)

API
====

Overview
----

`asyncify`, `chainify`, `join`, `loop`, `promise`, `sequence`, `subscription`, `subscription2promise`, `synchronize`, `trigger`, `whilst`

Futures.chainify(providers, consumers, context, params) / Futures.futurify()
---------------

Asynchronous method queueing allows you to chain actions on data which may or may not be readily available.
This is how Twitter's @Anywhere api works.

You might want a model which remotely fetches data in this fashion:

    Contacts.all(params).randomize().limit(10).display();
    Contacts.one(id, params).display();

Which could be implemented like so:

    var Contacts = Futures.chainify({
      // Providers must be promisables
      all: function(params) {
        var p = Futures.promise();
        $.ajaxSetup({ error: p.smash });
        $.getJSON('http://graph.facebook.com/me/friends', params, p.fulfill);
        $.ajaxSetup({ error: undefined });
        return p.passable();
      },
      one: function(id, params) {
        var p = Futures.promise();
        $.ajaxSetup({ error: p.smash });
        $.getJSON('http://graph.facebook.com/' + id, params, p.fulfill);
        $.ajaxSetup({ error: undefined });
        return p.passable();
      }
    },{
      // Consumers will be called in synchronous order
      // with the `lastResult` of the previous provider or consumer.
      // They should return either lastResult or a promise
      randomize: function(data, params) {
        data.sort(function(){ return Math.round(Math.random())-0.5); // Underscore.js
        return Futures.promise(data); // Promise rename to `immediate`
      },
      limit: function(results, n, params) {
        var data = results[0]; // array of args
        
        data = data.first(n); // underscore's limit function
        return Futures.promise(data);
      },
      display: function(data, params) {
        var data = results[0];

        $('#friend-area').render(directive, data); // jQuery+PURE
        // always return the data, even if you don't modify it!
        // otherwise your results could be unexpected

        return data;
      }
    });

Things to know:

  * `providers` - promisables which return data
  * `consumers` - functions which use and or change data
    * the first argument must be `data`
    * when returning a promisable the next method in the chain will not execute until the promise is fulfilled
    * when returning a "literal object" the next method in the chain will use that object
    * when returning `undefined` (or not returning anything) the next method in the chain will use the defined object
  * `context` - `apply()`d to each provider and consumer, thus becoming the `this` object
  * `params` - reserved for future use

Futures.promise() -- create a chainable promise object
-----------------

Creates a promise object

If guarantee (optional) is passed, an immediate (an already fulfilled promise) is returned instead

    Futures.promise(var guarantee)
        // Call all callbacks passed into `when` in the order they were received and pass result
        .fulfill(var result)
        // Returns immediately if the result is available, or as soon as it becomes available
        .when(function (result) {})
        // Call all errbacks passed into `fail` in the order they were received and pass error
        .smash(var error)
        //
        .fail(function (error) {})


Futures.subscription() -- create a subscription object
----------------------

Subscriptions may be delivered or held multiple times

    Futures.subscription()
        // delivers `data` to all subscribers
        .deliver(data) 
        // receives `data` each time deliver is called
        // returns unsubscribe(); 
        .subscribe(callback);
        // notifies that the subscription is "on hold"
        .hold(error)
        // receives notification on failure
        .miss(errback)

Futures.subscription()(*unsubscribe*)()(*resubscribe*)()...
-----------------------------------------------------

It is possible to put a single subscription "on hold" by calling the anonymous `unsubscribe()` function which is returned.
The same subscription can be resumed by calling its return function. The cycle continues indefinitely.

    var s,
    unsubscribe,
    resubscribe;

    s = Futures.subscription();
    unsubcribe = s.subscribe(callback);
    resubscribe = unsubscribe();
    unsubscribe = resubscribe();


Futures.subscrpition2promise() -- create a promise from a subscription
------------------------------

Pass in a subscription or subscribable and get back a promise. This is what Futures.join() uses internally to allow the joining of subscriptions.

    var promise = Futures.subscription2promise(subscription);


Futures.trigger() -- create an anonymous event listener / triggerer
-----------------

    var t = Futures.trigger();
    var mute = t.listen(callback1)
    var mute2 = t.listen(callback2);
    t.fire();

TODO chain mutes such that the last .listen returns all mutes?


Futures.join() -- create a promise joined from two or more promises / subscriptions
--------------

Joins return a promise which triggers when all joined promises (and subscriptions) have been fulfilled or smashed.

Join accepts both promises and subscriptions. One-time self-unsubscribing promises are generated automatically.

    // params is optional
    params = { 
      timeout : undefined // time in ms, undefined by default
    }

    Futures.join(promise1, promise2, subscription3, ..., params);
        .when(function (result1, result2, result3) {
            var arg0, arg1, arg2;
            arg0 = result1[0];
            arg1 = result1[1];
            arg2 = result1[2];
            // Each result is the arguments array of the results of the promise
         });
    Futures.join([p1, p2, p3, ...], params);
        .when(function (result_array) {
            // result_array holds an array of argument arrays for [p1, p2, p3] in order
         });


Futures.synchronize() -- create a subscription synchronized with two or more subscriptions
---------------------

Synchronizations trigger each time all of the subscriptions have delivered or held at least one new subscription

If s1 were to deliver 4 times before s2 and s3 deliver once, the 4th delivery is used

    var s = Futures.synchronize(s1, s2, s3, ...);
    s.subscribe(function (r1,r2,r3) {
        // most recent results returned in order
    });
    s = Futures.synchronize([s1, s2, s3, ...]);
    s.subscribe(function (s_arr) {
        // s_arr holds the most recent results of [s1, s2, s3, ...]
    });


Futures.sequence() -- chain two or more asynchronous (and synchronous) functions
------------------

Instead of nesting callbacks 10 levels deep, pass `fulfill` instead.

Each next function receives the previous result and an array of all previous results

    Futures.sequence(function (fulfill) {
          // fulfill is Futures.promise().fulfill

          fulfill("I'm ready.");
        })
        .then(function (fulfill, ready) {
          // ready === "I'm ready."

          fulfill(ready, "... and waiting");
        })
        .then(function (fulfill, ready, waiting) {
          // ready === "I'm ready."
          // waiting === "... and waiting"
        
          // this being the last in the sequence, `fulfill` is optional
        });

Futures.whilst() -- begin a "safe" loop with timeout, sleep, and max loop options
----------------

A breakable, timeoutable, asynchronous non-blocking while loop. 

Warning: this is too slow for long running loops (4ms+ intervals minimum)

Note: The next loop iteration will not occur until `this.breakIf()` has been called.

    Futures.whilst(function (previousResult) {
            // breakIf *MUST* be called once per loop. It may be called asynchronously.
            // expression may be something such as (i < 100)
            // letFinish = true will allow this iteration of the loop to finish 
            this.breakIf(expression, letFinish);
        }, {
            // You may control the length of runtime as well as the pause between loops here
            interval : 1, // how long to wait before executing the next loop. Due to "clamping" this is always >= 4ms,
            timeout : undefined, // forcefully break the loop after N ms (cumulative; not per-loop)
            maxLoops : undefined // forcefully break the loop after N iterations
        })
        .then(function (callback, previousResult, index, [result0, result1, ...]))
        .when(function (data) {})
        .breakNow(); // forcefully break the loop immediately

Example: This loop will try to interate every 1ms, but it will return early (without incrementing the loop count) nearly 200 times before `this.breakIf()` is called. It will timeout after 1 second, before the break condition is true.

    var i = 0;
    Futures.whilst(function (previousResult) {
      var that = this;
      setTimeout(function () {
        i += 1;
        that.breakIf(5 < i);
      }, 200);
    }, {interval:1, timeout: 1000, maxLoops: 10});


TODO: consider iteration timeout vs cumulative timeout

Futures.loop()
--------------

A breakable, timeoutable, asynchronous do loop. 

Warning: this is too slow for long running loops (4ms+ intervals minimum)
 
     Futures.whilst(function (previousResult) {
            // expression may be something such as (i < 100)
            // letFinish = true will allow this iteration of the loop to finish 
            this.until(expression, letFinish); // break when true
            this.whilst(expression, letFinish); // break when false
        }, {
            // You may control the length of runtime as well as the pause between loops here
            interval : 1, // how long to wait before executing the next loop. Due to "clamping" this is always >= 4ms,
            timeout : undefined, // forcefully break the loop after N ms
            maxLoops : undefined // forcefully break the loop after N iterations
        })
        .then(function (callback, previousResult, index, [result0, result1, ...])).then(...)
        .when(function (data) {}).when(...)
        .breakNow(); // forcefully break the loop immediately

Futures.asyncify() -- create an asynchronous function from a synchronous one
------------------

Given a syncback, returns a promisable - for all those times when you're depending on the order being unpredictable!

    // Given a syncback, returns a promisable function
    // `wait` is the number of ms before execution. If `true` it will be random between 1 - 1000ms
    // `context` is the object which should be `this`
    var async = Futures.asyncify(function (param1, param2, param3, ...) {
      return "Look how synchronous I am!";
    }, wait, context)

    // The returned function returns the `when` and `fail` methods after each call.
    async
      .when(callback)
      .fail(errback);

Futures.sleep() -- Sleep for some number of ms
---------------

Not implemented yet


Futures.wait() -- Wait for some number of ms
--------------

Not implemented yet


Futures.watchdog() -- Create a watchdog which throws if not kept alive
------------------

Not implemented yet


Futures.log() -- log messages to the console
-------------

Uses console.log if available. does nothing otherwise.

    Futures.log("Info message.");


Futures.error() -- throw an error and log message
---------------

Throws an exception and uses console.log if available.

    Futures.error("Error Message");


Related Projects
================

  * [Narwal Promises](http://github.com/kriskowal/narwhal-lib/blob/master/lib/narwhal/promise-util.js)
  * [MSDN Promise](http://blogs.msdn.com/b/rbuckton/archive/2010/01/29/promises-and-futures-in-javascript.aspx)
    * The major drawback to this library is the licensing issue.
  * [Dojo Promises](http://docs.dojocampus.org/dojo/Deferred)
  * [Strands](http://ajaxian.com/archives/javascript-strands-adding-futures-to-javascript)
  * [JavaScript API for E-based promises](http://waterken.sourceforge.net)
  * [E promises](http://www.skyhunter.com/marcs/ewalnut.html#SEC20)


Suggested Reading
=================

  * [CommonJS Promises](http://wiki.commonjs.org/wiki/Promises)
  * [Async Method Queues (Twitter Anywhere API)](http://www.dustindiaz.com/async-method-queues/)