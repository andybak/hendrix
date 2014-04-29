#Hendrix
**Twisted meets Django: *Making deployment easy***

##Overview

Hendrix is a **multi-threaded**, **multi-process** *and* **asynchronous**
web server for *Django* projects. The idea here is to have a highly extensible
wsgi deployment tool that is fast, transparent and easy to use.

###Features
* **Multi-processing** - The WSGI app can be served from multiple
processes on a single machine. Meaning that it can leverage multicore
servers.
* **Optional Resource Caching** - Process distributed resource caching, dynamically serving
gzipped files. It's also possible to extend the current caching functionality by
subclassing the hendrix `CacheService` and adding a reference to that subclass within a
`HENDRIX_SERVICES` tuple of tuples e.g. `(('name', 'app.MyCustomCache'),)` in your
Django settings file. Just use the `--nocache` flag to turn caching off.
* **Built-in SSL Support** - SSL is also process distributed, just provide the options
--key /path/to/private.key and --cert /path/to/signed.cert to enable SSL
and process distributed TLS
* **Multi-threading from within Django** - Enables the use of Twisted's `defer`
module and `deferToThread` and `callInThread` methods from within Django
* **Built-in Websockets Framework**
* **Daemonize** by passing the `-d` or `--daemonize` flags to `hx`

###Installation

Using pip

    pip install hendrix

or for the cutting edge...

    pip install -e git+git@github.com:hangarunderground/hendrix.git@master#egg=hendrix

###Deployment/Usage

Hendrix leverages the use of custom django management commands. That
and a little magic makes `hx` available as a shortcut to all of
the functionality you'd expect from `python manage.py hx`.

Note: `hx` needs to know where your project lives so `cd` to where you house
your Django project.

You can but *don't* have to include "hendrix" in your project's INSTALLED_APPS list
```python
INSTALLED_APPS = (
    ...,
    'hendrix',
    ...
)
```
But if you really want to run Hendrix using `python manage.py hx [action]` then
you have to include it.

###Ubuntu Servers
If you're running an Ubuntu/Debian server you may want to install
Hendrix as a service. To do this you can use the helper script `install-hendrix-service`.
Here's how you do it:

1. If you don't already have a [virtualenv](http://docs.python-guide.org/en/latest/dev/virtualenvs/)
for you project install one and in that install your project's requirements (which now includes Hendrix):

    ```bash
    virtualenv --no-site-packages --distribute /path/to/new/venv

    source /path/to/new/venv/bin/activate

    pip install -r /path/to/myproject/requirements.txt
    ```

2. Create a hendrix.conf file. Don't worry it's yaml (so it's easy). You can
copy the [example](https://github.com/hangarunderground/hendrix/blob/master/hendrix/utils/example-yaml.conf)
and save it wherever you want. You could even do something like this if you want:

    ```bash
    cd /wherever

    wget https://raw.githubusercontent.com/hangarunderground/hendrix/master/hendrix/utils/example-yaml.conf
    mv example-yaml.conf hendrix.conf

    # open the file in an editor and change the options
    nano -c hendrix.conf
    ```

3. Finally, run your own version of the following command:

    ```bash
    sudo install-hendrix-service /wherever/hendrix.conf
    ```

Now you have hendrix running as a service on your server. Easy.
Use the service as you would any other service i.e. `sudo service hendrix start`.

Coming soon/if you want to build one: an [ansible galaxy](https://galaxy.ansible.com/)
role would be great.

###From the Command Line
The following outlines how to use Hendrix in your day to day life/development.

#####For help and a complete list of the options:

    hx -h

or

    hx --help

#####Starting a server with 4 processes (1 parent and 3 child processes):

    hx start -w 3

#####Stoping that server:

    hx stop


** *Note that stoping a server is dependent on the settings file and http_port
used.* **

E.g. Running a server on port 8000 with local_settings.py would yield
8000_local_settings.pid which would be used to kill the server. I.e. if you
start with `hx start --settings local_settings` then stop by `hx stop --settings local_settings`

#####Restarting a server:

    hx restart



##Features in more depth

###Serving Static Files
Serving static files via **Hendrix** is optional but easy.


a default static file handler is built into Hendrix which can be used by adding the following to your settings:
```
HENDRIX_CHILD_RESOURCES = (
    'hendrix.contrib.resources.static.DefaultDjangoStaticResource',
)
```
No other configuration is necessary.  You don't need to add anything to urls.py.

You can also easily create your own custom static or other handlers by adding them to HENDRIX\_CHILD\_RESOURCES.


###Running the Development Server
```
hx start --reload
```
This will reload your server every time a change is made to a python file in
your project.

#####Using "dev" mode
You can also use the development mode, which colourfully prints requests.
```bash
hx start --dev
```
You could also mix it up with other options:
```
hx start --dev --reload --w 2
```
or
```
hx start --settings production_settings --dev
```
etcetera...


###SSL
This is made possible by creating a self-signed key. First make sure you have
the newest **patched** version of openssl.
Then generate a private key file:

    openssl genrsa > key.pem

Then generate a self-signed SSL certificate:

    openssl req -new -x509 -key key.pem -out cacert.pem -days 1000


Finally you can run single SSL server by running:

    hx start --dev --key key.pem --cert cacert.pem

or a process distributed set of SSL servers:

    hx start --dev --key key.pem --cert cacert.pem -w 3

Just go to `https://[insert hostname]:4430/` to check it out.
N.B. Your browser will warn you not to trust the site... You can also specify
which port you want to use by passing the desired number to the `--https_port`
option


###Caching

At the moment a caching server is deployed by default on port 8000. Is serves
gzipped content, which is pretty cool - right?

#####How it works

The Hendrix cache server is a reverse proxy that sits in front your Django app.
However, if you wanted to switch away from the cache server you can always point
to the http port (default 8080).

It works by forwarding requests to the http server running the app and
caches the response depending on the availability of `max-age` [seconds] in a
`Cache-Control` header.

#####Busting the cache

Note that you can bust the cache (meaning force it not to cache) by passing
a query in your GET request e.g. `http://somesite.com/my/resource?somevar=test`.
You can also force the query to cache by specifying `cache=true` in the query
e.g. `http://somesite.com/my/resource?somevar=test,cache=true` (so long as a
`max-age` is specified for the handling view).
What this means is that you can let the browser do some or none of the js/css
caching if you so want.

#####Caching in Django

In your project view modules use the `cache_control` decorator to add
a `max-age` of your choosing. e.g.

```python
from django.views.decorators.cache import cache_control

@cache_control(max_age=60*60*24)  # cache it for a day
def homePageView(request):
    ...
```

and that's it! Hendrix will do the rest. Django docs examples [here](https://docs.djangoproject.com/en/dev/topics/cache/#controlling-cache-using-other-headers)

#####Turning cache off

You can turn caching off by passing the flags `-n` or `--nocache`. You can also change which
port you want to use with the `--cache_port` option.

#####Global Vs Local

If you're running multiple process using the `-w` or `--workers` options caching
will be process distributed by default. Meaning there will be a reverse proxy
cache server for each process. However if you want to run the reverse
proxy server on a single process just use the `-g` or `--global_cache` flags.

... here "local" means local to the process.


###Testing
Tests live in `hendrix.test` and are most easily run using Twisted's
[trial](https://twistedmatrix.com/trac/wiki/TwistedTrial) test framework.
```
/home/jsmith:~$ trial hendrix
hendrix.test.test_deploy
  DeployTests
    test_multiprocessing ...                                               [OK]
    test_options_structure ...                                             [OK]
    test_settings_doesnt_break ...                                         [OK]

-------------------------------------------------------------------------------
Ran 3 tests in 0.049s

PASSED (successes=3)
```
**trial** will find your tests so long as you name the package/module such that
it starts with "test" e.g. `hendrix/contrib/cache/test/test_services.py`.

Note that the module needs to have a subclass of unittest.TestCase via the expected
unittest pattern. For more info on *trial* go [here](https://twistedmatrix.com/trac/wiki/TwistedTrial).

N.B. that in the `hendrix.test` `__init__.py` file a subclass of TestCase
called `HendrixTestCase` has been created to help tests various use cases
of `hendrix.deploy.HendrixDeploy`


###Contributions
Contributions are more than welcome. Feedback and bug reports especially.

Unfortunately a majority of the codebase is **not** covered by tests.
This means that if you want to make changes to Hendrix you'll need to *click
around* to see if it *works*. We're working hard to change this so
please bare with us until then.


###Twisted
Twisted is what makes this all possible. Mostly. Check it out [here](https://twistedmatrix.com/trac/).


###Yet to come
* Ensure stability of current implementation of web sockets
* Load Balancing
* Twisted logging
* Cache regex logic/config file
* Ansible deployment utility
* Salt deployment utility
* more tests...


###History
It started as a fork of the
[slashRoot deployment module](https://github.com/SlashRoot/WHAT/tree/44f50ee08c5d7acb74ed8a4ce928e85eb2dc714f/deployment).

The name is the result of some inane psychological formula wherein the
'twisted' version of Django Reinhardt is Jimi Hendrix.

Hendrix is currently maintained by [Reelio](reelio.com).
