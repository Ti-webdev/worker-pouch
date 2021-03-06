WorkerPouch [![Build Status](https://travis-ci.org/nolanlawson/worker-pouch.svg)](https://travis-ci.org/nolanlawson/worker-pouch)
=====

```js
// This pouch is powered by web workers!
var db = new PouchDB('mydb', {adapter: 'worker'});
```

Plugin to use PouchDB over [web workers](https://developer.mozilla.org/en-US/docs/Web/API/Worker). Transparently proxies all PouchDB API requests to a web worker, so that the most expensive database operations are run in a separate thread. Supports Firefox and Chrome.

Basically, WorkerPouch allows you use the PouchDB API like you normally would, but your UI will suffer fewer hiccups, because any blocking operations (such as IndexedDB or checksumming) are run inside of the worker. You don't even need to set up the worker yourself, because the script is loaded in a [Blob URL](https://developer.mozilla.org/en-US/docs/Web/API/Blob).

WorkerPouch passes [the full PouchDB test suite](https://travis-ci.org/nolanlawson/worker-pouch). It requires PouchDB 5.0.0+.

IE, Edge, Safari, and iOS are not supported due to browser bugs. Luckily, Firefox and Chrome are the browsers that [benefit the most from web workers](http://nolanlawson.com/2015/09/29/indexeddb-websql-localstorage-what-blocks-the-dom/). There is also an API to [detect browser support](#detecting-browser-support).

Usage
---

    $ npm install worker-pouch
    
When you `npm install worker-pouch`, the client JS file is available at `node_modules/worker-pouch/dist/pouchdb.worker-pouch.js`. Or you can just download it from Github above (in which case, it will be available as `window.workerPouch`).

Then include it in your HTML, after PouchDB:

```html
<script src="pouchdb.js"></script>
<script src="pouchdb.worker-pouch.js"></script>
```

Then you can create a worker-powered PouchDB using:

```js
var db = new PouchDB('mydb', {adapter: 'worker'});
```

##### In Browserify

The same rules apply, but you have to notify PouchDB of the new adapter:

```js
var PouchDB = require('pouchdb');
PouchDB.adapter('worker', require('worker-pouch'));
```

Performance benefits
----

These numbers were recorded using [this site](http://nolanlawson.github.io/database-comparison-worker-pouch/). The test involved inserting 10000 PouchDB documents, and was run on a 2013 MacBook Air. Browser data was deleted between each test.

| | Time (ms) | Blocked the DOM? |
| ----- | ------ | ------ |
| *Chrome 48* | | |
| &nbsp;&nbsp;&nbsp;`put()` - normal | 50070 | No |
| &nbsp;&nbsp;&nbsp;`put()` - worker | 56993 | No |
| &nbsp;&nbsp;&nbsp;`bulkDocs()` - normal | 2740 | Yes |
| &nbsp;&nbsp;&nbsp;`bulkDocs()` - worker| 3454 | No |
| *Firefox 43* |  | |
| &nbsp;&nbsp;&nbsp;`put()` - normal | 39595 | No |
| &nbsp;&nbsp;&nbsp;`put()` - worker | 41425 | No |
| &nbsp;&nbsp;&nbsp;`bulkDocs()` - normal | 1027 | Yes |
| &nbsp;&nbsp;&nbsp;`bulkDocs()` - worker| 1130 | No |

Basic takeaway: `put()`s avoid DOM-blocking (due to using many smaller transactions), but are much slower than `bulkDocs()`. With WorkerPouch, though, you can get nearly all the speed benefit of `bulkDocs()` without blocking the DOM.

(Note that by "blocked the DOM," I mean froze the animated GIF for a significant amount of time - at least a half-second. A single dropped frame was not penalized. Try the test yourself, and you'll see the difference is pretty stark.)

Detecting browser support
----

WorkerPouch doesn't support all browsers. So it provides a special API to dianogose whether or not the current browser supports WorkerPouch. Here's how you can use it:

```js
var workerPouch = require('worker-pouch');

workerPouch.isSupportedBrowser().then(function (supported) {
  var db;
  if (supported) {
    db = new PouchDB('mydb', {adapter: 'worker'});
  } else { // fall back to a normal PouchDB
	db = new PouchDB('mydb');
  }  
}).catch(console.log.bind(console)); // shouldn't throw an error
```

The `isSupportedBrowser()` API returns a Promise for a boolean, which will be `true` if the browser is supported and `false` otherwise.

If you are using this method to return the PouchDB object *itself* from a Promise, be sure to wrap it in an object, to avoid "circular promise" errors:

```js
var workerPouch = require('worker-pouch');

workerPouch.isSupportedBrowser().then(function (supported) {
  if (supported) {
    return {db: new PouchDB('mydb', {adapter: 'worker'})};
  } else { // fall back to a normal PouchDB
	return {db: new PouchDB('mydb')};
  }
}).then(function (dbWrapper) {
  var db = dbWrapper.db; // now I have a PouchDB
}).catch(console.log.bind(console)); // shouldn't throw an error
```

Debugging
-----

WorkerPouch uses [debug](https://github.com/visionmedia/debug) for logging. So in the browser, you can enable debugging by using PouchDB's logger:

```js
PouchDB.debug.enable('pouchdb:worker:*');
```

Q & A
---

#### Wait, doesn't PouchDB already work in a web worker?

Yes, you can use pure PouchDB inside of a web worker. But the point of this plugin is to let you use PouchDB from *outside a web worker*, and then have it transparently proxy to another PouchDB that is isolated in a web worker.

#### What browsers are supported?

Only those browsers that 1) allow blob URLs for web worker scripts, and 2) allow IndexedDB inside of a web worker. Today, that means Chrome and Firefox.

#### Can I use it with other plugins?

Not right now, although map/reduce is supported.

#### Don't I pay a heavy cost of structured cloning due to worker messages?

Yes, but apparently this cost is less than that of IndexedDB, because the DOM is significanty less blocked when using WorkerPouch. Another thing to keep in mind is that PouchDB's internal document representation in IndexedDB is more complex than the PouchDB documents you insert. So you clone a small PouchDB object to send it to the worker, and then inside the worker it's exploded into a more complex IndexedDB object. IndexedDB itself has to clone as well, but the more complex cloning is done inside the worker.

Building
----

    npm install
    npm run build

Your plugin is now located at `dist/pouchdb.worker-pouch.js` and `dist/pouchdb.worker-pouch.min.js` and is ready for distribution.

Testing
----

### In the browser

Run `npm run dev` and then point your favorite browser to [http://127.0.0.1:8000/test/index.html](http://127.0.0.1:8000/test/index.html).

The query param `?grep=mysearch` will search for tests matching `mysearch`.

### Automated browser tests

You can run e.g.

    CLIENT=selenium:firefox npm test
    CLIENT=selenium:phantomjs npm test

This will run the tests automatically and the process will exit with a 0 or a 1 when it's done. Firefox uses IndexedDB, and PhantomJS uses WebSQL.
