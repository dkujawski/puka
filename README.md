Puka - the opinionated RabbitMQ client
======================================

Puka is yet-another Python client library for RabbitMQ. But as opposed
to similar libraries, it does not try to expose a generic AMQP
API. Instead, it takes an opinionated view on how the user should
interact with RabbitMQ.


Puka is simple
--------------

Puka exposes a simple, easy to understand API. Take a look at the
`publisher` example:

    import puka

    client = puka.Puka("amqp://localhost/")

    promise = client.connect()
    client.wait(promise)

    promise = client.queue_declare(queue='test')
    client.wait(promise)

    promise = client.basic_publish(exchange='', routing_key='test',
                                  body='Hello world!')
    client.wait(promise)


Puka is asynchronous
--------------------

Puka by is fully asynchronous. Although, as you can see in example
above, it can behave synchronously. That's especially useful for
simple tasks when you don't want to introduce callbacks.

Here's the same code written in an asynchronous way:

    import puka

    def on_connection(promise, result):
        client.queue_declare(queue='test', callback=on_queue_declare)

    def on_queue_declare(promise, result):
        client.basic_publish(exchange='', routing_key='test',
                             body="Hello world!",
                             callback=on_basic_publish)

    def on_basic_publish(promise, result):
        print " [*] Message sent"
        client.loop_break()

    client = puka.Client("amqp://localhost/")
    client.connect(callback=on_connection)
    client.loop()


Puka never blocks
-----------------

In the pure asynchronous programming style Puka never blocks your
program waiting for network. However it is your responsibility to
notify when new data is available on the network socket. To allow that
Puka allows you to access the raw socket descriptor. With that in hand
you can construct your own event loop. Here's an the event loop that
may replace `wait_for_any` from previous example:

     fd = client.fileno()
     while True:
        client.run_any_callbacks()

        r, w, e = select.select([fd],
                                [fd] if client.needs_write() else [],
                                [fd])
        if r or e:
            client.on_read()
        if w:
            client.on_write()


Puka is fast
------------

Puka is asynchronous and has no trouble in handling many requests at a
time. This can be exploited to achieve a degree of parallelism. For
example, this snippet creates 1000 queues in parallel:

    promises = [client.queue_declare(queue='a%04i' % i) for i in range(1000)]
    for promise in promises:
        client.wait(promise)

Puka is sane
------------

Puka does expose only a sane subset of AMQP, as judged by the author.

The major differences between Puka and normal AMQP libraries include:

  - Puka doesn't expose AMQP channels to the users.
  - Puka treats `basic_publish` as a synchronous method (you can wait
    on it and make sure that your data is delivered).
  - Puka tries to cope with the AMQP exceptions and expose them
    to the user in a predictable way.


Puka is experimental
--------------------

Puka is a side project, written mostly to prove if it is possible to
create a reasonable API on top of the AMQP protocol. It is not finshed
and may be abandoned at any time.


I like it! Show me more!
------------------------

You can find more code in the `./examples` directory. Some
interesting bits:

  - `./examples/send.py`: sends one message
  - `./examples/receive_one.py`: receives one message
  - `./examples/stress_amqp_consume.py`: a script used to
    benchmark the throughput of the server


I want to install Puka
----------------------

To install Puka system-wide type:

    sudo pip install -e git+http://github.com/majek/puka.git#egg=puka

Even better is to use `virtualenv` local environment:

     virtualenv my_venv
     pip -E my_venv install -e git+http://github.com/majek/puka.git#egg=puka


I want to run the examples
--------------------------

Great. Make sure you have `rabbitmq` server installed and follow this
steps:

    git clone https://github.com/majek/puka.git
    cd puka
    make
    cd examples

Now you're ready to run the examples, start with:

    python send.py


I want to see the API documentation
-----------------------------------

The easiest way to get started is to take a look at the examples and
tweak them to your needs. You can't see more detailed documentation,
as it doesn't exist right now. If it existed it would live here:

[http://majek.github.com/puka/](http://majek.github.com/puka/)

There is also a bunch of fairly complicated examples hidden in the
tests.