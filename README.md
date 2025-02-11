# rawproto

Guess structure of protobuf binary from raw data

Very similar to `protoc --decode_raw`, but for javascript.

You can use this to reverse-engineer a protobuf protocol, based on a binary protobuf string.

See some example output (from the demo message in this repo) [here](https://gist.github.com/konsumer/3647d466b497e6950b12291e47f11eeb).

## installation

`npm i -S rawproto` will add this to your project.

If you just want the CLI, and don't use node, you can also find standalone builds [here](https://github.com/konsumer/rawproto/releases).


## usage

In ES6;

```js
import { readFileSync, writeFileSync } from 'fs'
import { getData, getProto } from 'rawproto'

const buffer = readFileSync('data.pb')

// get info about binary protobuf message
console.log( getData(buffer) )

// save guessed proto file for this binary data
writeFileSync('data.proto', getProto(buffer) )

```

In plain CommonJS:

```js
var fs = require('fs')
var rawproto = require('rawproto')

var buffer = fs.readFileSync('data.pb')

// get info about binary protobuf message
console.log( rawproto.getData(buffer) )

// save guessed proto file for this binary data
fs.writeFileSync('data.proto', rawproto.getProto(buffer) )

```

For both `getProto` and `getData` the second param is `stringMode`.

This is to decide how to deal with strings:

* `auto` - (default) guess how to deal with strings based on characters in string
* `string` - assume they are all strings. This can mangle strings that contain any extended chars
* `binary` - assume all strings are buffers 

```js
// get info about binary protobuf message, assume all strings are binary
console.log( rawproto.getData(buffer, 'binary') )
```

## cli

You can also use rawproto to parse binary on the command-line!

Install with `npm i -g rawproto` or use it without installation with `npx rawproto`.

If you just want the CLI, and don't use node, you can also find standalone builds [here](https://github.com/konsumer/rawproto/releases).

Use it like this:

```
cat myfile.pb | rawproto parse
```

or

```
rawproto parse < myfile.pb
```

```
Usage: rawproto <COMMAND>

Commands:
  rawproto guess  Guess the proto definition
  rawproto parse  Raw-parse the binary protobuf

Options:
  --version  Show version number                                       [boolean]
  --help     Show help                                                 [boolean]
```

so to putput a raw-json parse:

```
npx rawproto parse < ~/Downloads/details.pb
```

or guess the proto-structure:

```
npx rawproto guess < ~/Downloads/details.pb
```

## http

I noticed that various HTTP request libraries convert binary buffers into wrong-encoded text, and this can cause issues. If you build your buffer manually with `Buffer.concat`, it seems to work ok:

```js
import http from 'http'
import https from 'https'
import { getProto } from 'rawproto'

const getRaw = url => new Promise((resolve, reject) => {
  const get = url.substr(0,5) === 'https' ? https.get : http.get
  get(url, response => {
    if (response.statusCode < 200 || response.statusCode > 299) {
      return reject(new Error('Failed to load page, status code: ' + response.statusCode))
    }
    const body = []
    response.on('error', (err) => reject(err))
    response.on('data', (chunk) => body.push(chunk))
    response.on('end', () => resolve(getProto(Buffer.concat(body))))
  })
})

// use like this:
getRaw(YOUR_URL)
  .then(p => {
    // do stuff with p
  })
```


You can use `fetch`, like this (in ES6 with top-level `await`):

```js
import { getData } from 'rawproto'
import { fetch } from 'node-fetch'

const r = await fetch('YOUR_URL_HERE')
const b = await r.arrayBuffer()
console.log(getData(Buffer.from(b)))
```


## limitations

There are several types that just can't be guessed from the data. signedness and precision of numbers can't really be guessed, ints could be enums, and my `auto` system of guessing if it's a `string` or `bytes` is naive (but I don't think could be improved without any knowledge of the protocol.)

You should definitely tune the outputted proto file to how you think your data is structured. I add comments to fields, to help you figure out what [scalar-types](https://developers.google.com/protocol-buffers/docs/proto3#scalar) to use, but without the original proto file, you'll have to do some guessing of your own. The bottom-line is that the generated proto won't cause an error, but it's probably not exactly correct, either.


## todo

* Streaming data-parser for large input
* Collection analysis: better type-guessing with more messages
* `getTypes` that doesn't mess with JS data, and just gives possible types of every field
* partial-parsing like `protoc --decode`. It basically tries to decode, but leaves unknown fields raw.
