# clue/buzz-react [![Build Status](https://travis-ci.org/clue/php-buzz-react.svg?branch=master)](https://travis-ci.org/clue/php-buzz-react)

Simple, async HTTP client for concurrently processing any number of HTTP requests,
built on top of [React PHP](http://reactphp.org/).

This library is heavily inspired by the great
[kriswallsmith/Buzz](https://github.com/kriswallsmith/Buzz)
project. However, instead of blocking on each request, it relies on
[React PHP's EventLoop](https://github.com/reactphp/event-loop) to process
multiple requests in parallel.
This allows you to interact with multiple HTTP servers
(fetch URLs, talk to RESTful APIs, follow redirects etc.)
at the same time.
Unlike the underlying [react/http-client](https://github.com/reactphp/http-client),
this project aims at providing a higher-level API that is easy to use
in order to process multiple HTTP requests concurrently without having to
mess with most of the low-level details.

* **Async execution of HTTP requests** -
  Send any number of HTTP requests to any number of HTTP servers in parallel and
  process their responses as soon as results come in.
  The Promise-based design provides a *sane* interface to working with out of bound responses.
* **Lightweight, SOLID design** -
  Provides a thin abstraction that is [*just good enough*](http://en.wikipedia.org/wiki/Principle_of_good_enough)
  and does not get in your way.
  Builds on top of well-tested components and well-established concepts instead of reinventing the wheel.
* **Good test coverage** -
  Comes with an automated tests suite and is regularly tested in the *real world*

**Table of contents**

* [Quickstart example](#quickstart-example)
* [Usage](#usage)
  * [Browser](#browser)
    * [Methods](#methods)
    * [Promises](#promises)
    * [Blocking](#blocking)
    * [submit()](#submit)
    * [send()](#send)
    * [withOptions()](#withoptions)
    * [withSender()](#withsender)
    * [withBase()](#withbase)
    * [withoutBase()](#withoutbase)
  * [ResponseInterface](#responseinterface)
  * [RequestInterface](#requestinterface)
  * [UriInterface](#uriinterface)
  * [ResponseException](#responseexception)
* [Advanced](#advanced)
  * [Sender](#sender)
  * [DNS](#dns)
  * [Connection options](#connection-options)
  * [SOCKS proxy](#socks-proxy)
  * [UNIX domain sockets](#unix-domain-sockets)
  * [Options](#options)
  * [Streaming](#streaming)
* [Install](#install)
* [License](#license)

## Quickstart example

Once [installed](#install), you can use the following code to access a
HTTP webserver and send some simple HTTP GET requests:

```php
$loop = React\EventLoop\Factory::create();
$client = new Browser($loop);

$client->get('http://www.google.com/')->then(function (ResponseInterface $response) {
    var_dump($response->getHeaders(), (string)$response->getBody());
});

$loop->run();
```

See also the [examples](examples).

## Usage

### Browser

The `Browser` is responsible for sending HTTP requests to your HTTP server
and keeps track of pending incoming HTTP responses.
It also registers everything with the main [`EventLoop`](https://github.com/reactphp/event-loop#usage).

```php
$loop = React\EventLoop\Factory::create();
$browser = new Browser($loop);
```

If you need custom DNS or proxy settings, you can explicitly pass a
custom [`Sender`](#sender) instance. This is considered *advanced usage*.

#### Methods

The `Browser` offers several methods that resemble the HTTP protocol methods:

```php
$browser->get($url, array $headers = array());
$browser->head($url, array $headers = array());
$browser->post($url, array $headers = array(), $content = '');
$browser->delete($url, array $headers = array(), $content = '');
$browser->put($url, array $headers = array(), $content = '');
$browser->patch($url, array $headers = array(), $content = '');
```

If you need a custom HTTP protocol method, you can use the [`send()`](#send) method.

Each of the above methods supports async operation and either *resolves* with a [`ResponseInterface`](#responseinterface) or
*rejects* with an `Exception`.
Please see the following chapter about [promises](#promises) for more details.

#### Promises

Sending requests is async (non-blocking), so you can actually send multiple requests in parallel.
The `Browser` will respond to each request with a [`ResponseInterface`](#responseinterface) message, the order is not guaranteed.
Sending requests uses a [Promise](https://github.com/reactphp/promise)-based interface that makes it easy to react to when a transaction is fulfilled (i.e. either successfully resolved or rejected with an error):

```php
$browser->get($url)->then(
    function (ResponseInterface $response) {
        var_dump('Response received', $response);
    },
    function (Exception $error) {
        var_dump('There was an error', $error->getMessage());
    }
});
```

If this looks strange to you, you can also use the more traditional [blocking API](#blocking).

#### Blocking

As stated above, this library provides you a powerful, async API by default.

If, however, you want to integrate this into your traditional, blocking environment,
you should look into also using [clue/block-react](https://github.com/clue/php-block-react).

The resulting blocking code could look something like this:

```php
use Clue\React\Block;

$loop = React\EventLoop\Factory::create();
$browser = new Clue\React\Buzz\Browser($loop);

$promise = $browser->get('http://example.com/');

try {
    $response = Block\await($promise, $loop);
    // response successfully received
} catch (Exception $e) {
    // an error occured while performing the request
}
```

Similarly, you can also process multiple requests concurrently and await an array of `Response` objects:

```php
$promises = array(
    $browser->get('http://example.com/'),
    $browser->get('http://www.example.org/'),
);

$responses = Block\awaitAll($promises, $loop);
```

Please refer to [clue/block-react](https://github.com/clue/php-block-react#readme) for more details.

#### submit()

The `submit($url, array $fields, $headers = array(), $method = 'POST')` method can be used to submit an array of field values similar to submitting a form (`application/x-www-form-urlencoded`).

#### send()

The `send(RequestInterface $request)` method can be used to send an arbitrary
instance implementing the [`RequestInterface`](#requestinterface) (PSR-7).

#### withOptions()

The `withOptions(array $options)` method can be used to change the [options](#options) to use:

```php
$newBrowser = $browser->withOptions($options);
```

Notice that the [`Browser`](#browser) is an immutable object, i.e. the `withOptions()` method
actually returns a *new* [`Browser`](#browser) instance with the [options](#options) applied.

See [options](#options) for more details.

#### withSender()

The `withSender(Sender $sender)` method can be used to change the [`Sender`](#sender) instance to use:

```php
$newBrowser = $browser->withSender($sender);
```

Notice that the [`Browser`](#browser) is an immutable object, i.e. the `withSender()` method
actually returns a *new* [`Browser`](#browser) instance with the given [`Sender`](#sender) applied.

See [`Sender`](#sender) for more details.

#### withBase()

The `withBase($baseUri)` method can be used to change the base URI used to
resolve relative URIs to.

```php
$newBrowser = $browser->withBase('http://api.example.com/v3');
```

Notice that the [`Browser`](#browser) is an immutable object, i.e. the `withBase()` method
actually returns a *new* [`Browser`](#browser) instance with the given base URI applied.

Any requests to relative URIs will then be processed by first prepending the
base URI.
Please note that this merely prepends the base URI and does *not* resolve any
relative path references (like `../` etc.).
This is mostly useful for API calls where all endpoints (URIs) are located
under a common base URI scheme.

```php
// will request http://api.example.com/v3/example
$newBrowser->get('/example')->then(…);
```

#### withoutBase()

The `withoutBase()` method can be used to remove the base URI.

```php
$newBrowser = $browser->withoutBase();
```

Notice that the [`Browser`](#browser) is an immutable object, i.e. the `withoutBase()` method
actually returns a *new* [`Browser`](#browser) instance without any base URI applied.

See also [`withBase()`](#withbase).

### ResponseInterface

The `Psr\Http\Message\ResponseInterface` represents the incoming response received from the [`Browser`](#browser).

This is a standard interface defined in [PSR-7: HTTP message interfaces]
(http://www.php-fig.org/psr/psr-7/), see its [`ResponseInterface` definition]
(http://www.php-fig.org/psr/psr-7/#3-3-psr-http-message-responseinterface)
which in turn extends the [`MessageInterface` definition]
(http://www.php-fig.org/psr/psr-7/#3-1-psr-http-message-messageinterface).

### RequestInterface

The `Psr\Http\Message\RequestInterface` represents the outgoing request to be sent via the [`Browser`](#browser).

This is a standard interface defined in [PSR-7: HTTP message interfaces]
(http://www.php-fig.org/psr/psr-7/), see its [`RequestInterface` definition]
(http://www.php-fig.org/psr/psr-7/#3-2-psr-http-message-requestinterface)
which in turn extends the [`MessageInterface` definition]
(http://www.php-fig.org/psr/psr-7/#3-1-psr-http-message-messageinterface).

### UriInterface

The `Psr\Http\Message\UriInterface` represents an absolute or relative URI (aka URL).

This is a standard interface defined in [PSR-7: HTTP message interfaces]
(http://www.php-fig.org/psr/psr-7/), see its [`UriInterface` definition]
(http://www.php-fig.org/psr/psr-7/#3-5-psr-http-message-uriinterface).

### ResponseException

The `ResponseException` is an `Exception` sub-class that will be used to reject
a request promise if the remote server returns a non-success status code
(anything but 2xx or 3xx).
You can control this behavior via the ["obeySuccessCode" option](#options).

The `getCode()` method can be used to return the HTTP response status code.

The `getResponse()` method can be used to access its underlying [`ResponseInteface`](#responseinterface) object.

## Advanced

### Sender

The `Sender` is responsible for passing the [`RequestInterface`](#requestinterface) objects to
the underlying [`HttpClient`](https://github.com/reactphp/http-client) library
and keeps track of its transmission and converts its reponses back to [`ResponseInterface`](#responseinterfae) objects.

It also registers everything with the main [`EventLoop`](https://github.com/reactphp/event-loop#usage)
and the default [`Connector`](https://github.com/reactphp/socket-client) and [DNS `Resolver`](https://github.com/reactphp/dns).

See also [`Browser::withSender()`](#withsender) for changing the `Sender` instance during runtime.

### DNS

The [`Sender`](#sender) is also responsible for creating the underlying TCP/IP
connection to the remote HTTP server and hence has to orchestrate DNS lookups.
By default, it uses a `Connector` instance which uses Google's public DNS servers
(`8.8.8.8`).

If you need custom DNS settings, you can explicitly create a [`Sender`](#sender) instance
with your DNS server address (or `React\Dns\Resolver` instance) like this:

```php
$dns = '127.0.0.1';
$sender = Sender::createFromLoopDns($loop, $dns);
$browser = $browser->withSender($sender);
```

See also [`Browser::withSender()`](#withsender) for more details.

### Connection options

If you need custom connector settings (DNS resolution, SSL/TLS parameters, timeouts etc.), you can explicitly pass a
custom instance of the [`ConnectorInterface`](https://github.com/reactphp/socket-client#connectorinterface).

The below examples assume you've installed the latest SocketClient version:

```bash
$ composer require react/socket-client:^0.5
```

You can optionally pass additional
[socket context options](http://php.net/manual/en/context.socket.php)
to the constructor like this:

```php
// use local DNS server
$dnsResolverFactory = new DnsFactory();
$resolver = $dnsResolverFactory->createCached('127.0.0.1', $loop);

// outgoing connections via interface 192.168.10.1
$tcp = new DnsConnector(
    new TcpConnector($loop, array('bindto' => '192.168.10.1:0')),
    $resolver
);

$sender = Sender::createFromLoopConnectors($loop, $tcp);
$browser = $browser->withSender($sender);
```

You can optionally pass additional
[SSL context options](http://php.net/manual/en/context.ssl.php)
to the constructor like this:

```php
$ssl = new SecureConnector($tcp, $loop, array(
    'verify_peer' => false,
    'verify_peer_name' => false
));

$sender = Sender::createFromLoopConnectors($loop, $tcp, $ssl);
$browser = $browser->withSender($sender);
```

### SOCKS proxy

You can also establish your outgoing connections through a SOCKS proxy server
by adding a dependency to [clue/socks-react](https://github.com/clue/php-socks-react).

The SOCKS protocol operates at the TCP/IP layer and thus requires minimal effort at the HTTP application layer.
This works for both plain HTTP and SSL encrypted HTTPS requests.

See also the [SOCKS example](examples/socks).

### UNIX domain sockets

This library also supports connecting to a local UNIX domain socket path.
You have to explicitly create a [`Sender`](#sender) that passes every request through the
given UNIX domain socket.
For consistency reasons you still have to pass full HTTP URLs for every request,
but the host and port will be ignored when establishing a connection.

```php
$path = 'unix:///tmp/daemon.sock';
$sender = Sender::createFromLoopUnix($loop, $path);
$client = new Browser($loop, $sender);

$client->get('http://localhost/demo');
```

### Options

Note: This API is subject to change.

The [`Browser`](#browser) class exposes several options for the handling of
HTTP transactions. These options resemble some of PHP's
[HTTP context options](http://php.net/manual/en/context.http.php) and
can be controlled via the following API (and their defaults):

```php
$newBrowser = $browser->withOptions(array(
    'followRedirects' => true,
    'maxRedirects' => 10,
    'obeySuccessCode' => true
));
```

Notice that the [`Browser`](#browser) is an immutable object, i.e. the `withOptions()` method
actually returns a *new* [`Browser`](#browser) instance with the options applied.

### Streaming

Note: This API is subject to change.

The [`Sender`](#sender) emits a `progress` event array on its `Promise` that can be used
to intercept the underlying outgoing request stream (`React\HttpClient\Request` in the `requestStream` key)
and the incoming response stream (`React\HttpClient\Response` in the `responseStream` key).

```php
$client->get('http://www.google.com/')->then($handler, null, function ($event) {
    if (isset($event['responseStream'])) {
        /* @var $stream React\HttpClient\Response */
        $stream = $event['responseStream'];
        $stream->on('data', function ($data) { });
    }
});
```

## Install

The recommended way to install this library is [through Composer](http://getcomposer.org).
[New to Composer?](http://getcomposer.org/doc/00-intro.md)

This will install the latest supported version:

```bash
$ composer require clue/buzz-react:^0.4
```

See also the [CHANGELOG](CHANGELOG.md) for details about version upgrades.

## License

MIT
