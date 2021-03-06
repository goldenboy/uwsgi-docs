uWSGI RPC Stack (updated to 1.9)
================================

Albeit you may fall in love with this subsystem, try to use it when you need it.

There are lot of higher-level RPC technologies best suited for the vast majority of situations.

Where uWSGI RPC subsystem shines, is in performance and memory usage. As an example if you need to split the load of a request to multiple
servers, the uWSGI RPC is a great choice, as it allows you to offload tasks with very little effort.

Its biggest limit is in its "typeless" approach:

.. note:: RPC functions receive arguments in the form of binary strings, so every RPC exportable function must assume that each argument is a string. Every RPC function returns a binary string of 0 or more characters.

so, if you need 'elegence' or strong typization, just look in another place.

Since 1.9 the RPC subsystem is fully async-friendly, so you can use it over engine like gevent and Coro::AnyEvent

Learning by example
-------------------

Let's start with a simple RPC call from ``10.0.0.1:3031`` to ``10.0.0.2:3031``.

So let's export a "hello" function on ``.2``.

.. code-block:: py

    import uwsgi
    
    def hello_world():
        return "Hello World"
    
    uwsgi.register_rpc("hello", hello_world)

This uses :py:meth:`uwsgi.register_rpc` to declare a function called "hello" to be exported. We'll start the server with ``--server :3031``.

On the caller's side, on ``10.0.0.1``, let's declare the world's (second) simplest WSGI app.

.. code-block:: py

    import uwsgi
    
    def application(env, start_response):
        start_response('200 Ok', [('Content-Type', 'text/html')]
        return uwsgi.rpc('10.0.0.2:3031', 'hello')

That's it!

What about, let's say, Lua? 
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Glad you asked. If you want to export functions in Lua, simply do:

.. code-block:: lua

    function hello_with_args(arg1, arg2)
        return 'args are '..arg1..' '..arg2
    end
    
    uwsgi.register_rpc('hellolua', hello_with_args)

And in your Python WSGI app:

.. code-block:: py

    import uwsgi
    
    def application(env, start_response):
        start_response('200 Ok', [('Content-Type', 'text/html')]
        return uwsgi.rpc('10.0.0.2:3031', 'hellolua', 'foo', 'bar')


Doing RPC locally
-----------------

Doing RPC locally may sound a little silly, but if you need to call a Lua function from Python with the absolute least possible overhead, uWSGI RPC is your man.

If you want to call a RPC defined in the same server (governed by the same master, etc.), simply set the first parameter of :py:meth:`uwsgi.rpc`` to None or nil, or use the convenience function :py:meth:`uwsgi.call`.


Doing RPC from nginx
--------------------

As Nginx supports low-level manipulation of the uwsgi packets sent to upstream uWSGI servers, you can do RPC directly through it. Madness!

.. code-block:: nginx

    location /call {
        uwsgi_modifier1 173;
        uwsgi_modifier2 1;
        
        uwsgi_param hellolua foo
        uwsgi_param bar ""
    
        uwsgi_pass 10.0.0.2:3031;
    
        uwsgi_pass_request_headers off;
        uwsgi_pass_request_body off;
    }

Zero size strings will be ignored by the uWSGI array parser, so you can safely use them when the numbers of parameters + function_name is not even. 

Modifier2 is set to 1 to inform that raw strings (HTTP responses in this case) are received. Otherwise the RPC subsystem would encapsulate the output in an uwsgi protocol packet, and nginx isn't smart enough to read those.
