# request-caching

An HTTP client with caching, built on top of [request](https://github.com/mikeal/request).

## Features

* Drop-in replacement for `request`.
* Redis or In-memory (LRU) storage. Easy to build new storage backends.
* Supports both [public and private caching](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.9.1).
* Takes advantage of the `ETag` response header by using the `If-None-Match` request header.
* Takes advantage of the `Last-Modified` response header by using the `If-Modified-Since` request header.
* Cache TTL is based on the `Expires` response header or `max-age` value in `Cache-Control`, but can be overridden.
* Highly customizeable with sensible defaults.

## Examples

Below are some common use cases:

### In-memory cache

```javascript
var LRU = require('lru-cache');
var store = new request.MemoryStore(new LRU());
var cache = new request.Cache(store);

request('http://some.url', {cache: cache}, function(err, res, body) {
  
});
```

### Redis cache

```javascript
var redis = require('redis').createClient();
var store = new request.RedisStore(redis);
var cache = new request.Cache(store);

request('http://some.url', {cache: cache}, function(err, res, body) {
  
});
```

### Cache flushing

You should always provide a `prefix` the key with the name of your app (or API). When you invoke `Cache.flush`, it will flush *all* keys starting with that prefix. If you don't specify a prefix, you'll flush the entire cache.

```javascript
var prefix = 'yourapp:';
var cache = new request.Cache(store, prefix);
```

### Private caching

Some HTTP responses should be cached privately - i.e. they shouldn't be available for other users.
This is the case when the server responds with `Cache-Control: private`.

To handle this you should construct the `Cache` with a `privateSuffix` String argument. This suffix will be appended to the key when caching a private response.

```javascript
var prefix = 'yourapp:';
var unique = req.currentUser._id; // or req.cookies['connect.sid']
var privateSuffix = 'private:' + unique;
var cache = new request.Cache(store, prefix, privateSuffix);

request('http://some.url', {cache: cache}, function(err, res, body) {
  
});
```

You will get an error if no `privateSuffix` was provided when caching a private response.

### Custom TTL

```javascript
var store = new request.RedisStore(redis);

function myTtl(ttl) {
  return ttl * 1000; // cache it longer than the server told us to.
}

var cache = new request.Cache(store, null, null, myTtl);

request('http://some.url', {cache: cache}, function(err, res, body) {
  
});
```

## How it works

* All cacheable responses are cached, even if they are expired.
* The TTL for a cached entry uses the TTL from the response, but can be overridden.
* If a cached response is not expired, returns it.
* If a cached response is expired, issue request with `If-None-Match` value from cached response's `ETag`.
  * If response is 304 (Not Modified), returned cached response.
* Cacheable responses marked as private are cached with a private cache key.
* Cache lookups look in public cache first, and then in the private cache.
