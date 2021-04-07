---
title: "The new Node.js console.log"
date: 2021-04-07T14:29:42-03:00
draft: false
---

Yesterday I was looking for something to justify the usage of [pino](https://github.com/pinojs/pino)
and [pino-pretty](https://github.com/pinojs/pino-pretty/) in some CLI tools.

I found some people over the internet (blogs/StackOverflow/YouTube) talking about the
performance degradation caused by `console.log` in a Node.js app.

In some cases makes no much sense to have a logger library/module for a CLI app, but since pino is suggested in the [Node.js reference architecture](https://github.com/nodeshift/nodejs-reference-architecture/blob/main/docs/operations/logging.md) I decided to take a look.

After some attempts to suppress the pid, hostname, and timestamp from the output, I found this issue
[add support for ignore: *](https://github.com/pinojs/pino-pretty/issues/82) and I started to think
"Why am I doing this?"

So I decided to revisit the `console.log vs process.stdout.write` over the internet to see what happened
overtime...

One of the first results via Google > StackOverflow is: [difference-between-process-stdout-write-and-console-log-in-node-js](https://stackoverflow.com/questions/4976466/difference-between-process-stdout-write-and-console-log-in-node-js)...

But actually, something bigger and new has happened (and maybe still happening) to `console.log`. If you take a look
on the 7.x branch, you still can see the basic version:


https://github.com/nodejs/node/blob/v7.x/lib/console.js#L43
```
Console.prototype.log = function log(...args) {
  this._stdout.write(`${util.format.apply(null, args)}\n`);
};
```

In the 8.x branch we can see the start of the evolution:

https://github.com/nodejs/node/blob/v8.x/lib/console.js#L127
```
Console.prototype.log = function log(...args) {
  write(this._ignoreErrors,
        this._stdout,
        util.format.apply(null, args),
        this._stdoutErrorHandler,
        this[kGroupIndent]);
};
```

I'm not going to put all the Node.js code here but something is happening!

At the 11.x branch (I'm not sure if that happened before 11.x anyway) the console was moved to another directory:

https://github.com/nodejs/node/blob/v11.x/lib/internal/console/constructor.js

Not only the code has changed but also it is following a standard: https://console.spec.whatwg.org/

The console.`log` now calls another thing:

```
  log(...args) {
    this[kWriteToConsole](kUseStdout, this[kFormatForStdout](args));
  },
```

This is the code for the 16.x branch related to the `KWriteToConsole: 

```
  [kWriteToConsole]: {
    ...consolePropAttributes,
    value: function(streamSymbol, string) {
      const ignoreErrors = this._ignoreErrors;
      const groupIndent = this[kGroupIndent];

      const useStdout = streamSymbol === kUseStdout;
      const stream = useStdout ? this._stdout : this._stderr;
      const errorHandler = useStdout ?
        this._stdoutErrorHandler : this._stderrErrorHandler;

      if (groupIndent.length !== 0) {
        if (StringPrototypeIncludes(string, '\n')) {
          string = StringPrototypeReplace(string, /\n/g, `\n${groupIndent}`);
        }
        string = groupIndent + string;
      }
      string += '\n';

      if (ignoreErrors === false) return stream.write(string);

      // There may be an error occurring synchronously (e.g. for files or TTYs
      // on POSIX systems) or asynchronously (e.g. pipes on POSIX systems), so
      // handle both situations.
      try {
        // Add and later remove a noop error handler to catch synchronous
        // errors.
        if (stream.listenerCount('error') === 0)
          stream.once('error', noop);

        stream.write(string, errorHandler);
      } catch (e) {
        // Console is a debugging utility, so it swallowing errors is not
        // desirable even in edge cases such as low stack space.
        if (isStackOverflowError(e))
          throw e;
        // Sorry, there's no proper way to pass along the error here.
      } finally {
        stream.removeListener('error', noop);
      }
    }
  },
  ```

Yeah, a lot different from the previous.

```
this._stdout.write(`${util.format.apply(null, args)}\n`);
```

But for me, the most interesting part is that it seems to be alive, like the "living standard" which it is implementing.

```
What does "Living Standard" mean?
The WHATWG standards are described as Living Standards. This means that they are standards that are continuously updated as they receive feedback, either from web developers, browser vendors, tool vendors, or indeed any other interested party. It also means that new features get added to them over time, at a rate intended to keep the standard a little ahead of the implementations, but not so far ahead that the implementations give up.
```

https://whatwg.org/faq#living-standard

Conclusion (for me at least)

1) Depending on the technology (in this case Node.js) the best source is the source code (and/or comments on issues).

2) Is good to have a day in a month (?) or maybe a weekend in a semester to catch up the new things because some statements
get easily false overtime on internet.
