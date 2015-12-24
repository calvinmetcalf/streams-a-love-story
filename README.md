Streams and You: A Love Story
====

Talk for node-interactive, viewable at [streams.how](http://streams.how).

Buffering
===

Streams are 'semilazy' or 'quasieager' or they have 'mixed feelings'.  What that means is that streams will eagerly consume or produce data to fill an internal buffer but then stop once that buffer is filled up and will only start producing more data when that buffer in danger of getting empty.  This internal buffer can be tweaked with the `highWaterMark` option you can pass to the constructor.

Object Streams vs Binary Streams.
===

All internal node streams and many userland streams are 'binary' streams.  They take a buffer or text and emit a buffer or text. These represent views into a larger piece of data so in theory could be combined or split at will, in other words if you have multiple messages you are writing to a stream you can't count on that boundary being respected by stream chunks in a binary stream.  By default if you pass in a string instead of a buffer it gets turned into a buffer with the encoding being assumed to be 'utf8' (the exception is the crypto module which always defaults to 'binary').  Most places you can write something to a stream accept an encoding argument, these are the same as with a buffer and include 'hex' and 'base64'.  Binary streams will throw an error if you try to write an object to them.  Some methods allow you to specify how much data you want from a stream, they will return at most this amount, they may return less if the stream ends.    The internal buffer is measured in bytes and defaults to 1024 * 1024 * 16 bytes (16 kb).

Object mode streams are streams where each chunk of data is a single JavaScript object.  Objects may be Buffers or strings (so this would be how you would create a stream of binary messages btw). The encoding option is always ignored for object mode streams.  You can enable the object mode option by passing `objectMode` : `true` to the stream constructor.  The internal buffer is measured in terms of the number of objects and defaults to 16 objects.  You can't write `null` to an object stream because null is used to specify end of stream.

How to make a stream
===

Node provides stream objects that are missing a function that you need to supply. You need to either pass that function to the constructor in the options object or subclass the stream and provide a method onto the prototype with a leading underscore.

For instance readable stream takes a function called `read` which is called with a single argument of 'length' which is purely advisory and represents how much data is needed, if you don't give it enough it will just get more when it needs it. You can ignore this argument when creating objectMode streams.  You produce data in a readable stream by call the the internal `push` method which either returns false if the internal buffer is full or true if you can write in more data.  When you have no more data to write then write `null`.

N.B. I find the API for creating a readable stream very confusing especially for async data sources and wrote a module called [noms](https://npmjs.org/noms) that I feel makes it MUCH easier to write them, but I'm just going to cover the internal streams here.

Say we wanted to make a stream that took a user supplied number and emitted all the numbers between 0 and that number, there are 3 ways to do it.

You could do an ES5 style sub classing of the stream.

```js
var inherits = require('utils').inherits;
var Readable = require('stream').Readable;

inherits(CountingStream, Readable);
function CountingStream(max) {
  Readable.call(this, {
    objectMode: true
  });
  this.max = max;
  this.cur = 0;
}
CountingStream.prototype._read = function () {
  var result;
  do {
    if (this.cur > this.max) {
      return this.push(null);
    }
    result = this.push(this.cur++);
  } while (result)
}
```

In newer versions of node you can also use ES6 classes

```js
var Readable = require('stream').Readable;

class CountingStream extends Readable {
  constructor(max) {
    super({
      objectMode: true
    });
    this.max = max;
    this.cur = 0;
  }
  _read() {
    var result;
    do {
      if (this.cur > this.max) {
        return this.push(null);
      }
      result = this.push(this.cur++);
    } while (result)
  }
}
```

You can also make a once of stream instances without making a full class first

```js
var Readable = require('stream').Readable;

function countingStream(max) {
  var cur = 0;
  return new Readable({
    objectMode: true,
    read() {
      var result;
      do {
        if (cur > max) {
          return this.push(null);
        }
        result = this.push(cur++);
      } while (result)
    }
  })
}
```

Now should you create a class or directly create a stream?  The answer depends, but my rule of thumb us:

If it is a one off stream used at a specific point in a program and nowhere else I create an instance directly.  If it's a stream that is being used multiple times, especially if it's in a library then I create a class.


Piping
===

Piping has some sharp edges, notably errors are not forwarded and instead must be listened to on each stream in the pipe chain, if a destination stream errors out the streams further back in the chain aren't cleaned up, checkout [pump](https://npmjs.org/pump) for a utility to make piping behave like you'd expect and [pumpify](https://www.npmjs.com/package/pumpify) which is good for dealing with long piping chains where a pipes to b which pipes to etc.

Reading Data
===

You can listen for the `data` event of the stream, or you can listen for the `readable` event and then call `.read()` it returns `null` and then listen for the `readable` event again. In both cases you'd want to also listen for the `end` event to know there is nothing left to read.

In the incredibly common scenario you just want all the data when it's done you can use [concat-stream](https://www.npmjs.com/package/concat-stream).

Combining Streams
===

If you want to create a stream but you need to acquire part or all of it asynchronously then [duplexify](https://www.npmjs.com/package/duplexify) is super helpful.  It is also useful when you want to make your own duplex stream because you have a readable and a writable but want them to appear to be a single stream.

through2
===
through2 used to be an easy way to make transform streams, it's not necessary any more now that you can pass functions to the constructor.

with through2

```js
var myStream = through2({
  objectMode: true
}, function (chunk, enc, next) {}, function (done) {});
```

or


```js
var myStream = through2.obj( function (chunk, enc, next) {}, function (done) {});
```

With the new way of making streams in node

```js
var myStream = new stream.Transform({
  objectMode: true,
  transform (chunk, enc, next) {},
  flush (done) {}
});
```
