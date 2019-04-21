---
title: "The Curious Case of Protractor and Page Synchronization"
date: 2018-08-07T18:05:06+05:30
tags: ["Protractor", "Javascript", "Async-Await"]
slug: "Protractor and Page Synchronization"
---

<style>
.caption {
    font-size: 0.9em;
    margin: 0px 50px;
    text-align: center;
    margin-bottom: 20px;
}
</style>
Protractor is an amazing tool but use it incorrectly and it will make your life miserable. This blog post is about how a simple `setTimeout()` made my life miserable.
<div>
![](/magnifying-glass.jpeg)
<div class="caption">“A book with a magnifying glass on top of it, next to a pen, and globes on a desk in Cianorte” by [João Silas](https://unsplash.com/@joaosilas?utm_source=medium&utm_medium=referral) on [Unsplash](https://unsplash.com/)</div>
</div>
I recently started working on the [fabric8-planner](https://github.com/fabric8-ui/fabric8-planner) project which is part of the [fabric8-ui](https://github.com/fabric8-ui/fabric8-ui) project. fabric8-ui is the upstream for [openshift.io](https://openshift.io/) and we use Protractor for end-to-end testing of our application.

Protractor is an end-to-end test framework for Angular and AngularJS applications. It is a [Node.js](http://nodejs.org/) program built on top of [WebDriverJS](https://github.com/SeleniumHQ/selenium/wiki/WebDriverJs). It runs tests against your application running in a real browser, interacting with it as a user would.


Protractor wraps WebDriverJS which is Javascript Selenium bindings — in other words, Protractor interacts with a browser through the Selenium WebDriver. It provides a really convenient API and has some unique Angular-specific features, like Angular specific element locating strategies (`by.model()`, `by.binding()`, `by.repeater()`), automatic synchronization between Protractor and Angular that helps to minimize the use of explicit waits here and there.

## The Background —
Before we get to the problem, we need to understand how protractor deals with asynchronous nature of Javascript and provides a synchronous API.


Despite being asynchronous, protractor allows us to write synchronous tests. This is possible because of the WebDriverJS library which uses a **[promise manager](http://seleniumhq.github.io/selenium/docs/api/javascript/module/selenium-webdriver/lib/promise.html)** to ease the pain of working with a purely asynchronous API. WebDriverJS maintains a queue of pending promises, called the **control flow**, to keep execution organized. For example, consider this test:

```javascript
it('should find an element by text input model', function() {
    browser.get('app/index.html#/form');

    var username = element(by.model('username'));
    username.clear();
    username.sendKeys('Jane Doe');

    var name = element(by.binding('username'));

    expect(name.getText()).toEqual('Jane Doe');

    // Point A
  });
```
At Point A, none of the tasks have executed yet. The `browser.get` call is at the front of the control flow queue, and the `name.getText()` call is at the back. The value of `name.getText()` at point A is an unresolved promise object.

Protractor provides two ways to handle asynchronous actions

1. Promise Manager/Control Flow
2. Async/Await

## The Promise Manager/Control Flow Monster
![](/cute-monster.jpeg)

Before performing any action, Protractor waits until there are no pending asynchronous tasks in your Angular application. **This means that all timeouts and HTTP requests are finished**. For Angular apps, Protractor will wait until the [Angular Zone](https://medium.com/@MertzAlertz/what-the-hell-is-zone-js-and-why-is-it-in-my-angular-2-6ff28bcf943e) stabilizes. This means long-running asynchronous operations will block your test from continuing.

>So, if you have an infinite timeout (usually used to refresh tokens), your tests will wait indefinitely.

In fabric8-ui, we have a piece of code that refreshes the JWT token

```javascript
setupRefreshTimer(refreshInSeconds: number) {
    ....
    this.clearTimeoutId = setTimeout(() => 
        this.refreshToken(),  refreshInMs
    );
    ....
}
```

The above code worked as expected by triggering a new timer just when the existing token was about to expire. This meant the control flow queue would always have a setTimeout() call within it. As soon as the current timeout was about to expire, a new one was added to the queue. This meant that the Angular Zone would never stabilize and our tests would keep waiting indefinitely. Our tests would wait indefinitely and always fail with the following error

```
Failed: Timed out waiting for Protractor to synchronize with the page after 11 seconds. Please see https://github.com/angular/protractor/blob/master/docs/faq.md.
Error: Timeout - Async callback was not invoked within timeout specified by jasmine.DEFAULT_TIMEOUT_INTERVAL 
```

The easiest way to fix the above error was to run the setTimeout function outside the angular zone with the following code

```javascript
this.ngZone.runOutsideAngular(() => {
  setTimeout(() => {
    // Changes here will not propagate into your view.
    this.ngZone.run(() => {
      // Run inside the ngZone to trigger change detection.
    });
  }, REALLY_LONG_DELAY);
});
```
But we decided to fix the underlying issue for once and for all by using Async/Await instead of Control Flow.

## Slaying the Monster with Async/Await
The Async/Await way allows us to choose to when to wait for an action. Instead of waiting for angular to stabilize on every action, we can selectively wait for angular to stabilize on selected actions. Let’s look at an example

```javascript
describe('angularjs homepage', function() {
  it('should greet the named user', async function() {
    await browser.get('http://www.angularjs.org');

    await element(by.model('yourName')).sendKeys('Julie');

    var greeting = element(by.binding('yourName'));

    expect(await greeting.getText()).toEqual('Hello Julie!');
  });
```

In the above example, protractor will wait only at lines which are awaiting for the promise to resolve. We can add “await” keyword to each operation that we want our program to wait for. Before you start using the Async/Await pattern with Protractor, you’ll have to disable the Control Flow (at least for now). [Control Flow will soon be deprecated](https://github.com/SeleniumHQ/selenium/issues/2969) and Async/Await will be the default.

>Protractor won’t wait for HTTP requests/async operations unless it is explicitly specified via await.

Control flow uses a queue for actions to wait upon while async/await waits only at the specified lines. We ended up fixing the Waiting for Protractor to synchronize issue by migrating to async/await from control flow. You can find our typescript based tests [here](https://github.com/fabric8-ui/fabric8-planner/tree/master/src/tests).

## Takeaways —
 
1. Use Async/Await instead of Control Flow, unless you have a concrete reason to not use Async/Await.
2. Wrap your timeouts and long duration async operations in ngZone.runOutsideAngular() if you plan to use Control Flow.
3. The Control Flow will soon be deprecated and async/await will be the default (See [this Github issue](https://github.com/SeleniumHQ/selenium/issues/2969)).

---
### Further Reading —


- https://christianliebel.com/2016/11/angular-2-protractor-timeout-heres-fix/

- https://github.com/angular/protractor/blob/master/docs/async-await.md

- https://stackoverflow.com/questions/44691940/explain-about-async-await-in-protractor