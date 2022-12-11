# next-session

[![npm](https://badgen.net/npm/v/next-session)](https://www.npmjs.com/package/next-session)
[![minified size](https://badgen.net/bundlephobia/min/next-session)](https://bundlephobia.com/result?p=next-session)
[![CircleCI](https://circleci.com/gh/hoangvvo/next-session.svg?style=svg)](https://circleci.com/gh/hoangvvo/next-session)
[![codecov](https://codecov.io/gh/hoangvvo/next-session/branch/master/graph/badge.svg)](https://codecov.io/gh/hoangvvo/next-session)
[![PRs Welcome](https://badgen.net/badge/PRs/welcome/ff5252)](CONTRIBUTING.md)

一个基于 Promise，适用于[Next.js](https://github.com/zeit/next.js)的轻量级 Session 中间件。

> 你也可以使用例如[next-iron-session](https://github.com/vvo/next-iron-session)等其他库。 [nextjs-mongodb-app](https://github.com/hoangvvo/nextjs-mongodb-app) 中有这个库的具体使用 demo。

## 安装

```sh
// NPM
npm install next-session
// Yarn
yarn add next-session
```

## 使用

:point_right: **从 v1.x 升级到 v2.x?** [点击这里阅读升级日志](https://github.com/hoangvvo/next-session/releases/tag/v2.0.0)!

:point_right: **从 v2.x 升级到 v3.x?** [点击这里阅读升级日志](https://github.com/hoangvvo/next-session/releases/tag/v3.0.0)!

:point_right: **从 v3.x 升级到 v4.x?** [点击这里阅读升级日志](https://github.com/hoangvvo/next-session/releases/tag/v4.0.0)!

**注意** 在 `options?.store`为`undefined`时默认使用的`MemoryStore`**并不**在生产环境或是 Serverless 环境中生效。你必须使用一个[Session Store](#session-store).

```js
// ./lib/get-session.js
import nextSession from "next-session";
export const getSession = nextSession(options);
```

### 在 API 路由中使用

```js
import { getSession } from "./lib/get-session.js";

export default function handler(req, res) {
  const session = await getSession(req, res);
  session.views = session.views ? session.views + 1 : 1;
  // 也在req.session中生效:
  // req.session.views = req.session.views ? req.session.views + 1 : 1;
  res.send(
    `In this session, you have visited this website ${session.views} time(s).`
  );
}
```

在 API 路由中可能会导致`API resolved without sending a response`。 这可以添加一下逻辑解决:

```js
import nextSession from "next-session";
const getSession = nextSession();

export default function handler(req, res) {
  const session = await getSession(req, res);
  /* ... */
}

export const config = {
  api: {
    externalResolver: true,
  },
};
```

或者将`options.autoCommit`设置为`false`并且调用`await session.commit()`.

```js
import nextSession from "next-session";
const getSession = nextSession({ autoCommit: false });

export default function handler(req, res) {
  const session = await getSession(req, res);
  /* ... */
  await session.commit();
}
```

### 在 getServerSideProps 中使用

```js
import { getSession } from "./lib/get-session.js";

export default function Page({ views }) {
  return (
    <div>In this session, you have visited this website {views} time(s).</div>
  );
}

export async function getServerSideProps({ req, res }) {
  const session = await getSession(req, res);
  session.views = session.views ? session.views + 1 : 1;
  // Also available under req.session:
  // req.session.views = req.session.views ? req.session.views + 1 : 1;
  return {
    props: {
      views: session.views,
    },
  };
}
```

### 其他的使用 DEMO

[express](https://github.com/expressjs/express), [next-connect](https://github.com/hoangvvo/next-connect)

```js
const express = require("express");
const app = express();
app.use(async (req, res, next) => {
  await getSession(req, res); // session is set to req.session
  next();
});
app.get("/", (req, res) => {
  req.session.views = req.session.views ? req.session.views + 1 : 1;
  res.send(
    `In this session, you have visited this website ${req.session.views} time(s).`
  );
});
```

[micro](https://github.com/vercel/micro), [Vercel Serverless Functions](https://vercel.com/docs/functions/introduction)

```js
module.exports = (req, res) => {
  const session = await getSession(req, res);
  res.end(
    `In this session, you have visited this website ${session.views} time(s).`
  );
};
```

[Node.js HTTP Server](https://nodejs.org/api/http.html)

```js
const http = require("http");

const server = http.createServer(async (req, res) => {
  const session = await getSession(req, res);
  res.end(`In this session, you have visited this website ${session.views} time(s).`;
});
server.listen(8080);
```

## Options

`next-session` 选项的参数如下：

| 选项            | 描述                                                                                           | 默认值                                   |
| --------------- | ---------------------------------------------------------------------------------------------- | ---------------------------------------- |
| name            | cookie 在请求或是相应中的 name。                                                               | `sid`                                    |
| store           | Session 存储实例， 在生产环境中 **必须使用**！                                                 | `MemoryStore`                            |
| genid           | 生成新 Session ID 的算法函数                                                                   | [`nanoid`](https://github.com/ai/nanoid) |
| encode          | 在设置 session id 对其进行装换。这个函数会拿到原始 session id 返回编码/加密后的 session id.    | undefined                                |
| decode          | 解码或解密 cookie 中的 session id。这个函数会拿到编码/加密后 session id 返回原始的 session id. | undefined                                |
| touchAfter      | 自动延长 session 有效期。 详情看 [touchAfter](#touchAfter)。                                   | `-1` (Disabled)                          |
| autoCommit      | 自动提交 session。 如果你想手动调用`session.commit()`，可以将这项设置为`false`。               | `true`                                   |
| cookie.secure   | 设置 cookie 的**Secure**属性。                                                                 | `false`                                  |
| cookie.httpOnly | 设置 cookie 的 **httpOnly** 。                                                                 | `true`                                   |
| cookie.path     | 设置 cookie 的 **Path**。                                                                      | `/`                                      |
| cookie.domain   | 设置 cookie 的 **Domain**。                                                                    | unset                                    |
| cookie.sameSite | 设置 cookie 的 **SameSite**。                                                                  | unset                                    |
| cookie.maxAge   | 设置 cookie 的 **Max-Age**。**（单位为秒）**                                                   | unset (Browser session)                  |

### touchAfter

如果 touchAfter 设置了非负值，在解析当前 session 时，检测到距离当前 session 生效时间超过了 touchAfter 秒后，自动延长当前 session 过期时间并设置当前时间为生效时间。touchAfter 会在一定程度上防止当前 session 在一段时间后过期。
为了性能优化，在 `autoCommit` 模式中 (默认开启)如果当前 session 没有修改，那么当前 session 只会被更新访问时间。

### encode / decode

你可以修改每个请求的 cookie _编/解码_ 或是 _加/解密_ 的方式。

```javascript
// `express-session` signing strategy
const signature = require("cookie-signature");
const secret = "keyboard cat";
session({
  decode: (raw) => signature.unsign(raw.slice(2), secret),
  encode: (sid) => (sid ? "s:" + signature.sign(sid, secret) : null),
});
```

## API

### session 对象

这个对象允许你 **设置** 或是 **获取** 与当前 Session 关联的相关字段的值。

```javascript
//  Set a value
if (loggedIn) session.user = "John Doe";
//  Get a value
const currentUser = session.user; // "John Doe"
```

### session.touch()

手动修改 session 的访问时间。 以当前时间为基础，延长一个 maxAge 的 session 有效期。 **注意:** 如果设置了`autoCommit = false`的话，你还需要手动调用 session.commit()。

```js
session.touch();
```

如果 `touchAfter` 被设置了非负的值, 这个方法会被自动执行。

### session.destroy()

销毁并从 Store 中删除当前的 Session 数据。

```javascript
if (loggedOut) await session.destroy();
```

### session.commit()

保存 Session 并设置必要的 headers。这个方法必须在 _发送 headers (`res.writeHead`) 或 响应 (`res.send`、`res.end`等等)_ 前被调用。

如果 `autoCommit` 是 `false`，在变更 session 后 **必须** 使用该方法。

```javascript
session.hello = "world";
await session.commit();
// always calling res.end or res.writeHead after the above
```

### session.id

与当前 Session 相关联的唯一 ID。

## Session Store

next-session 中间件使用的存储实例（详见上方的`options`项）。

### 实践

适用的 session store 必须包含 `set(sid, session)`, `get(sid)`, 和 `destroy(sid)`三个方法。建议也包含一个`touch(sid, session)`方法。这几个方法要返回 **Promises**。

可以参考 [MemoryStore](https://github.com/hoangvvo/next-session/blob/master/src/memory-store.ts)源码。

_TypeScript:_ `SessionStore` 这个类型可用来协助编写自定义 store:

```ts
import type { SessionStore } from "next-session";

class CustomStore implements SessionStore {}
```

### 适配 Express/Connect stores

#### Promisify 函数

为了使用 [Express/Connect stores](https://github.com/expressjs/session#compatible-session-stores), 你必须将 `get`、`set`、`destroy` 和 `touch`（如果存在的话）这个几个方法 promise 化, 也可以是使用 Nodejs 自带的 [`util.promisify`](https://nodejs.org/dist/latest/docs/api/util.html#util_util_promisify_original)工具。

我们在 `next-session/lib/compat` 内置了 [`promisifyStore`](./src/compat.ts#L29) 工具来实现这一点:

```js
import nextSession from "next-session";
import { promisifyStore } from "next-session/lib/compat";
import SomeConnectStore from "connect-xyz";

const connectStore = new SomeConnectStore();

const getSession = nextSession({
  store: promisifyStore(connectStore),
});
```

如果你的 Connect Store 符合如下情况，你可以使用 `next-session/lib/compat` 中的 `expressSession`。

```javascript
const session = require("express-session");
const RedisStore = require("connect-redis")(session);

// Use `expressSession` from `next-session/lib/compat` as the replacement

import nextSession from "next-session";
import { expressSession, promisifyStore } from "next-session/lib/compat";
import RedisStoreFactory from "connect-redis";
import Redis from "ioredis";

const RedisStore = RedisStoreFactory(expressSession);
export const getSession = nextSession({
  store: promisifyStore(
    new RedisStore({
      client: new Redis(),
    })
  ),
});
```

## 建设

[点击这里查看相关文档](CONTRIBUTING.md).

## License

[MIT](LICENSE)
