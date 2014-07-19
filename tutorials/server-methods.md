# Server Methods
## Server methods

Server methods are a useful way of sharing functions by attaching them to your server object rather than requiring a common module everywhere it is needed. To register a server method, you need access to either the `server` object or the `plugin` object. Two different forms are available, either passing separate parameters:

```javascript
var add = function (x, y, next) {
    next(null, x + y);
};

server.method('add', add, {});
```

Or an object with `name`, `fn`, and `options` parameters (note that you may also pass an array of these objects):

```javascript
var add = function (x, y, next) {
    next(null, x + y);
};

server.method({
    name: 'add',
    fn: add,
    options: {}
});
```

## Name

The `name` parameter is a string used to retrieve the method from the server later, via `server.methods[name]`. Note that if you specify a `name` with a `.` character, it is registered as a nested object rather than the literal string. As in:

```javascript
server.method('math.add', add);
```

becomes accessible as server.methods.math.add

## Function

The `fn` parameter is the actual function to call when the method is invoked. It can take any number of arguments but *must* accept a callback as its last parameter. The callback accepts three parameters: `err`, `result`, and `ttl`. The first two parameters are err-first callbacks. If an error occurs in your method, pass it as the first argument, otherwise the first argument should be undefined or null and the return value as the second argument. The `ttl` argument is used to tell hapi how long the return value can be cached; if it is specified as `0` then the value will never be cached.

## Caching

Speaking of caching, another major advantage of server methods is that they may leverage hapi's native caching. The default is to not cache, however if a valid configuration is passed when registering the method, the return value will be cached and served from there instead of re-running your method every time it is called. The configuration looks like the following:

```javascript
server.method('add', add, {
    cache: {
        expiresIn: 60000,
        expiresAt: '30:22',
        staleIn: 30000,
        staleTimeout: 10000,
        // segment: '##math.add',
        // cache: 'name',
    }
});
```

The parameters mean:

* `expiresIn`: milliseconds since the item was created to keep in cache
* `expiresAt`: MM:HH notation for a specific time to invalidate the cache, this cannot be used at the same time as expiresIn
* `staleIn`: milliseconds to wait before a cached item is marked stale, this must be *less* than expiresIn
* `staleTimeout`: milliseconds to wait for a response before serving a stale value
* `segment`: an optional segment name used to isolate cache items, if specified this *must* begin with `'##'`
* `cache`: an optional string with the name of the cache connection configured on your server to use

More information on the caching options can be found in the [API Reference](/api#servermethodmethod) as well as the documentation for [catbox](https://github.com/spumko/catbox#policy).

## Generate a custom key

In addition to the above options, you may also pass a custom function used to generate a key based on the parameters passed to your method. If your method only accepts some combination of string, number, and boolean values hapi will generate a sane key for you. However, if your method accepts an object parameter, you should specify a function that will generate a key similar to the following:

```javascript
var sum = function (array, next) {
    var sum = 0;

    array.forEach(function (item) {
        sum += item;
    });

    next(null, sum);
};

server.method('sum', sum, {
    generateKey: function (array) {
        return array.join(',');
    }
});
```

Any arguments that are passed to your method are available to the generateKey method, including the callback though it is important to *not* use the callback while calculating your key. If the `generateKey` method returns `null` (which should be the case if a key is unable to be generated) then the result of the server method will never be cached, regardless of its return value and `ttl`.

## Bind

The last option available to server methods is `bind`. The `bind` option changes the `this` context within the method. It defaults to `null` when added by server, or to the same object as `plugin.bind` if added by plugin. This can be useful for passing in a database client without needing to pass it as a parameter and requiring a custom `generateKey` function, as in:

```javascript
var lookup = function (id, next) {
    this.getOne({ id: id }, function (err, value) {
        next(err, value);
    });
};

server.method('lookup', lookup);
```
