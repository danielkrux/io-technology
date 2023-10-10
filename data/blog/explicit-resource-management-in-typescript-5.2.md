---
title: 'ECMAScript Explicit Resource Management early implementation in TypeScript 5.2'
date: '2023-09-08'
tags: ['typescript', 'web-development', 'javascript']
summary: 'The TypeScript implementation of the `using` and `await using` declarations from the TC39 Explicit Resource Management proposal, which is currently at Stage 3'
images:
  [
    '/articles/explicit-resource-management-in-typescript-5.2/explicit-resource-management-in-typescript-5.2.webp.webp',
  ]
authors: ['mohi-bagherani']
theme: 'blue'
canonicalUrl: 'https://medium.com/@bagherani/ecmascript-explicit-resource-management-early-implementation-in-typescript-5-2-5e4d08b2aee3'
---

# ECMAScript Explicit Resource Management early implementation in TypeScript 5.2

This article on "ECMAScript Explicit Resource Management Early Implementation in TypeScript 5.2" is for developers who are interested in managing resources explicitly in their JavaScript programs. It explains how this feature works and how it can simplify resource management code.

Explicit Resource Management indicates a system whereby the lifetime of a “resource” is managed explicitly by the user. This can be done either imperatively (by directly calling a method like `Symbol.dispose`) or declaratively (through a block-scoped declaration like `using`).

A "Resource" is something that has a specific lifespan. When that lifespan is over, the program needs to perform a specific action, like closing a file or freeing up memory, so that the program can continue running smoothly. Examples of resources include file handles and network sockets.

At the moment of writing, the ECMAScript [proposal](https://github.com/tc39/proposal-explicit-resource-management) for Explicit Resource Management is at [stage 3](https://tc39.es/process-document/) which means it will be launched within a few months. However, the TypeScript community has already implemented it in TypeScript [version 5.2](https://devblogs.microsoft.com/typescript/announcing-typescript-5-2-beta/#using-declarations-and-explicit-resource-management).

This implementation will add a feature to the JavaScript language similar to other languages such as C# `using` syntax [[link](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/statements#1314-the-using-statement)], Python `with` syntax [[link](https://docs.python.org/3/reference/compound_stmts.html#the-with-statement)], or `try` [[link](https://docs.oracle.com/javase/tutorial/essential/exceptions/tryResourceClose.html)] in Java.

```csharp
// C# example of the 'using' statement
using (var file = File.OpenRead(“path-to-file”))
{
    // use the file
}
// file.dispose() method has been called now to release resources.
```

In general, without using this new feature, a conventional free-up pattern is by using the `try/finally` syntax:

```javascript
var obj
try {
  obj = someResource()
  // ...
} finally {
  obj.release() // or any other clean-up method provided by the resource
}
```

Now this can be written in this way in TypeScript 5.2:

```javascript
using obj = someResource();
// ...
```

When the program's execution goes outside of the block that the `obj` has been defined, the object will no longer be needed and the `dispose` method will be called.

TypeScript introduced an interface called `Disposable` that can be used to implement the `using` statement. To do this, your class should implement the `Disposable` interface, which requires the class to have a `[Symbol.dispose]` method. This method is responsible for freeing up any resources used by the class when it is no longer needed. By implementing the `Disposable` interface, the TypeScript compiler knows how to translate the `using` syntax into a `try/finally` block that the current JavaScript engine can understand.

```typescript
class MyResource implements Disposable {
    [Symbol.dispose]() {
      // clean-up logic
    }
}

using obj = new MyResource();
// ...
```

Or, if you have a **function** instead of a **class**, your function should return an object that has the `Symbol.dispose` method:

```typescript
function myResource(): Disposable {
    return {
        [Symbol.dispose](){ /* clean-up logic */}
    }
}

using obj = myResource();
```

Good to know that after compilation, the above code will become something like this:

```javascript
class MyResource {
  [Symbol.dispose]() {
    // clean-up logic
  }
}
var obj
const env_1 = { stack: [], error: void 0, hasError: false }
try {
  obj = __addDisposableResource(env_1, MyResource(), false)
} catch (e_1) {
  env_1.error = e_1
  env_1.hasError = true
} finally {
  __disposeResources(env_1)
}
```

As you can see here the TypeScript compiler has compiled the `using` statement into the traditional `try/finally` pattern that I mentioned before and of course, with some simple functions to manage the allocated resources(`__addDisposableResource` and `__disposeResources` will be generated by the compiler in the output file).

### Nested scopes and using

Objects can be created in the nested scopes:

```typescript
function work() {
    using a = resource();
    {
        using b = resource();
    }
}

work();
// b.dispose()
// a.dispose()
```

In this case, after leaving the scope, the `dispose` methods will be called respectively.

### DisposableStack class

Implementing the `Symbol.dispose` method can be a good pattern to use when you have a class or function that needs to be disposed of at some point. However, there are a few potential problems to consider. Firstly, implementing the `dispose` method can add unnecessary abstraction to your code in some cases, which can make it more difficult to read and understand. Secondly, many existing libraries and modules may not have implemented the `dispose` method yet, in which case you will need to create a wrapper around them and implement the clean-up code (dispose) yourself. Keep these considerations in mind when deciding whether or not to implement the `dispose` method in your code.

Here is an example from the TypeScript blog:

```typescript
class TempFile implements Disposable {
    #path: string;
    #handle: number;

    constructor(path: string) {
        this.#path = path;
        this.#handle = fs.openSync(path, "w+");
    }

    // other methods

    [Symbol.dispose]() {
        // Close the file and delete it.
        fs.closeSync(this.#handle);
        fs.unlinkSync(this.#path);
    }
}

function doSomeWork() {
    using file = new TempFile("path-to-file");
}
```

This class has been implemented (with all of its sophistication that it might have later) to only have a `Symbol.dispose` method that would be called by the engine. However, TypeScript 5.2 introduces the `DisposableStack` class, which makes it simpler to use Explicit Resource Management.

Here is how:

```typescript
function doSomeWork() {
    const path = "path-to-file";
    const file = fs.openSync(path, "w+");

    using cleanup = new DisposableStack();
    cleanup.defer(() => {
        fs.closeSync(file);
        fs.unlinkSync(path);
    });

    // use file...
}
```

Here we’ve created an instance of `DisposableStack` right after opening the file; It has the defer method that accepts a callback function which is suitable for clean-up that will be invoked at the moment of disposing the `cleanup` object.  
After the above code is compiled, the resulting JavaScript code will include a `finally` block. This `finally` block will call the callback function that was passed to the `defer` method. This ensures that the callback function is always executed, even if an error occurs in the try block.

### Async dispose method

Sometimes for your **dispose** logic you might need to use asynchronous operations using `async/await`. In this case, you can simply use the `await` before the `using` keyword:

```typescript
await using file = OpenFile('...');
```

In this case, in the `OpenFile` function or class, you should implement the dispose method using `Symbol.asyncDispose` like below:

```typescript
import * as fs from 'fs'

function OpenFile(path): AsyncDisposable {
  const file = fs.open(path)

  return {
    file,
    async [Symbol.asyncDispose]() {
      await fs.anAsyncOperation()
    },
  }
}
```

### Polyfills

Because this feature is so recent, most runtimes will not support it natively. To use it, you will need runtime polyfills for the following:

- Symbol.dispose
- Symbol.asyncDispose
- DisposableStack
- AsyncDisposableStack
- SuppressedError

`Symbol.dispose` and `Symbol.asyncDispose` can be polyfilled like this:

```javascript
Symbol.dispose ??= Symbol('Symbol.dispose')
Symbol.asyncDispose ??= Symbol('Symbol.asyncDispose')
```

The compilation target in the `tsconfig` file should be es2022 or below and the library setting to either include `"esnext"` or `"esnext.disposable"`

```json
{
  "compilerOptions": {
    "target": "es2022",
    "lib": ["es2022", "esnext.disposable", "dom"]
  }
}
```

For more information on this feature, take a look at the work on [GitHub](https://github.com/microsoft/TypeScript/pull/54505)!