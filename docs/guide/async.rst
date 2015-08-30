Asynchronous and non-Blocking I/O
---------------------------------

Real-time web features require a long-lived mostly-idle connection per
user.  In a traditional synchronous web server, this implies devoting
one thread to each user, which can be very expensive.``目前看golang用的go协程这么做的，刚刚看到说gevent和golang很像``

To minimize the cost of concurrent connections, Tornado uses a
single-threaded event loop.  This means that all application code
should aim to be asynchronous and non-blocking because only one
operation can be active at a time.

The terms asynchronous and non-blocking are closely related and are
often used interchangeably, but they are not quite the same thing.

Blocking
~~~~~~~~

A function **blocks** when it waits for something to happen before
returning.  A function may block for many reasons: network I/O, disk
I/O, mutexes, etc.  In fact, *every* function blocks, at least a
little bit, while it is running and using the CPU (for an extreme
example that demonstrates why CPU blocking must be taken as seriously
as other kinds of blocking, consider password hashing functions like
`bcrypt <http://bcrypt.sourceforge.net/>`_, which by design use
hundreds of milliseconds of CPU time, far more than a typical network
or disk access).

A function can be blocking in some respects and non-blocking in
others.  For example, `tornado.httpclient` in the default
configuration blocks on DNS resolution but not on other network access
(to mitigate this use `.ThreadedResolver` or a
``tornado.curl_httpclient`` with a properly-configured build of
``libcurl``).  In the context of Tornado we generally talk about
blocking in the context of network I/O, although all kinds of blocking
are to be minimized.``主要说网络阻塞``

Asynchronous
~~~~~~~~~~~~

An **asynchronous** function returns before it is finished, and
generally causes some work to happen in the background before
triggering some future action in the application (as opposed to normal
**synchronous** functions, which do everything they are going to do
before returning).  There are many styles of asynchronous interfaces:

* Callback argument
* Return a placeholder (`.Future`, ``Promise``, ``Deferred``)
* Deliver to a queue
* Callback registry (e.g. POSIX signals)

Regardless of which type of interface is used, asynchronous functions
*by definition* interact differently with their callers; there is no
free way to make a synchronous function asynchronous in a way that is
transparent to its callers (systems like `gevent
<http://www.gevent.org>`_ use lightweight threads to offer performance
comparable to asynchronous systems, but they do not actually make
things asynchronous).``不管使用那种异步方法，走不能和同步的一样一样的，必然多少不同表现``

``限于硬件资源：阻塞是绝对的，不阻塞是相对的；从响应时间上看：同步是绝对的，异步是相对的(部分的方面，部分的逻辑)``

Examples
~~~~~~~~

Here is a sample synchronous function:

.. testcode::

    from tornado.httpclient import HTTPClient

    def synchronous_fetch(url):
        http_client = HTTPClient()
        response = http_client.fetch(url)
        return response.body

.. testoutput::
   :hide:

And here is the same function rewritten to be asynchronous with a
callback argument:

.. testcode::

    from tornado.httpclient import AsyncHTTPClient

    def asynchronous_fetch(url, callback):
        http_client = AsyncHTTPClient()
        def handle_response(response):
            callback(response.body)
        http_client.fetch(url, callback=handle_response)

.. testoutput::
   :hide:

And again with a `.Future` instead of a callback:

.. testcode::

    from tornado.concurrent import Future

    def async_fetch_future(url):
        http_client = AsyncHTTPClient()
        my_future = Future()
        fetch_future = http_client.fetch(url)
        fetch_future.add_done_callback(
            lambda f: my_future.set_result(f.result()))
        return my_future

.. testoutput::
   :hide:

The raw `.Future` version is more complex, but ``Futures`` are
nonetheless recommended practice in Tornado because they have two
major advantages.  Error handling is more consistent since the
`.Future.result` method can simply raise an exception (as opposed to
the ad-hoc error handling common in callback-oriented interfaces), and
``Futures`` lend themselves well to use with coroutines.  Coroutines
will be discussed in depth in the next section of this guide.  Here is
the coroutine version of our sample function, which is very similar to
the original synchronous version:
``意思是“for this”引申为“for this purpose only”``
.. testcode::

    from tornado import gen

    @gen.coroutine
    def fetch_coroutine(url):
        http_client = AsyncHTTPClient()
        response = yield http_client.fetch(url)
        raise gen.Return(response.body)

.. testoutput::
   :hide:

The statement ``raise gen.Return(response.body)`` is an artifact of
Python 2 (and 3.2), in which generators aren't allowed to return
values. To overcome this, Tornado coroutines raise a special kind of
exception called a `.Return`. The coroutine catches this exception and
treats it like a returned value. In Python 3.3 and later, a ``return
response.body`` achieves the same result.``人工制品，手工艺品，加工品; 石器; ``
