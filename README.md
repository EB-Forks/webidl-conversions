# Web IDL Type Conversions on JavaScript Values

This package implements, in JavaScript, the algorithms to convert a given JavaScript value according to a given [Web IDL](http://heycam.github.io/webidl/) [type](http://heycam.github.io/webidl/#idl-types).

The goal is that you should be able to write code like

```js
"use strict";
const conversions = require("webidl-conversions");

function doStuff(x, y) {
    x = conversions["boolean"](x);
    y = conversions["unsigned long"](y);
    // actual algorithm code here
}
```

and your function `doStuff` will behave the same as a Web IDL operation declared as

```webidl
void doStuff(boolean x, unsigned long y);
```

## API

This package's main module's default export is an object with a variety of methods, each corresponding to a different Web IDL type. Each method, when invoked on a JavaScript value, will give back the new JavaScript value that results after passing through the Web IDL conversion rules. (See below for more details on what that means.) Alternately, the method could throw an error, if the Web IDL algorithm is specified to do so: for example `conversions["float"](NaN)` [will throw a `TypeError`](http://heycam.github.io/webidl/#es-float).

## Conversions implemented

Conversions for all of the basic types from the Web IDL specification are implemented:

- [`any`](https://heycam.github.io/webidl/#es-any)
- [`void`](https://heycam.github.io/webidl/#es-void)
- [`boolean`](https://heycam.github.io/webidl/#es-boolean)
- [Integer types](https://heycam.github.io/webidl/#es-integer-types), which can additionally be provided the boolean options `{ clamp, enforceRange }` as a second parameter
- [`float`](https://heycam.github.io/webidl/#es-float), [`unrestricted float`](https://heycam.github.io/webidl/#es-unrestricted-float)
- [`double`](https://heycam.github.io/webidl/#es-double), [`unrestricted double`](https://heycam.github.io/webidl/#es-unrestricted-double)
- [`DOMString`](https://heycam.github.io/webidl/#es-DOMString), which can additionally be provided the boolean option `{ treatNullAsEmptyString }` as a second parameter
- [`ByteString`](https://heycam.github.io/webidl/#es-ByteString), [`USVString`](https://heycam.github.io/webidl/#es-USVString)
- [`object`](https://heycam.github.io/webidl/#es-object)
- [`Error`](https://heycam.github.io/webidl/#es-Error)
- [Buffer source types](https://heycam.github.io/webidl/#es-buffer-source-types)

Additionally, for convenience, the following derived type definitions are implemented:

- [`ArrayBufferView`](https://heycam.github.io/webidl/#ArrayBufferView)
- [`BufferSource`](https://heycam.github.io/webidl/#BufferSource)
- [`DOMTimeStamp`](https://heycam.github.io/webidl/#DOMTimeStamp)
- [`Function`](https://heycam.github.io/webidl/#Function)
- [`VoidFunction`](https://heycam.github.io/webidl/#VoidFunction) (although it will not censor the return type)

Derived types, such as nullable types, promise types, sequences, records, etc. are not handled by this library. You may wish to investigate the [webidl2js](https://github.com/jsdom/webidl2js) project.

### A note on the `long long` types

The `long long` and `unsigned long long` Web IDL types can hold values that cannot be stored in JavaScript numbers, so the conversion is imperfect. For example, converting the JavaScript number `18446744073709552000` to a Web IDL `long long` is supposed to produce the Web IDL value `-18446744073709551232`. Since we are representing our Web IDL values in JavaScript, we can't represent `-18446744073709551232`, so we instead the best we could do is `-18446744073709552000` as the output.

This library actually doesn't even get that far. Producing those results would require doing accurate modular arithmetic on 64-bit intermediate values, but JavaScript does not make this easy. We could pull in a big-integer library as a dependency, but in lieu of that, we for now have decided to just produce inaccurate results if you pass in numbers that are not strictly between `Number.MIN_SAFE_INTEGER` and `Number.MAX_SAFE_INTEGER`.

## Background

What's actually going on here, conceptually, is pretty weird. Let's try to explain.

Web IDL, as part of its madness-inducing design, has its own type system. When people write algorithms in web platform specs, they usually operate on Web IDL values, i.e. instances of Web IDL types. For example, if they were specifying the algorithm for our `doStuff` operation above, they would treat `x` as a Web IDL value of [Web IDL type `boolean`](http://heycam.github.io/webidl/#idl-boolean). Crucially, they would _not_ treat `x` as a JavaScript variable whose value is either the JavaScript `true` or `false`. They're instead working in a different type system altogether, with its own rules.

Separately from its type system, Web IDL defines a ["binding"](http://heycam.github.io/webidl/#ecmascript-binding) of the type system into JavaScript. This contains rules like: when you pass a JavaScript value to the JavaScript method that manifests a given Web IDL operation, how does that get converted into a Web IDL value? For example, a JavaScript `true` passed in the position of a Web IDL `boolean` argument becomes a Web IDL `true`. But, a JavaScript `true` passed in the position of a [Web IDL `unsigned long`](http://heycam.github.io/webidl/#idl-unsigned-long) becomes a Web IDL `1`. And so on.

Finally, we have the actual implementation code. This is usually C++, although these days [some smart people are using Rust](https://github.com/servo/servo). The implementation, of course, has its own type system. So when they implement the Web IDL algorithms, they don't actually use Web IDL values, since those aren't "real" outside of specs. Instead, implementations apply the Web IDL binding rules in such a way as to convert incoming JavaScript values into C++ values. For example, if code in the browser called `doStuff(true, true)`, then the implementation code would eventually receive a C++ `bool` containing `true` and a C++ `uint32_t` containing `1`.

The upside of all this is that implementations can abstract all the conversion logic away, letting Web IDL handle it, and focus on implementing the relevant methods in C++ with values of the correct type already provided. That is payoff of Web IDL, in a nutshell.

And getting to that payoff is the goal of _this_ project—but for JavaScript implementations, instead of C++ ones. That is, this library is designed to make it easier for JavaScript developers to write functions that behave like a given Web IDL operation. So conceptually, the conversion pipeline, which in its general form is JavaScript values ↦ Web IDL values ↦ implementation-language values, in this case becomes JavaScript values ↦ Web IDL values ↦ JavaScript values. And that intermediate step is where all the logic is performed: a JavaScript `true` becomes a Web IDL `1` in an unsigned long context, which then becomes a JavaScript `1`.

## Don't use this

Seriously, why would you ever use this? You really shouldn't. Web IDL is … strange, and you shouldn't be emulating its semantics. If you're looking for a generic argument-processing library, you should find one with better rules than those from Web IDL. In general, your JavaScript should not be trying to become more like Web IDL; if anything, we should fix Web IDL to make it more like JavaScript.

The _only_ people who should use this are those trying to create faithful implementations (or polyfills) of web platform interfaces defined in Web IDL. Its main consumer is the [jsdom](https://github.com/tmpvar/jsdom) project.
