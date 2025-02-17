---
html_meta:
  "description": ""
  "property=og:description": ""
  "property=og:title": ""
  "keywords": ""
---

# WSGI options

The following options affect the contents of the `wsgi.ini` file generated by `plone.recipe.zope2instance`.

## http-address

This is the same for both ZServer and waitress.
However for the WSGI case the actual configuration will be written to `wsgi.ini` rather than `zope.conf`.

You can specify both a simple port (e.g. 9090) or a socket (e.g. 127.0.0.1:9090) with this option.
The default is to listen on 0.0.0.0:8080.

```{note}
Setting this to an empty string (as proposed in outdated p.r.zope2instance documentation in order to run Zope under an external server) will result in a flaky
listen directive in wsgi.ini (`listen = 0.0.0.0:`)
Waitress will open on a random non-privileged port in this case.
```

## threads

The new `threads` option specifies the number of worker threads for both WSGI and ZServer.
The WSGI default is 4 threads since this is the waitress default.
The ZServer default of two threads remains unchanged.
`zserver-threads` can still be used to specify the number of worker threads for ZServer but not waitress.
`zserver-threads` is deprecated.

```{note}
To assess the advantages and disadvantages of a particular server configuration, we need to understand some basics of how client requests are processed in a WSGI server.

Incoming client HTTP requests are often handled in an asynchronous way using the [asyncore](https://docs.python.org/3/library/asyncore.html#module-asyncore) module from the Python standard library.
It was originally written by Sam Rushing as part of [Medusa](http://www.nightmare.com/medusa) and is thus part of Zope's [ZServer](https://github.com/zopefoundation/ZServer/tree/master/src/ZServer).
Its function in ZServer is to handle incoming client requests in an efficient way using the `select` and `poll` system calls.
It is responsible for clearing the request queue by dispatching incoming requests to available workers.
Only a **single** thread is needed for this task.
Waitress uses `asyncore` just like ZServer for exactly the same task.
However `asyncore` is deprecated since Python 3.6 in favour of the new [asyncio](https://docs.python.org/3/library/asyncio.html#module-asyncio) library using `async/await` syntax.
[Waitress has therefore "vendored" the module as wasyncore](https://docs.pylonsproject.org/projects/waitress/en/stable/glossary.html#term-asyncore) in case it will be removed from the Python standard library in Python 3.8 or later.
Both ZServer and waitress use "traditional" multi-threading based on the Python standard library's `thread` (renamed to `_thread` in Python 3) or `threading` modules to spawn worker threads.
The number of workers is configured in `wsgi.ini` (waitress) or `zope.conf` (ZServer).
Worker threads are spawned **once** when the Plone instance is starting up.
In Zope and Plone each of these workers will also open a ZODB connection.
You can check this in the ZMI Control Panel, {guilabel}`Database Management`.

Other WSGI servers like bjoern, gunicorn and uWSGI implement different or additional types of workers and they also differ in the way they are clearing the request queue.
We will try to cover at least some of the details in later chapters.
```

## wsgi

This parameter has two forms.
It can be either `on` or `off`, signaling whether to use the default WSGI server waitress, or not use a WSGI server but use ZServer instead.
The latter is only an option for Python 2 since ZServer is not (yet?) available for Python 3.

Alternatively you can specify your own PasteDeploy `ini` file with this option.
No `wsgi.ini` configuration will be provided in this case.
Most likely `wsgi-ini-template` will be easier to use however, see the next section.

## wsgi-ini-template

The path to a WSGI configuration file template for a PasteDeploy compatible WSGI server.
We will use this form later in the training.

## debug-exceptions

Use WSGI middleware debugging facilities.
When set to `on` this option will disable exception views and thus propagate errors to the WSGI stack.
This allows you to use specific debugging WSGI middleware like the [werkzeug debugger](https://werkzeug.palletsprojects.com/en/0.15.x/debug/).
We will cover this in the {doc}`debugging chapter <debugging>`.

## access-log, z2-log

`access-log` has been introduced as an alias for the `z2-log` option to specify the location of the access log file.

## access-log-level, z2-log-level

`access-log-level` is a new alias for `z2-log-level` to specify the log level of the access log file.
For WSGI the default is `INFO` (`WARN` for ZServer).

## Advanced logging for WSGI

`access-log-handler`, `access-log-args`, `access-log-kwargs`, `event-log-handler`, `event-log-args`, `event-log-kwargs` are WSGI only options that you can use to configure arbitrary Python logging handlers.
Numerous logging handlers are defined in the [Python standard library](https://docs.python.org/3/library/logging.handlers.html).

### Exercise 1

Configure Plone access logging to a TCP Server using `logging.handlers.SocketHandler` from the Python standard library.
Use a local address.

````{admonition} Solution
:class: toggle

Add the following two lines to your buildout configuration (we use `options.cfg`) from the `wsgitraining_buildout` as a starting point:

```{code-block} ini
:emphasize-lines: 9-10

...
[instance]
recipe = plone.recipe.zope2instance
user = admin:admin
zeo-client = on
zeo-address = 8100
shared-blob = on
blob-storage = ${buildout:directory}/var/blobstorage
access-log-handler = logging.handlers.SocketHandler
access-log-args = ('localhost', 9000)
eggs =
    Plone
    wsgitraining.site
```

`plone.recipe.zope2instance` uses default values for the `access-log-args` option, but not the `access-log-kwargs` option.
Same for the `event-log-args/event-log-kwargs` options.
This means you **have** to provide the `*-log-args` parameter, otherwise you will end up with the (in our case nonsensical) defaults in your `wsgi.ini`.
After running buildout with `buildout -c options.cfg` you can start your instance with `bin/instance fg`.
Use a tool such as [netcat](http://netcat.sourceforge.net/) (there is a package for your linux distribution) to open a listening socket: `nc -l 9000`.
You will see the incoming log entries in pickled format when navigating to your Plone instance in the browser.
````

## sentry\* options

Sentry support for WSGI is  available through `plone.recipe.zope2instance`.
We will cover these options later in the {doc}`add-ons chapter <addons>`.

## Options that are currently unavailable for WSGI

The following options are currently not available for WSGI:

- `access-log-custom`, `access-log-oldfiles`, `access-log-max-size`, `event-log-custom`, `event-log-oldfiles` and `event-log-max-sie` can be replaced by the new `*-log-handler`, `*-log-args` and `*-log-kwargs` options.
  See above and also the examples given in the recipe [README](https://github.com/plone/plone.recipe.zope2instance#advanced-logging-options-for-wsgi).
- `ip-address` is not necessary because HTTP is the only supported protocol for WSGI and the IP address can be specified with `http-address`.
- `ftp-address` since FTP is not supported by waitress.
- `icp-address` since ICP is also not supported by waitress.
- `webdav-address/webdav-force-connection-close` since WebDAV is also not supported by waitress.
- `http-header-max-length` waitress has a `max_request_header_size` parameter so it should be possible to add this to `plone.recipe.zope2instance`.
  You could use the `wsgi-ini-template` option to provide this parameter.
