## Web worker compatibility problems

Lucas Garron  
2020-12-28

This document summarizes my main issues with web worker compatibility, from
almost 8 years of working with web workers and from building the
[`scrambles`](https://github.com/cubing/jsss/tree/scrambles/src/scrambles/worker)
library in particular.

Note that I use [`esbuild`](https://github.com/evanw/esbuild) and [Parcel
2](https://v2.parceljs.org/) for developing JS libraries and web apps myself, so
I mention them more often in my examples below. I know that other bundlers can
solve some of these problems, but I'm not here to discuss bundler tradeoffs.

I'd more like to give an impression of the specific ways that web worker
compatibility is currently a mess. To me, it feels a bit like the way that JS
module systems themselves were a mess (until recently), although I don't see as
much ecosystem motivation for fix these web worker issues.

# Motivating example

Suppose you have a task that takes about 1 second of calculations, and you're
writing a library for it:

```js
// foo.js

export function calculateFoo(input) {
  // about 1 second of work
}
```

```js
// client.js

import { calculateFoo } from "foo.js";
console.log(calculateFoo(1));
```

In my case, "about 1 second of calculation" is necessary to generate fair
[puzzle scrambles](https://www.youtube.com/watch?v=fMDxNgXzLwM) for speedcubing.
In particular, it can take about a second to generate a fair scramble for
[4x4x4](https://en.wikipedia.org/wiki/Rubik%27s_Revenge). But this document
applies to any situation with a non-trivial amount of work, which might include
search problems or crypto operations.

## Promise

When a function will take some time, good pattern is to encourage using a `Promise`s / async pattern:

```js
// foo.js

export async function calculateFoo(input) {
  // about 1 second of work
}
```

```js
// client.js

import { calculateFoo } from "foo.js";
calculateFoo(1).then(console.log);
```

However, this is only really useful if `calculateFoo()` involves waiting (e.g.
for a server to respond). It's not useful if `calculateFoo()` has to do a significant number of
calculations, because JavaScript is single-threaded. Unless the implementation
goes out of its way to yield during its execution, all JS on the page will be
blocked, and the page will appear unresponsive. Even if the implementation
yields multiple times per second, the user experience is still not likely to be
good.

# Web worker

The standard solution is to use a web worker. If you're writing a library that
is best run in a web worker, the most responsible thing would be to
automatically instantiate the worker inside your code. Here's where the pain begins.

In an ideal world, I think this should be similar to importing a module. Here's a simple
"dream syntax" example for how that could look:

```js
// foo.js

import { doHeavyWork } from "./worker.js" as worker; // `as worker` is dream syntax for this

export async function calculateFoo(input) {
  //...
  const heavyResult = await doHeavyWork();
  //...
}
```

```js
// worker.js

export async function doHeavyWork(input) {
  // about 1 second of work
}
```

This dream syntax has some edge-case issues, but conceptually it works for
a lot of use cases.

However, reality is very far from this. Right now, using a web worker requires
rewriting the library to use `postMessage`. Here's a hacky example:

```js
// foo.js

var worker = new Worker("./worker.js");
let pendingResolve;
worker.addEventListener("message", function (e) {
  pendingResolve(e.data);
});

function calculateFoo(input) {
  return new Promise((resolve, reject) => {
    pendingResolve = resolve;
    worker.postMessage(4);
  });
}
```

```js
// worker.js

function doHeavyWork(input) {
  // about 1 second of work
}

self.addEventListener("message", function (e) {
  self.postMessage(doHeavyWork(e.data));
});
```

```js
// client.js

import { calculateFoo } from "./foo.js";
console.log(calculateFoo(1));
```

# Problems

## **Problem 1**: Firefox and Safari don't support module syntax in web workers.

There is standard syntax for instantiating a web worker as an ES6 module:

```js
new Worker("./worker.js", { type: "module" });
```

This allows using various useful features, such as sharing code with other parts
of the codebase.

Chrome implemented this [in 2019](https://web.dev/module-workers/). As of mid-2021, Safari Technology Preview has support, so this will probably come to WebKit on macOS and iOS soon. Unfortunately, [Firefox Bug
1247687](https://bugzilla.mozilla.org/show_bug.cgi?id=1247687) is not showing a lot of activity. I'm not aware of a simple way to test
for module worker support before trying to use it — the closest option is to try/catch the worker instantiation and live with an error in the JS console when it fails.

If a site wants to use web workers across all modern browsers, it has to use a
"classic" web worker. You can still "import" code instead of loading it all at
once, but you have to use the completely different
[`importScripts()`](https://developer.mozilla.org/en-US/docs/Web/API/WorkerGlobalScope/importScripts)
which essentially requires communicating with global variables. I think some
bundlers have basic support for this, but sharing code across workers and
non-workers is definitely not simple. The simplest approach is to keep the
worker code separate from non-worker code, but this still causes extra
development overhead, is harder to debug, and might cause clients to load a
significant amount of "duplicated" code.

## **Problem 2**: Keeping track of an extra entry point

The path for a web worker is referenced using a string inside source code,
without any syntax/semantics to indicate that a string specifies a source file.

```js
import { a, b, c } from "./foo.js"; // `foo.js` clearly refers to a file.

const worker = new Worker("./worker.js"); // "./worker.js" is no different from any other string.
```

This breaks many bundlers/tools unless you install extra plugins and work very
carefully. For example, many bundlers will lose track of `./worker.js` if it is
not defined inline:

```js
// Some (most?) bundlers will no longer understand that the `worker.js` file is referenced here.
const workerPath = "./worker.js";
var worker = new Worker(workerPath);
```

Note that it's possible to instantiate a worker from a string (via data URL or
object URL)), but that requires
converting the target worker code to a string and embedding it in the output
code. Another option might be to instantiate the worker from a string that calls
`importScripts("./worker.js")`, but this is even more brittle.

This makes it tricky to implement workarounds for the issues below, as well.

## **Problem 3**: Broken worker code will not return an error

Suppose that `worker.js` contains code, but that code has an issue that breaks
its ability to respond to any messages. The worker will be instantiated
successfully, but the instantiator will have no way to distinguish a broken
worker from one that is taking a long time to respond. It's possible to add a
liveness check or timeouts, but this complicates the calling code and requires
selecting a timeout that doesn't cause false negatives. (Selecting a flat
timeout can be an issue for calculations that fluctuate in running time.)

## **Problem 4**: Messages have to be serialized

Everyone writing web worker code has to select their own way to serialize command
and arguments over the wire. The
[Transferable](https://developer.mozilla.org/en-US/docs/Web/API/Transferable)
interface can make this easier, but it doesn't work for everything.

## **Problem 5** It's not simple to wrap `MessagePort` in `Promise` semantics.

The code example above with `pendingResolve` has various issues:

- If `calculateFoo()` is called twice in a row, the first call will never be
  resolved.
- If the worker encounters an error, the `Promise` is never rejected.
- It assumes that only one function is "exported" from the worker.

It's possible to handle all of these by adding additional a custom protocol on
top of the
message port, but this requires non-trivial code and there is no web standard
for this.

It's also possible to embrace message port semantics instead of trying to wrap
existing APIs. However, it is significantly easier to use modules as the
conceptual basis for an API instead of requiring every developer to learn
something new.

## **Problem 6**: Static analysis and type safety are significantly more difficult.

The previous two problems make it much more difficult to write code that can be
statically verified to do the right thing, by either a computer or a human.

## **Problem 7**: Web workers cannot be instantiated cross-origin.

While `import` runs code in the same origin as its caller, a web worker runs
with the origin of its URL. Instantiating a worker whose URL has a different
origin from the current page (even if the worker in is instantiated from a
library script on the same origin as the worker script) is blocked by most
browsers.

CORS does not offer a workaround. As far as I know, there is no way to say
"allow another site to instantiate this file as a web worker", either in the
caller's origin, or the worker file's origin.

This breaks:

- flat-file JS development
- library code on CDNs.

Flat-file JS development is a lost art. I once [put a lot of effort into using
web workers in flat-file
development](https://github.com/lgarron/MagicWorker.js), but I had to give that
up when I started using TypeScript and modules. It's probably a moot point for
most devs, but I still feel disappointed. (Running a local development server is
not simple for everyone. Unfortunately, it seems that most projects have said "if we have to
run a dev server, we might as well use a bundler" with tens of thousands of transitive
dependency files. I think we should push the ecosystem away from this in the
long term.)

As for library code, this means that our example fails if `worker.js` and
`client.js` are not on the same site. It's possible to work around this by
instantiating an ugly but fairly short "trampoline":

```js
// https://cdn.cubing.net/foo.js

const workerURL = "https://cdn.cubing.net/worker.js";
const importSrc = `import "${workerURL}";`;
const blob = new Blob([importSrc], {
  type: "text/javascript",
});
new Worker(URL.createObjectURL(blob), { type: "module" });
// ...
```

Note that this requires the site to allow `blob:` as a [`worker-src`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/worker-src) if they are
using CSP.

Also, dynamically generating code is not generally a good idea. If `workerURL`
is not calculated safely, an attacker could easily inject
arbitrary code into the trampoline.

## **Problem 8**: Calculating relative paths requires `import.meta.url`

Notice that the full worker URL is hardcoded inside `foo.js` in the previous example. This
is because the instantiated worker cannot itself compute a URL relative to its
instantiator. It would be more flexible to use a relative path.

The "correct" way to compute a relative script path is as follows:

```js
const url = new URL("./worker.js", import.meta.url);
```

Unfortunately, `import.meta.url` causes a
syntax error **for the entire file** even if `import.meta.url` is only used in
some code branches. There is no way to transpile this code to ES2017 or earlier.
This makes it difficult to publish a library with a CommonJS/UMD build.

The only workaround I know is to place the URL calculation in a separate file
that is conditionally imported, and hope that your bundlers don't inline it. But
this doesn't solve the issue of instantiating the worker from the non-module
version of the code.

Also note that the relative newness of `import.meta.url` can cause indirect
compatibility issues. For example, [`esbuild`](https://github.com/evanw/esbuild)
doesn't allow `import.meta.url` when targeting `es2017` (the most compatible
"modern" version of JS). You have to use `es2020`, but this will no longer
transpile other syntax (`??=`) that will cause a syntax error in some browsers.
There are ways around this with most bundlers, but it's possible that a
developer would not notice this if they only develop in a single browser.

## **Problem 9**: `node` doesn't support relative paths

Suppose you want your library to work in `node` (as well as the browsers) and publish the following two files in the same folder:

```js
const worker = new Worker("./worker.js"));
```

While this doesn't work for the cross-origin example I mentioned, this will work
for same-origin code in browsers directly, and most bundlers have a way to
handle it.

However, `node`
does not support relative file imports like this. If you want to support `node`, you
have to calculate the path:

```js
import { Worker } from "worker_threads";
import { join } from "path";

const worker = new Worker(join(__dirname, "worker.js"));
```

If you run this through a transpiler, some bundlers will no longer recognize
`worker.js` as a relative file path. Some will recognize it if you use
`__dirname + "/worker.js"` instead, but this may break Windows compatibility.

In addition, if you want to publish a version of your code that works in both
`node` and browsers, then you have to be careful that the `node`-specific
imports are not used in the browser, e.g. like this:

```js
// foo.js

async function newWorker() {
  if (typeof Worker === "undefined") {
    return new Worker("./worker.js")
  } else {
    return (await import("./foo-node.js")).newNodeWorker(__dirname, "./worker.js"):
  }
}
```

```js
// foo-node.js
import { Worker } from "worker_threads";
import { resolve } from "path";

export function newNodeWorker(dir, file) {
  const worker = new Worker(resolve(dir, file));
}
```

You'll also have to make sure your bundler doesn't inline `foo-node.js` into
`foo.js`.

Depending on your bundler, or any assumptions you can make about the
bundlers/environments used by the code that calls your library, you may be able
to simplify some of this. However, this kind of code is often handled
differently based on bundler heuristics and/or plugins. Some bundlers (e.g. `snowpack`) will emit errors even if the `node` imports are in code paths unused by any browser.

It's also possible to build different versions of your code for browsers and
`node`, but the relevant `package.json` for this
(`main`/`module`/`browser`/`exports`) have significant compatibility issues.
Also, many libraries already publish CJS/ESM and/or raw/minified/bundled-dependency
versions of their code, so this can contribute to an explosion in the number of
builds for a library, which can in turn bloat the package size.

## **Problem 10**: `node` workers differ from browsers

`node` workers actually do not have the same semantics as browser workers.
Depending on the `MessagePort` features you use, you may need extra code to
treat browser workers and node workers as similar. For example, see
[`node-adapter.ts`](https://github.com/GoogleChromeLabs/comlink/blob/master/src/node-adapter.ts)
in the `comlink` library.

(Note that `comlink` currently does not publish module `exports` or sub-path
`package.json` to automatically specify which version of the adapter to use for
CJS/ESM. This means that you also need your code/build system/bundler to switch between two of the
build files manually.)

## **Problem 11**: TypeScript development often requires an extra wrapper

Suppose you maintain a library using TypeScript source code. The most natural
would be to write the worker source in TypeScript as well:

```js
// foo.ts

const worker = new Worker("./foo.ts");
```

```js
// worker.ts

// ...
```

As described above, there are various issues with worker source file paths.
However, TypeScript adds an additional level of complexity. I've found that in
practice I usually need to use a `.js` file as the worker entry point in order
to maximize compatibility with various tools.

This is a fairly minor issue, but unless you're diligent about using `.js` file as
a pure stub wrapper for typed files, it can mask type issues in your code. As
described above, this doesn't always show up as an error, and can be annoying to
debug.

## **Problem 12**: Many bundlers break with workers

Bundlers often require plugins to work with workers, which can be
under-maintaned and undersupported. Even bundlers that claim built-in support
have various issues, and custom file name heuristics can make it a hassle to try
out different bundlers.

I personally use Parcel 2, which is fairly featureful, but still fails if [any
code is shared between the main thread and a
worker](https://github.com/parcel-bundler/parcel/issues/5504) or [any code in
the worker source code uses
`import()`](https://github.com/parcel-bundler/parcel/issues/5503). Both of these
bugs are directly triggered by workarounds for some of the problems I've
desribed above.

I don't want to blame any bundler maintainers, because I know they have a hard
job. But I have yet to find a bundler that makes workers significantly less of a
pain.

# Libraries

I'm aware of three libraries that attempt to solve solve some of the problems I
describe below:

- [`comlink`](https://github.com/GoogleChromeLabs/comlink) is the first library
  I learned of. I have to work around some issues with it, but I am still using
  it to address problems 3, 4, and 9.
- [`web-worker`](https://github.com/developit/web-worker). This covers some
  `node` compatibility issues, but still requires using `postMessage`.
- [`threads.js`](https://github.com/andywer/threads.js) looks like a promising
  alternative to `comlink`. However, I can't use it because [it's completely
  incompatible with Parcel 2](https://github.com/andywer/threads.js/issues/319).

# What next?

I would really love to see something that gets us closer to the dream syntax I
mentioned above:

```js
import { a, b, c } from "./worker.js" as worker;
```

Dynamically:

```js
const worker = import("./worker.js", { type: "module", worker: "dedicated" });
```

This would require standardization around things like:

- Making all the imported functions return `Promise`s, either by requiring the
  module implementation to return `Promise`s, or by wrapping in a `Promise` on
  the calling side.
- How to pass data and errors.

The libraries I mentioned do a fairly good job of papering over all this
already. If it weren't for file path issues and syntax compatibility, it would
almost be possible to polyfill such a worker. But in practice, this would
probably need to be standardized as part of ECMAScript and web APIs.

For now, it would be nice if a library author could instantiate a worker a
single way in their source code, and have bundlers convert that into code that
is compatible with CJS/ESM/`node`/browser environments _without_ any special
work on behalf of the author.

# State of web worker compatibility in Jan. 2023

Out-of-the-box support for the following syntax is implemented in all modern browser engines (Chrome stable, Safari Technology Preview, Firefox Nightly) as of this month! 😻🥳
```js
// main.js
const worker = new Worker(import.meta.resolve("./worker.js"), {type: "module"})
worker.addEventListener("message", console.log)

// worker.js
self.postMessage("hello!")
```

However:
- `import.meta.resolve(...)`
  - `import.meta.resolve(...)` is implemented in Safari Technology Preview and enabled by default, but it is not possible to enable in the stable version of Safari 16.2. However, it is possible to polyfill using: `import.meta.resolve = (s) => new URL(s, import.meta.url).href;`
  - [Some bundlers will break code](https://github.com/evanw/esbuild/issues/2866) with this syntax. 
  - `new URL(..., import.meta.url).href` can be used for older browsers and most JS libraries/apps should probably ship this syntax instead of `import.meta.resolve(...)` for now, but [some bundlers will also break code](https://github.com/evanw/esbuild/issues/312) with this syntax.
- Firefox
  - `{type: "module"}` [only just landed](https://bugzilla.mozilla.org/show_bug.cgi?id=1247687) in Firefox Nightly (after being landing one before and being backed out).
  - Firefox still needs [support for dynamic imports in module workers](https://bugzilla.mozilla.org/show_bug.cgi?id=1540913).
- `node`
  - `node` has a worker API, but it's sufficiently incompatible with web workers that you can't just write browser code and expect it to work in `node`. The `node` authors [seem open to a community contribution to add web-compatible workers](https://github.com/nodejs/node/issues/43583).
  - `node` 19.4 only [just added `import.meta.resolve`](https://nodejs.org/api/esm.html#importmetaresolvespecifier-parent) behind an experimental flag, but it returns a `Promise`, which is perfectly poised to introduce subtle bugs to the ecosystem if it ships unflagged. They have [begun work to support a synchronous version](https://github.com/whatwg/html/pull/5572#issuecomment-1041876685).
- Cross-origin workers
  - We still need [`BlankWorker`](https://github.com/whatwg/html/issues/6911) or [module blocks](https://github.com/tc39/proposal-module-expressions) to avoid a [gnarly trampoline](https://github.com/lgarron/web-worker-compat-problems#problem-7-web-workers-cannot-be-instantiated-cross-origin) for libraries running on a CDN origin.
