# r2d2

[![Docs](https://doc.deno.land/badge.svg)](https://doc.deno.land/https://deno.land/x/r2d2/mod.ts)
[![CI](https://github.com/iuioiua/r2d2/actions/workflows/ci.yml/badge.svg)](https://github.com/iuioiua/r2d2/actions/workflows/ci.yml)
[![codecov](https://codecov.io/gh/iuioiua/r2d2/branch/main/graph/badge.svg?token=8IDAVSL014)](https://codecov.io/gh/iuioiua/r2d2)

Fast, lightweight and simple Redis client library for
[Deno](https://deno.land/).

## Features

- Supports [RESPv2](#respv2), [RESP3](#resp3), [raw data](#raw-data),
  [pipelining](#pipelining), [pub/sub](#pubsub), [transactions](#transactions),
  [eval scripts](#eval-script) and [Lua scripts](#lua-script).
- The fastest Redis client in Deno. [See below](#benchmarks) and
  [try benchmarking yourself](#contributing)!
- Written to be easily understood and debugged.
- Encourages the use of actual Redis commands without intermediate abstractions.

## Usage

Must be run with `--allow-net` permission. Check out the full documentation
[here](https://doc.deno.land/https://deno.land/x/r2d2/mod.ts).

### RESPv2

```ts
import { sendCommand } from "https://deno.land/x/r2d2/mod.ts";

const redisConn = await Deno.connect({ port: 6379 });

// Returns "OK"
await sendCommand(redisConn, ["SET", "hello", "world"]);

// Returns "world"
await sendCommand(redisConn, ["GET", "hello"]);
```

If you don't care about the reply:

```ts
import { writeCommand } from "https://deno.land/x/r2d2/mod.ts";

const redisConn = await Deno.connect({ port: 6379 });

// Returns nothing
await writeCommand(redisConn, ["SHUTDOWN"]);
```

### RESP3

```ts
import { sendCommand } from "https://deno.land/x/r2d2/mod.ts";

const redisConn = await Deno.connect({ port: 6379 });

// Switch to RESP3 protocol
await sendCommand(redisConn, ["HELLO", 3]);

// Returns 2
await sendCommand(redisConn, ["HSET", "hash3", "foo", 1, "bar", 2]);

// Returns { foo: "1", bar: "2" }
await sendCommand(redisConn, ["HGETALL", "hash3"]);
```

### Raw data

Set the last argument, `raw`, to `true` and bulk string replies will return raw
data instead of strings.

```ts
import { sendCommand } from "https://deno.land/x/r2d2/mod.ts";

const redisConn = await Deno.connect({ port: 6379 });

const data = new Uint8Array([0, 1, 2, 3, 4, 5, 6, 7, 8, 9]);

// Returns "OK"
await sendCommand(redisConn, ["SET", "binary", data]);

// Returns same value as `data` variable
await sendCommand(redisConn, ["GET", "binary"], true);
```

### Pipelining

```ts
import { pipelineCommands } from "https://deno.land/x/r2d2/mod.ts";

const redisConn = await Deno.connect({ port: 6379 });

// Returns [1, 2, 3, 4]
await pipelineCommands(redisConn, [
  ["INCR", "X"],
  ["INCR", "X"],
  ["INCR", "X"],
  ["INCR", "X"],
]);
```

### Pub/Sub

```ts
import { readReplies, writeCommand } from "https://deno.land/x/r2d2/mod.ts";

const redisConn = await Deno.connect({ port: 6379 });

await writeCommand(redisConn, ["SUBSCRIBE", "mychannel"]);
for await (const reply of readReplies(redisConn)) {
  // Prints ["subscribe", "mychannel", 1] first iteration
  console.log(reply);
}
```

### Transactions

```ts
import { sendCommand } from "https://deno.land/x/r2d2/mod.ts";

const redisConn = await Deno.connect({ port: 6379 });

// Returns "OK"
await sendCommand(redisConn, ["MULTI"]);

// Returns "QUEUED"
await sendCommand(redisConn, ["INCR", "FOO"]);

// Returns "QUEUED"
await sendCommand(redisConn, ["INCR", "FOO"]);

// Returns [1, 1]
await sendCommand(redisConn, ["EXEC"]);
```

### Eval Scripts

```ts
import { sendCommand } from "https://deno.land/x/r2d2/mod.ts";

const redisConn = await Deno.connect({ port: 6379 });

// Returns "hello"
await sendCommand(redisConn, ["EVAL", "return ARGV[1]", 0, "hello"]);
```

### Lua Scripts

```ts
import { sendCommand } from "https://deno.land/x/r2d2/mod.ts";

const redisConn = await Deno.connect({ port: 6379 });

// Returns "mylib"
await sendCommand(redisConn, [
  "FUNCTION",
  "LOAD",
  "#!lua name=mylib\nredis.register_function('knockknock', function() return 'Who\\'s there?' end)",
]);

// Returns "Who's there?"
await sendCommand(redisConn, ["FCALL", "knockknock", 0]);
```

### Timeouts

```ts
import { deadline } from "https://deno.land/std/async/mod.ts";
import { sendCommand } from "https://deno.land/x/r2d2/mod.ts";

const redisConn = await Deno.connect({ port: 6379 });

// Rejects if the command takes longer than 100 ms
await deadline(sendCommand(redisConn, ["SLOWLOG", "GET"]), 100);
```

## Contributing

Before submitting a pull request, please run:

1. `deno fmt`
2. `deno lint`
3. `deno task redis:start && deno task test` and ensure all tests pass
4. `deno task redis:start && deno task bench` and ensure performance hasn't
   degraded

> Note: Redis must be installed on your local machine. For installation
> instructions, see [here](https://redis.io/docs/getting-started/installation/).

## Comparison

Data recorded on November 28, 2022.

### Benchmarks

```
cpu: Apple M2
runtime: deno 1.28.2 (aarch64-apple-darwin)

benchmark        time (avg)             (min … max)       p75       p99      p995
--------------------------------------------------- -----------------------------
r2d2         173.32 µs/iter    (143.42 µs … 6.4 ms) 173.62 µs 326.04 µs 355.67 µs
deno-redis    238.4 µs/iter   (190.46 µs … 6.05 ms) 235.92 µs 436.75 µs 634.58 µs
npm:ioredis  350.98 µs/iter   (206.71 µs … 4.17 ms) 262.25 µs   3.29 ms   3.65 ms
npm:redis       518 µs/iter   (302.21 µs … 4.33 ms) 379.25 µs   3.33 ms   3.65 ms

summary
  r2d2
   1.38x faster than deno-redis
   2.03x faster than npm:ioredis
   2.99x faster than npm:redis
```

> Node: Results were produced using `deno task redis:start && deno task bench`.

### Size

| Module      | Size (KB) | Dependencies |
| ----------- | --------- | ------------ |
| r2d2        | 77.94     | 8            |
| deno-redis  | 187.76    | 25           |
| npm:ioredis | 890.96    | 10           |
| npm:redis   | 891.60    | 9            |

> Note: Results were produced using `deno info <module>`
