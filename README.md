# fast-streamsearch

fast-streamsearch 是一款 Node.js 流搜索工具，最快可以比 [streamsearch](https://www.npmjs.com/package/streamsearch) 快 100 倍 🚀。

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

## 特点

- 速度快
- 零依赖，仅 100 多行代码
- 匹配成功时可以为后面的内容设置新的 needle
- 所有回调的 data 都是安全的（不会被改变）

## 安装

```bash
npm i fast-streamsearch
```

## 快速开始

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

你甚至可以使用不同的 needle:

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

### 构造函数

`new FastStreamSearch(needle: Buffer, callback: function)`

创建一个用于搜索 `needle` 的实例。当匹配成功时或有肯定不匹配的数据时，回调函数会执行，回调参数如下:

- `isMatch`
  - 类型: `boolean`
  - 说明: 表示是否匹配到了 `needle`。
- `data`
  - 类型: `Buffer`
  - 说明: 不可能匹配上 `needle` 的数据。当 `isMatch` 为 `true` 时，`data` 为空，反之，`data` 不为空

### 属性

- `needle`
  - 类型: `Buffer`
  - 说明: 当前匹配过程正在使用的 `needle`

### 方法

- `push(chunk: Buffer)`: 添加新的待搜索 `chunk`。
- `end()`: 当流结束时调用，会回调出最后剩余的未匹配数据。
- `setNeedle(needle: Buffer)`: 更新搜索过程中使用的 `needle`，只有当匹配成功时可以调用，否则会导致不可预期的后果。
- `init()`: 重置内部状态，如果你想搜索一个新的流时，可以使用，否则请不要调用。

## streamsearch vs fast-streamsearch

以下结果为搜索一个 4GB 大小的流的耗时情况

<img width="221" alt="image" src="https://user-images.githubusercontent.com/5093611/211139840-9f7ad768-0109-4237-b17a-6c366233305b.png">

## 为什么 fast-streamsearch 更快

fast-streamsearch 的整个搜索过程采用的是 [boyer-moore-horspool](https://en.wikipedia.org/wiki/Boyer%E2%80%93Moore%E2%80%93Horspool_algorithm) 算法，但当待匹配内容可以直接使用 Buffer 原生的 [indexOf](https://nodejs.org/api/buffer.html#bufindexofvalue-byteoffset-encoding) 方法进行匹配时，会优先使用 `indexOf` 来进行匹配，并根据 `indexOf` 的结果推动整体的搜索指针前进。这么做的好处在于原生的 `indexOf` 通常比 horspool 算法有更快的速度，特别是 needle 较短时，速度优势非常明显，horspool 算法在 needle 较长时会比 `indexOf` 快一点，但优势不明显。所以 fast-streamsearch 通常会比只基于 horspool 算法的 streamsearch 更快，尤其是 needle 较短时。当 needle 长度为 1 时，甚至可以比 streamsearch 快 100 倍之多，needle 长度在 256 以下时，也有至少 2 倍的性能优势，大部分搜索场景，needle 都不会太长。所以 fast-streamsearch 通常是更好的选择，不过 streamsearch 也是一个很棒的工具。

附上一段 horspool 与 `indexOf` 速度对比的代码:

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
