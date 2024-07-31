# ecma262#3195

This is the repo for the nomative change of https://github.com/tc39/ecma262/pull/3195.

Status: Stage 2

Champions: Chengzhong Wu ([@legendecas](https://github.com/legendecas))

## Motivation

The goal is avoid revealing internal slot [[PromiseState]] with
`promise.then(onFulfill, onRejection)` to the promise handlers by
host hook requirement.

ECMA262 [HostEnqueuePromiseJob](https://tc39.es/ecma262/#sec-hostenqueuepromisejob)
defines that a host must implement it as:

- Let scriptOrModule be [GetActiveScriptOrModule](https://tc39.es/ecma262/#sec-getactivescriptormodule)()
  at the time HostEnqueuePromiseJob is invoked. If realm is not null, each time job is invoked the
  implementation must perform [implementation-defined](https://tc39.es/ecma262/#implementation-defined)
  steps such that scriptOrModule is the [active script or module](https://tc39.es/ecma262/#job-activescriptormodule)
  at the time of job's invocation.

The timing of `HostEnqueuePromiseJob` depends on internal Promise
`[[PromiseState]]`. The motivation instead expect the `ActiveScriptOrModule`
been determined AT the time `promise.then()` is invoked.

```js
// module A
const {promise, resolve} = Promise.withResolvers();
promise.then(handler); // (1)

// module B
resolve(); // HostEnqueuPromiseJob for (1)
promise.then(handler); // (2) HostEnqueuPromiseJob
```

That is, in the above example, the ActiveScriptOrModule for the handler at call site (1)
should be "module A", and at call site (2) should be "module B".

## HTML integration

HTML [HostEnqueuePromiseJob](https://html.spec.whatwg.org/multipage/webappapis.html#hostenqueuepromisejob)
doesn't follow the requirements of ecma262.

However, HTML spec defines that the active ScriptOrModule is saved at the time of
[HostMakeJobCallback](https://html.spec.whatwg.org/multipage/webappapis.html#hostmakejobcallback)
is invoked rather than at the time of HostEnqueuePromiseJob is invoked. The two
host hooks are invoked with the same active script or module consecutively.

This change also mandates that the active scripts are propagated through
promise jobs and FinalizationRegistry cleanup callbacks.

For example, the following example uses HTML APIs that depends on the original
script's incumbent document URL.

```js
const frame = frames[0];
const setLocationHref = Object
  .getOwnPropertyDescriptor(frame.location, "href")
  .set
  .bind(frame.location);

framePromise.resolve("./page-1").then(setLocationHref);
```

This proposed behavior exists in the HTML spec since HTML
`HostMakeJobCallback` already propagates the active script.

## Proposed change

Checkout https://github.com/tc39/ecma262/pull/3195

The reality of the implementation status is:

 . | `resolved.then(f)` | `resolve.then(v => f(v))` | `pending.then(f)` | `pending.then(v => f(v))`
--- | --- | --- | --- | ---
HTML | outer | outer | outer | outer
ECMA-262 | outer | outer | ☹️ inner | ☹️ inner
Chrome/Firefox | ☹️ inner | outer | ☹️ inner | outer
Safari | outer | outer | ☹️ inner | ☹️ inner

> - pending is resolve in inner iframe.
> - f is iframe `setLocationHref`.
> - outer: url is resolved to outer page.
> - inner: url is resolved to inner iframe.

Conclusion:
- ECMA-262 and Safari reveal the `[[PromiseState]]`.
- Chrome/Firefox don't respect `.then(f)` equals to `.then(v => f(v))`
- HTML and the proposed behavior have neither of those problems

The proposed change also fixes the inconsistent requirement on ecma262
`HostEnqueuePromiseJob` and HTML `HostMakeJobCallback`.
