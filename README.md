[![Build Status](https://travis-ci.org/mojo-js/mesh.js.svg)](https://travis-ci.org/mojo-js/mesh.js) [![Coverage Status](https://coveralls.io/repos/mojo-js/mesh.js/badge.svg?branch=master)](https://coveralls.io/r/mojo-js/mesh.js?branch=master) [![Dependency Status](https://david-dm.org/mojo-js/mesh.js.svg)](https://david-dm.org/mojo-js/mesh.js) [![Join the chat at https://gitter.im/mojo-js/mesh.js](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/mojo-js/mesh.js?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)


Mesh is a universal interface for communicating with data sources whether it's your API, mongobus, pubnub, webrtc, socket.io, redis, or local storage. Easily build sophisticated features such as offline-mode, realtime data, rollbacks, and more with little effort.

Mesh is entirely customizable, and doesn't make assumptions about how a data source works. You can easily build your own API adapter that's interoperable with all the other mesh plugins.

Here's a basic example of how you might implement an API that caches temporarily to local storage:

```javascript
var mesh = require("mesh");
var http = require("mesh-http");
var localStorage = require("mesh-local-storage");

// local storage cache - keep stuff for one minute max
var cache = localStorage({
	ttl: 1000 * 60
});
var api = http({
	prefix: "/api"
});

// pipe all persistence operations to the cache
api(mesh.op("tail")).pipe(mesh.open(cache));

// the DB we'll use, return the first result returned, and
// only pass 'load' operations to the cache
var bus = mesh.fallback(mesh.accept("load", cache), api);


bus(mesh.op("insert", {
	collection: "people"

	// path is automatically resolved from the collection param,
	// but you can easily override it.
	path: "/people",

	// POST is resolved from the operation name, but it's
	// also overridable
	method: "POST",

	data: {
		name: "john"
	}
})).on("data", function(personData) {

	// load the person saved. This should result in a cache
	// hit for local storage. Also note that the HTTP path & method
	// will automatically get resolved.
	bus(mesh.op("load", {
		collection: "people",
		query: {
			name: person.name
		}
	})).
	on("data", function(personData) {
		// do stuff with data
	});
});
```

#### Highlights

- Streamable interface.
- Works with any library, or framework.
- Works on any platform.
- Tiny (11kb).
- Works nicely with other stream-based libraries such as [highland](http://highlandjs.org/).
- Isomorphic. Easily use different data sources for different platforms.
- Easily testable. Stub out any data source for a fake one.
- Simple design. Use it for many other things such as an event bus, message-queue service, etc.

#### Roadmap

- Examples
	- [MeteorJS](http://meteor.com/) clone with ReactJS
		- latency compensation
		- same views & models on the front-end & backend
		- offline-mode (fully operational with CRUD)
		- realtime data
		- mongobus queries server-side/client-side with socket.io
- Adapters
	- RPC adapter
	- ReactJS
	- AngularJS Adapter
	- EmberJS

#### Installation

```
npm install mesh
```

Or via bower:

```
bower install mesh
```

#### Adapters

- [pubnub](http://github.com/mojo-js/mesh-pubnub) - [pubnub](http://www.pubnub.com/) sync adapter
- [socket.io](http://github.com/mojo-js/mesh-socket.io) - [socket.io](http://socket.io/) sync adapter
- [webrtc](http://github.com/mojo-js/mesh-webrtc) - [webrtc](http://www.webrtc.org/) sync adapter
- [loki](http://github.com/mojo-js/mesh-loki) - [loki](http://lokijs.org/#/) in-memory data source
- [memory](http://github.com/mojo-js/mesh-memory) - another in-memory data source
- [local-storage](http://github.com/mojo-js/mesh-local-storage) - local storage data source
- [http](http://github.com/mojo-js/mesh-http) - HTTP adapter
- [mongobus](http://github.com/mojo-js/mesh-mongobus) - Mongobus Adapter (server-side)

#### Examples

- [collaborative todos](http://crudlet.herokuapp.com/#/live-todos)
- [api + models](https://github.com/mojo-js/mesh-http#http--caplet-example)
- [web worker DB](http://crudlet.herokuapp.com/#/live-web-workers)
- [collaborative drawing](http://crudlet.herokuapp.com/#/live-scribble)


#### Realtime Example

Below is an example of a realtime DB that uses [pubnub](https://github.com/mojo-js/mesh-pubnub), and [local storage](https://github.com/mojo-js/mesh-local-storage).

```javascript
var mesh = require("mesh");
var pubnub = require("mesh-pubnub");
var localStorage = require("mesh-local-storage");

// store data locally on the users machine
var localBus = localStorage();

// pubnub adapter for sending operations to other connected clients
var pubBus = pubnub({
	publishKey: "publish key",
	subscribeKey: "subscribe key",
	channel: "chatroom"
});

// the actual DB we're going to use. Pass
// all operations to localstorage, and pubnub
var bus = mesh.parallel(localBus, pubBus);

// tail all operations from pubnub to the local DB. Note
// that remote operations don't get re-sent to pubnub.
pubbus(mesh.op("tail")).pipe(mesh.open(bus));

// create a child data source - collection will get passed to each operation
var peopleBus = mesh.child(bus, {
	collection: "people"
});

// insert some people
peopleBus(mesh.op("insert", {
	data: [
    {	name: "Gordon Ramsay" },
    {	name: "Ben Stiller" }
  ]
})).on("data", function() {
	// handle data here
});
```

#### [stream.Readable](https://nodejs.org/api/stream.html#stream_class_stream_readable) bus(operation)

Runs a new operation.

> Note that the supported operations & required options may change depending on the data store you're using.

```javascript
var localStorage = require("mesh-local-storage");

var localBus = localStorage();
localBus(mesh.operation("insert", {
	collection: "people",
	data: { name: "Arnold Schwarzenegger" }
})).on("data", function() {
	// handle data here
});
```

#### [stream.Stream](https://nodejs.org/api/stream.html#stream_class_stream_readable) mesh.open(bus)

Creates a new operation stream.

```javascript
var operationStream = mesh.open(bus);

// emitted when the operation is performed
operationStream.on("data", function() {

});

operationStream.write(mesh.operation("insert", {
	collection: "people",
	data: { name: "Sandra Bullock" }
}));

operationStream.write(mesh.operation("remove", {
	collection: "people",
	query: { name: "Jeff Golbusloom" }
}));
```

#### operation mesh.operation(name, option)

creates a new operation which can be written to a data source stream. See `mesh.open(bus)`.

```javascript
mesh.open(bus).write(mesh.operation("insert", {
	collection: "friends",
	data: { name: "Blakers" }
}));
```

#### operation mesh.op(name, options)

shorthand for `mesh.operation(name options)`.

#### bus mesh.top(bus)

`to operation` - Makes it so that you can simply call `bus(operationName, options)` instead of passing in the operation
each time.

```javascript
var bus = mesh.top(localStorage());

// enables this
bus("insert", {
	collection: "people",
	data: { name: "Jorge" }
});

// also accepts this
bus(mesh.operation("insert"));
```

#### bus mesh.child(bus, options)

Creates a new child data source. `options` is essentially just added to each operation performed.

```javascript
var peopleBus = mesh.top(mesh.child(bus, { collection: "people" }));

// insert a new person into the people collection
peopleBus("insert", {
  data: { name: "Shrek" }
});
```

#### bus mesh.tailable(bus, reject)

Makes the bus tailable. This simply allows you to listen for any operations invoked on a bus such as `create`, `update`, `remove`, and `load`.

`reject` is an array of operations to ignore. Default is `[load]`.

```javascript
var bus = mesh.tailable(localbus);

bus(mesh.op("tail", function() {

}));

var peopleBus = mesh.child(bus, { collection: "people" });

mesh
	.open(peopleBus)
	.write(mesh.op("insert", { data: { name: "Donkey" }})) // trigger tail
	.write(mesh.op("remove", { query: { name: "Donkey" }})) // trigger tail
	.write(mesh.op("update", { query: { name: "Donkey" }, data: { name: "Donkay" }})) // trigger tail
	.write(mesh.op("load", { query: { name: "Donkey" }})) // ignored by tail
	.end();
```

#### group mesh.parallel(...buss)

Combines data sources and executes operations in parallel.

<!-- note about emitting data multiple times -->

```javascript
var bus = mesh.parallel(localbus, httpBus);

// execute "load" on localbus at the same time
bus(mesh.op("load")).on("data", function() {
  // Note that his will get called TWICE
}).on("end", function() {
  // called when operation is executed on all buss
});
```

#### group mesh.sequence(...buss)

Combines data sources and executes operations in sequence.

<!-- note about emitting data multiple times -->

```javascript
var bus = mesh.top(mesh.sequence(localBus, httpBus));

// load data from localbus first, then move to httpBus
bus("load").on("data", function() {
  // Note that his will get called TWICE
});
```

#### group mesh.fallback(...busses)

Runs busses in sequence, but stops when a result is emitted from a data source.

```javascript
var bus = mesh.fallback(localStorage(), http());

// load data from local storage if it exists, or continue
// to http storage
bus(mesh.op("load", { collection: "people" })).on("data", function() {

});
```

#### group mesh.race(...busses)

Runs all busses in parallel, but only emits data from the fastest one.

```javascript

var busA = mesh.wrapCallback(function(operation, next) {
	setTimeout(next, 1, void 0, 'faster');
});

var busB = mesh.wrapCallback(function(operation, next) {
	setTimeout(next, 1, void 0, 'slower');
});


var bus = mesh.race(busA, busB);

// load data from local storage if it exists, or continue
// to http storage
bus(mesh.op("race!")).on("data", function(data) {
	console.log(data); // faster
});
```

#### group(operation)

A group of buses that are run in parallel or sequence.

#### group.add(...busses[, priority])

Adds a new bus to the group.

- `busses` - busses to add
- `priority` - priority for buses. `0` is high priority. `-Infinity` is highest priority.

For example:

```javascript
var mesh = require("mesh");

// global event bus used in the app.
var bus = mesh.parallel();

var router = mesh.race();

// register the HTTP router
bus.add(router);

var redirect = mesh.wrapCallback(function(operation, next) {
	location.hash = operation.pathname;
	next();
});

router.add(
	mesh.accept(mesh.op("redirect", { pathname: "/home" }), redirect),
	mesh.accept(mesh.op("redirect", { pathname: "/contact" }), redirect)
);

// redirect to the home page
bus(mesh.op("redirect", { pathname: "/home" }));
```

#### group.remove(bus)

removes a bus from the group

#### bus mesh.accept([...filter, ]bus)

Accepts only the provided operations.

```javascript
// main DB - api server
var httpBus = mesh.tailable(http());

// temporary cache
var localBus = localStorage();

// main DB - get cached data from local storage before
// checking the server
var bus = mesh.fallback(mesh.accept("load", localBus), httpBus);

// pipe all persistence operations back to local storage
httpBus(mesh.op("tail")).pipe(mesh.open(localBus));
```

you can also provide functions to filter ops:

```javascript
var bus = mesh.accept(function(operation) {
  return operation.name === "blah"
}, bus);
```

#### bus mesh.reject([...filter, ]bus)

Runs all operations except the ones provided.

<!--#### bus mesh.intercept()

Intercepts an operation based on the given test-->

#### mesh.run(bus, operationName, options, onRun)

Runs a data source operation

```javascript
mesh.run(peopleBus, "insert", { data: { name: "blarg"}}, function(err, insertedItem) {
	// handle result here
});
```

#### mesh.wrapCallback(callback)

wraps a callback as a bus handler

```javascript
var bus = mesh.parallel(
	mesh.accept("load", mesh.wrapCallback(function(operation, next) {
		// do stuff
	})),
	mesh.accept("showPopup", mesh.wrapCallback(function(operation, next) {
		document.body.appendChild(operation.element);
		next();
	}))
);

bus(mesh.op("showPopup", { element: document.createTextNode("Hello!") }));
```

### Building a custom bus adapter

Building a custom data source is pretty easy. All you need to do
is return a stream when `bus(opName, options)` is called.

Here's some scaffolding for a custom bus:

```javascript
// slimmed down version of node streams.
var stream = require("obj-stream");

function createBus(options) {

	// create adapter here

	// return fn that executes operations
	return function(operation) {
		var writable = stream.writable();

		// this is important so that data can be piped to other things
		process.nextTick(function() {

			// collection MUST exist
			if (!operation.collection) return writable.reader.emit("error", new Error("missing collection"));

			// perform task here

			// write data from insert/load
			writable.write(data);

			// must call end operation when complete
			writable.end();
		});

		return writable.reader;
	};
}
```

> Keep in mind that there are a few conventions you should follow when writing custom adapters. These conventions are here to ensure that they're are interoperable with each other.

#### bus(insert, options)

Insert a new item in the data source. Note that `data` is emitted for each item inserted in the data source.

- `options` - bus options
  - `data` - data to insert. Accepts 1 or many items
  - `collection` - collection to insert (optional for buss that don't have it)

```javascript
var _ = require("highland");
var peopleBus = mesh.top(mesh.child(bus, { collection: "people" }));

// insert one item
peopleBus("insert", {
  data: { name: "jeff" }
});

// insert many items & collect the results in 1
// array
peopleBus("insert", {
  data: [
    { name: "Joe" },
    { name: "Rogan" }
  ]
}).pipe(_.pipeline(_.collect)).on("data", function(people) {

});
```

#### bus(update, options)

Updates an item in the data source. Doesn't return any values.

- `options`
  - `query` - search query for items to update
  - `data`  - data to merge with
  - `collection` - bus collection
  - `multi` - `true` to update multiple items. `false` is default.

```javascript
var peopleBus = mesh.top(mesh.child(bus, {
	collection: "people"
}));

peopleBus("update", {
	query: { name: "jake"	},
	data: { age: 17 }
});

// update multiple items
peopleBus("update", {
	multi: true,
	query: { name: "joe" },
	data: {	age: 17 }
});
```

#### bus(upsert, options)

Updates an item if it exists. Inserts an item if it doesn't.

- `options`
  - `query` - search query for items to update
  - `data`  - data to merge or insert
  - `collection` - bus collection

```javascript
var peopleBus = mesh.top(mesh.child(bus, {
	collection: "people"
}));

// insert
peopleBus("upsert", {
	query: { name: "jake" },
	data: {
		name: "jake",
		age: 17
	}
}).on("end", function() {

	// update
	peopleBus("upsert", {
		query: { name: "jake" },
		data: {
			name: "jake",
			age: 18
		}
	})
});
```

#### bus(load, options)

Loads one or many items from the data source.

- `options`
  - `query` - search query for items
  - `collection` - bus collection
  - `multi` - `true` if loading multiple. `false` is default.

```javascript

var peopleBus = mesh.top(mesh.child(bus, { collection: "people" }));

// load one item
peopleBus("load", {
  query: { name: "tina" }
}).on("data", function() {
  // handle
});


// load many items
peopleBus("load", {
  multi: true,
  query: { name: "tina" }
}).pipe(_.pipeline(_.collect)).on("data", function(people) {
  // handle
});
```

#### bus(remove, options)

Removes an item from the data source.

- `options`
  - `query` - query to search
  - `collection` - collection to search
  - `multi` - `true` to remove multiple items

```javascript

// remove one item
bus("remove", {
  collection: "people",
  query: { name: "batman" }
});


// remove all instances where age = 54
bus("remove", {
  collection: "people",
  query: { age: 54 },
  multi: true
});
```
