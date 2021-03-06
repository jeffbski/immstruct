Immstruct [![NPM version][npm-image]][npm-url] [![Build Status][travis-image]][travis-url] [![Dependency Status][depstat-image]][depstat-url] [![Gitter][gitter-image]][gitter-url]
======

A wrapper for [Immutable.js](https://github.com/facebook/immutable-js/tree/master/contrib/cursor) to easily create cursors that notify when they
are updated. Handy for use with immutable pure components for views,
like with [Omniscient](https://github.com/omniscientjs/omniscient) or [React.js](https://github.com/facebook/react).

## Usage

```js
// someFile.js
var immstruct = require('immstruct');
var structure = immstruct('myKey', { a: { b: { c: 1 } } });

// Use event `swap` or `next-animation-frame`
structure.on('swap', function (newStructure, oldStructure, keyPath) {
  console.log('Subpart of structure swapped.');
  console.log('New structure:', newStructure.toJSON());

  // e.g. with usage with React
  // React.render(App({ cursor: structure.cursor() }), document.body);
});

var cursor = structure.cursor(['a', 'b', 'c']);

// Update the value at the cursor. As cursors are immutable,
// this returns a new cursor that points to the new data
var newCursor = cursor.update(function (x) {
  return x + 1;
});

// The value of the old `cursor` to is still `1`
console.log(cursor.deref()); //=> 1

// `newCursor` points to the new data
console.log(newCursor.deref()); //=> 2
```


```js
// anotherFile.js
var immstruct = require('immstruct');
var structure = immstruct('myKey');

var cursor = structure.cursor(['a', 'b', 'c']);

var updatedCursor = cursor.update(function (x) { // triggers `swap` in somefile.js
  return x + 1;
});

console.log(updatedCursor.deref()); //=> 3
```

## References

While Immutable.js cursors are immutable, Immstruct lets you create references
to a piece of data from where cursors will always be fresh.

```js

var structure = immstruct({ 'foo': 'bar' });
var ref = structure.reference('foo');

console.log(ref.cursor().deref()) //=> 'bar'

var oldCursor = structure.cursor('foo');
console.log(oldCursor.deref()) //=> 'bar'

var newCursor = structure.cursor('foo').update(function () { return 'updated'; });
console.log(newCursor.deref()) //=> 'updated'

assert(oldCursor !== newCursor);

// You don't need to manage and track fresh/stale cursors.
// A reference cursor will do it for you.
console.log(ref.cursor().deref()) //=> 'updated'
```

Updating a cursor created from a reference will also update the underlying structure.

This offers benefits similar to that of [Om](https://github.com/omcljs/om/wiki/Advanced-Tutorial#reference-cursors)'s `reference cursors`, where
[React.js](http://facebook.github.io/react/) or [Omniscient](https://github.com/omniscientjs/omniscient/) components can observe pieces of application
state without it being passed as cursors in props from their parent components.

References also allow for listeners that fire when their path or the path of sub-cursors change:

```js
var structure = immstruct({
  someBox: { message: 'Hello World!' }
});
var ref = structure.reference(['someBox']);

var unobserve = ref.observe(function () {
  // Called when data the path 'someBox' is changed.
  // Also called when the data at ['someBox', 'message'] is changed.
});

// Update the data using the ref
ref.cursor().update(function () { return 'updated'; });

// Update the data using the initial structure
structure.cursor(['someBox', 'message']).update(function () { return 'updated again'; });

// Remove the listener
unobserve();
```

### Notes

Parents' change listeners are also called when sub-cursors are changed.

Cursors created from references are still immutable. If you keep a cursor from
a `var cursor = reference.cursor()` around, the `cursor` will still point to the data
at time of cursor creation. Updating it may rewrite newer information.

## Usage Undo/Redo

```js
// optionalKey and/or optionalLimit can be omitted from the call
var optionalLimit = 10; // only keep last 10 of history, default Infinity
var structure = immstruct.withHistory('optionalKey', optionalLimit, { 'foo': 'bar' });
console.log(structure.cursor('foo').deref()); //=> 'bar'

structure.cursor('foo').update(function () { return 'hello'; });
console.log(structure.cursor('foo').deref()); //=> 'hello'

structure.undo();
console.log(structure.cursor('foo').deref()); //=> 'bar'

structure.redo();
console.log(structure.cursor('foo').deref()); //=> 'hello'

```

## Examples

Creates or retrieves [structures](#structure--eventemitter).

See examples:

```js
var structure = immstruct('someKey', { some: 'jsObject' })
// Creates new structure with someKey
```


```js
var structure = immstruct('someKey')
// Get's the structure named `someKey`.
```

**Note:** if someKey doesn't exist, an empty structure is created

```js
var structure = immstruct({ some: 'jsObject' })
var randomGeneratedKey = structure.key;
// Creates a new structure with random key
// Used if key is not necessary
```


```js
var structure = immstruct()
var randomGeneratedKey = structure.key;
// Create new empty structure with random key
```

You can also create your own instance of Immstruct, isolating the
different instances of structures:

```js
var localImmstruct = new immstruct.Immstruct()
var structure = localImmstruct.get('someKey', { my: 'object' });
```

## API Reference

See [API Reference](./api.md).

## Structure Events

A Structure object is an event emitter and emits the following events:

* `swap`: Emitted when cursor is updated (new information is set). Emits no values. One use case for this is to re-render design components. Callback is passed arguments: `newStructure`, `oldStructure`, `keyPath`.
* `next-animation-frame`: Same as `swap`, but only emitted on animation frame. Could use with many render updates and better performance. Callback is passed arguments: `newStructure`, `oldStructure`.
* `change`: Emitted when data/value is updated and it existed before. Emits values: `path`, `newValue` and `oldValue`.
* `delete`: Emitted when data/value is removed. Emits value: `path` and `removedValue`.
* `add`: Emitted when new data/value is added. Emits value: `path` and `newValue`.

**NOTE:** If you update cursors via `Cursor.update` or `Cursor.set`, and if the underlying Immutable collection is not inherently changed, `swap` and `changed` events will not be emitted, neither will the history (if any) be applied.

[See tests for event examples](./tests/structure_test.js)

[npm-url]: https://npmjs.org/package/immstruct
[npm-image]: http://img.shields.io/npm/v/immstruct.svg?style=flat

[travis-url]: http://travis-ci.org/omniscientjs/immstruct
[travis-image]: http://img.shields.io/travis/omniscientjs/immstruct.svg?style=flat

[depstat-url]: https://gemnasium.com/omniscientjs/immstruct
[depstat-image]: http://img.shields.io/gemnasium/omniscientjs/immstruct.svg?style=flat

[gitter-url]: https://gitter.im/omniscientjs/omniscient?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge
[gitter-image]: https://badges.gitter.im/Join%20Chat.svg

## License

[MIT License](http://en.wikipedia.org/wiki/MIT_License)
