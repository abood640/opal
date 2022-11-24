# Asynchronous code (PromiseV2 / async / await)

## PromiseV2

In Opal 1.2 we introduced PromiseV2 which is to replace the default Promise in Opal 2.0
(which will become PromiseV1). Right now it's experimental, but the interface of PromiseV1
stay unchanged and will continue to be supported.

It is imperative that during the transition period you either `require 'promise/v1'` or
`require 'promise/v2'` and then use either `PromiseV1` or `PromiseV2`.

If you write library code it's imperative that you don't require the promise itself, but
detect if `PromiseV2` is defined and use the newer implementation, for instance using the
following code:

```ruby
module MyLibrary
  Promise = defined?(PromiseV2) ? PromiseV2 : ::Promise
end
```

The difference between `PromiseV1` and `PromiseV2` is that `PromiseV1` is a pure-Ruby
implementation of a Promise, while `PromiseV2` is reusing the JavaScript `Promise`. Both are
incompatible with each other, but `PromiseV2` can be awaited (see below) and they translate
1 to 1 to the JavaScript native `Promise` (they are bridged; you can directly return a
`Promise` from JavaScript API without a need to translate it). The other difference is that
`PromiseV2` always runs a `#then` block a tick later, while `PromiseV1` would could run it
instantaneously.

## Async/await

In Opal 1.3 we implemented the CoffeeScript pattern of async/await. As of now, it's hidden
behind a magic comment, but this behavior may change in the future.

Example:

```ruby
# await: true

require "await"

def wait_5_seconds
  puts "Let's wait 5 seconds..."
  sleep(5).__await__
  puts "Done!"
end

wait_5_seconds.__await__
```

It's important to understand what happens under the hood: every scope in which `#__await__` is
encountered will become async, which means that it will return a PromiseV2 that will resolve
to a value. This includes methods, blocks and the top scope. This means, that `#__await__` is
infectious and you need to remember to `#__await__` everything along the way, otherwise
a program will finish too early and the values may be incorrect.

It is certainly correct to `#__await__` any value, including non-Promises, for instance
`5.__await__` will correctly resolve to `5` (except that it will make the scope an async
function, with all the limitations described above).

The `await` stdlib module includes a few useful functions, like async-aware `each_await`
function and `sleep` that doesn't block the thread. It also includes a method `#await`
which is an alias of `#itself` - it makes sense to auto-await that method.

[You can take a look at how we ported Minitest to support asynchronous tests.](https://github.com/opal/opal/pull/2221/files#diff-bdc8868ad4476bff7b25475f1b19059ac684d64c6531a645b3ba0aef0c466d0f).

This approach is certainly incompatible with what Ruby does, but due to a dynamic nature
of Ruby and a different model of JavaScript this was the least invasive way to catch up
with the latest JavaScript trends and support `Promise` heavy APIs and asynchronous code.

## Auto-await

The magic comment also accepts a comma-separated list of methods to be automatically
awaited. An individual value can contain a wildcard character `*`. For instance,
those two blocks of code are equivalent:

```ruby
# await: true

require "await"

[1,2,3].each_await do |i|
  p i
  sleep(i).__await__
end.__await__
```

```ruby
# await: sleep, *await*

require "await"

[1,2,3].each_await do |i|
  p i
  sleep i
end
```
