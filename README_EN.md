# fast-streamsearch

Streaming searching for Node.js, but up to 100 times faster than [streamsearch](https://www.npmjs.com/package/streamsearch) 🚀.

<p align="center">
    <a href="https://www.npmjs.com/package/fast-streamsearch" target="_blank"><img src="https://img.shields.io/npm/v/fast-streamsearch.svg?style=flat-square" alt="Version"></a>
    <a href="https://npmcharts.com/compare/fast-streamsearch?minimal=true" target="_blank"><img src="https://img.shields.io/npm/dm/fast-streamsearch.svg?style=flat-square" alt="Downloads"></a>
    <a href="https://github.com/haochuan9421/fast-streamsearch" target="_blank"><img src="https://visitor-badge.glitch.me/badge?page_id=haochuan9421.fast-streamsearch"></a>
    <a href="https://github.com/haochuan9421/fast-streamsearch/commits/master" target="_blank"><img src="https://img.shields.io/github/last-commit/haochuan9421/fast-streamsearch.svg?style=flat-square" alt="Commit"></a>
    <a href="https://github.com/haochuan9421/fast-streamsearch/issues" target="_blank"><img src="https://img.shields.io/github/issues-closed/haochuan9421/fast-streamsearch.svg?style=flat-square" alt="Issues"></a>
    <a href="https://github.com/haochuan9421/fast-streamsearch/blob/master/LICENSE" target="_blank"><img src="https://img.shields.io/npm/l/@haochuan9421/fast-streamsearch.svg?style=flat-square" alt="License"></a>
</p>

[简体中文](https://github.com/haochuan9421/fast-streamsearch/blob/master/README.md)&emsp;
[English](https://github.com/haochuan9421/fast-streamsearch/blob/master/README_EN.md)&emsp;

## Features

- Fast
- Zero dependencies, just over 100 lines of code
- When match, you can set a new needle for the following content
- All callback data is safe (will not be changed)

## Installation

```bash
npm i fast-streamsearch
```

## Quick Start

```js
const { inspect } = require("util");
const FastStreamSearch = require("fast-streamsearch");

const needle = Buffer.from("\r\n");
const fss = new FastStreamSearch(needle, (isMatch, data) => {
  if (isMatch) {
    console.log("match!");
  } else {
    console.log("data: " + inspect(data.toString("latin1")));
  }
});

const chunks = [
  "foo",
  " bar",
  "\r",
  "\n",
  "baz, hello\r",
  "\n world.",
  "\r\n Node.JS rules!!\r\n\r\n",
];
for (const chunk of chunks) {
  fss.push(Buffer.from(chunk));
}

// data: 'foo'
// data: ' bar'
// match!
// data: 'baz, hello'
// match!
// data: ' world.'
// match!
// data: ' Node.JS rules!!'
// match!
// match!
```

You can even use different needles:

```js
const { inspect } = require("util");
const FastStreamSearch = require("fast-streamsearch");

const foo = Buffer.from("foo");
const bar = Buffer.from("bar");

const fss = new FastStreamSearch(foo, (isMatch, data) => {
  if (isMatch) {
    console.log(`match ${fss.needle.toString("latin1")}!`);
    if (fss.needle === foo) {
      fss.setNeedle(bar);
    }
  } else {
    console.log("data: " + inspect(data.toString("latin1")));
  }
});

const chunks = [
  "ab",
  "foo",
  "cd",
  "foo",
  "ef",
  "bar"
];
for (const chunk of chunks) {
  fss.push(Buffer.from(chunk));
}

// data: 'ab'
// match foo!
// data: 'cd'
// data: 'foo'
// data: 'ef'
// match bar!
```

## API

### Constructor

`new FastStreamSearch(needle: Buffer, callback: function)`

Create an instance for searching `needle`. `callback` is called when there is a needle match or non-matching data, the `callback` has following parameters:

- `isMatch`
  - type: `boolean`
  - description: `needle` is matched or not.
- `data`
  - type: `Buffer`
  - description: Non-matching data. When `isMatch` is `true`, `data` is empty, and vice versa.

### Properties

- `needle`
  - type: `Buffer`
  - description: The `needle` being used by the current search process.

### Methods

- `push(chunk: Buffer)`: Add a new `chunk` to be searched.
- `end()`: Should be called when the stream ends and will emit any last remaining unmatched data.
- `setNeedle(needle: Buffer)`: Update the `needle` used in the search process. Can be called only when the match is successful, otherwise it will lead to unpredictable consequences.
- `init()`: Reset the internal state, Useful for when you wish to start searching a new/different stream for example.

## streamsearch vs fast-streamsearch

The following result shows the time taken to search for a 4GB size stream.

<img width="221" alt="image" src="https://user-images.githubusercontent.com/5093611/211139840-9f7ad768-0109-4237-b17a-6c366233305b.png">

## Why fast-streamsearch is faster

fast-streamsearch uses the [boyer-moore-horspool](https://en.wikipedia.org/wiki/Boyer%E2%80%93Moore%E2%80%93Horspool_algorithm) algorithm for the entire search process but when the content to be matched can be matched directly using Buffer's [indexOf](https://nodejs.org/api/buffer.html#bufindexofvalue-byteoffset-encoding) method, `indexOf` is used first to drives the search pointer forward based on the `indexOf` result. The advantage of this is that the native `indexOf` is usually faster than the horspool algorithm, especially when the needle is short, and the horspool algorithm is a little faster than `indexOf` when the needle is long, but not by much. So fast-streamsearch is usually faster than streamsearch based on horspool algorithm only, especially when the needle is short. When the needle length is 1, it can even be as much as 100 times faster than streamsearch, and when the needle length is under 256, there is at least a 2 times performance advantage, and in most search scenarios, the needle is not too long. So fast-streamsearch is usually the better choice, but streamsearch is also a great tool.

Here is a piece of code that compares the speed of horspool with `indexOf`:

```js
function horspool(haystack, needle) {
  const len = needle.length;
  const last = len - 1;
  const move = new Array(256).fill(len);
  for (let i = 0; i < last; i++) {
    move[needle[i]] = last - i;
  }

  let i = 0;
  let j = 0;
  while (i <= haystack.length - len) {
    const start = i;

    while (j < len && haystack[i] === needle[j]) {
      i++;
      j++;
    }

    if (j === len) {
      return start;
    }

    i = start + move[haystack[start + last]];
    j = 0;
  }
  return -1;
}

const haystack = Buffer.allocUnsafe(2 ** 30); // 1GB
for (let i = 0; i < haystack.length; i++) {
  haystack[i] = Math.floor(Math.random() * 128);
}

const needleLen = 1;
const needle = Buffer.allocUnsafe(needleLen);
for (let i = 0; i < needle.length; i++) {
  needle[i] = 128 + Math.floor(Math.random() * 128);
}

console.time("indexOf");
haystack.indexOf(needle);
console.timeEnd("indexOf");

console.time("horspool");
horspool(haystack, needle);
console.timeEnd("horspool");
```
