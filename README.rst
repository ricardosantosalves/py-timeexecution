Time Execution
==============


.. image:: https://github.com/kpn/py-timeexecution/actions/workflows/test.yml/badge.svg?branch=master
    :target: https://github.com/kpn/py-timeexecution/actions/workflows/test.yml

.. image:: https://img.shields.io/codecov/c/github/kpn/py-timeexecution/master.svg
    :target: http://codecov.io/github/kpn/py-timeexecution?branch=master

.. image:: https://img.shields.io/pypi/v/timeexecution.svg
    :target: https://pypi.org/project/timeexecution

.. image:: https://img.shields.io/pypi/pyversions/timeexecution.svg
    :target: https://pypi.org/project/timeexecution

.. image:: https://readthedocs.org/projects/py-timeexecution/badge/?version=latest
    :target: http://py-timeexecution.readthedocs.org/en/latest/?badge=latest

.. image:: https://img.shields.io/pypi/l/timeexecution.svg
    :target: https://pypi.org/project/timeexecution

.. image:: https://img.shields.io/badge/code%20style-black-000000.svg

This package is designed to record application metrics into specific backends.
With the help of Grafana_ or Kibana_ you can easily use these metrics to create meaningful monitoring dashboards.


Features
--------

- Sending data to multiple backends (e.g. ElasticSearch)
- Custom backends
- Hooks to include additional data per metric.

Available backends
------------------

- InfluxDB 0.8
- Elasticsearch >=5,<7
- Kafka


Installation
------------

If you want to use it with the ``ElasticSearchBackend``:

.. code-block:: bash

    $ pip install timeexecution[elasticsearch]

with ``InfluxBackend``:

.. code-block:: bash

    $ pip install timeexecution[influxdb]

with ``KafkaBackend``:

.. code-block:: bash

    $ pip install timeexecution[kafka]

or if you prefer to have all backends available and easily switch between them:

.. code-block:: bash

    $ pip install timeexecution[all]


Usage
-----

To use this package you decorate the functions you want to time its execution.
Every wrapped function will create a metric consisting of 3 default values:

- ``name`` - The name of the series the metric will be stored in. Byt default, timeexecution will use the fully qualified name of the decorated method or function (e.g. ).
- ``value`` - The time it took in ms for the wrapped function to complete
- ``hostname`` - The hostname of the machine the code is running on

See the following example

.. code-block:: python

    from time_execution import settings, time_execution
    from time_execution.backends.influxdb import InfluxBackend
    from time_execution.backends.elasticsearch import ElasticsearchBackend

    # Setup the desired backend
    influx = InfluxBackend(host='influx', database='metrics', use_udp=False)
    elasticsearch = ElasticsearchBackend('elasticsearch', index='metrics')

    # Configure the time_execution decorator
    settings.configure(backends=[influx, elasticsearch])

    # Wrap the methods where u want the metrics
    @time_execution
    def hello():
        return 'World'

    # Now when we call hello() and we will get metrics in our backends
    hello()

This will result in an entry in the influxdb

.. code-block:: json

    [
        {
            "name": "__main__.hello",
            "columns": [
                "time",
                "sequence_number",
                "value",
                "hostname",
            ],
            "points": [
                [
                    1449739813939,
                    1111950001,
                    312,
                    "machine.name",
                ]
            ]
        }
    ]

And the following in Elasticsearch

.. code-block:: json

    [
        {
            "_index": "metrics-2016.01.28",
            "_type": "metric",
            "_id": "AVKIp9DpnPWamvqEzFB3",
            "_score": null,
            "_source": {
                "timestamp": "2016-01-28T14:34:05.416968",
                "hostname": "dfaa4928109f",
                "name": "__main__.hello",
                "value": 312
            },
            "sort": [
                1453991645416
            ]
        }
    ]

It's also possible to run backend in different thread with logic behind it, to send metrics in bulk mode.

For example:

.. code-block:: python

    from time_execution import settings, time_execution
    from time_execution.backends.threaded import ThreadedBackend

    # Setup threaded backend which will be run on separate thread
    threaded_backend = ThreadedBackend(
        backend=ElasticsearchBackend,
        backend_kwargs={
            "host" : "elasticsearch",
            "index": "metrics",
        }
    )

    # there is also possibility to configure backend by import path, like:
    threaded_backend = ThreadedBackend(
        backend="time_execution.backends.kafka.KafkaBackend",
        #: any other configuration belongs to backend
        backend_kwargs={
            "hosts" : "kafka",
            "topic": "metrics"
        }
    )

    # Configure the time_execution decorator
    settings.configure(backends=[threaded_backend])

    # Wrap the methods where u want the metrics
    @time_execution
    def hello():
        return 'World'

    # Now when we call hello() we put metrics in queue to send it either in some configurable time later
    # or when queue will reach configurable limit.
    hello()

It's also possible to decorate coroutines or awaitables in Python >=3.5.

For example:

.. code-block:: python

    import asyncio
    from time_execution import time_execution_async

    # ... Setup the desired backend(s) as described above ...

    # Wrap the methods where you want the metrics
    @time_execution_async
    async def hello():
        await asyncio.sleep(1)
        return 'World'

    # Now when we schedule hello() we will get metrics in our backends
    loop = asyncio.get_event_loop()
    loop.run_until_complete(hello())


.. _usage-hooks:

Hooks
-----

``time_execution`` supports hooks where you can change the metric before its
being sent to the backend.

With a hook you can add additional and change existing fields. This can be
useful for cases where you would like to add a column to the metric based on
the response of the wrapped function.

A hook will always get 3 arguments:

- ``response`` - The returned value of the wrapped function
- ``exception`` - The raised exception of the wrapped function
- ``metric`` - A dict containing the data to be send to the backend
- ``func_args`` - Original args received by the wrapped function.
- ``func_kwargs`` - Original kwargs received by the wrapped function.

From within a hook you can change the `name` if you want the metrics to be split
into multiple series.

See the following example how to setup hooks.

.. code-block:: python

    # Now lets create a hook
    def my_hook(response, exception, metric, func, func_args, func_kwargs):
        status_code = getattr(response, 'status_code', None)
        if status_code:
            return dict(
                name='{}.{}'.format(metric['name'], status_code),
                extra_field='foo bar'
            )

    # Configure the time_execution decorator, but now with hooks
    settings.configure(backends=[backend], hooks=[my_hook])


There is also possibility to create decorator with custom set of hooks. It is needed for example to track `celery` tasks.

.. code-block:: python

    from multiprocessing import current_process
    # Hook for celery tasks
    def celery_hook(response, exception, metric, func, func_args, func_kwargs):
        """
        Add celery worker-specific details into response.
        """
        p = current_process()
        hook = {
            'name': metric.get('name'),
            'value': metric.get('value'),
            'success': exception is None,
            'process_name': p.name,
            'process_pid': p.pid,
        }
        return hook

    # Create time_execution decorator with extra hooks
    time_execution_celery = time_execution(extra_hooks=[celery_hook])

    @celery.task
    @time_execution_celery
    def celery_task(self, **kwargs):
        return True

    # Or do it in place where it is needed
    @celery.task
    @time_execution(extra_hooks=[celery_hook])
    def celery_task(self, **kwargs):
        return True

    # Or override default hooks by custom ones. Just setup `disable_default_hooks` flag
    @celery.task
    @time_execution(extra_hooks=[celery_hook], disable_default_hooks=True)
    def celery_task(self, **kwargs):
        return True



Manually sending metrics
------------------------

You can also send any metric you have manually to the backend. These will not
add the default values and will not hit the hooks.

See the following example.

.. code-block:: python

    from time_execution import write_metric

    loadavg = os.getloadavg()
    write_metric('cpu.load.1m', value=loadavg[0])
    write_metric('cpu.load.5m', value=loadavg[1])
    write_metric('cpu.load.15m', value=loadavg[2])


Custom Backend
--------------

Writing a custom backend is very simple, all you need to do is create a class
with a `write` method. It is not required to extend `BaseMetricsBackend`
but, in order to easily upgrade, we recommend you do.

.. code-block:: python

    from time_execution.backends.base import BaseMetricsBackend


    class MetricsPrinter(BaseMetricsBackend):
        def write(self, name, **data):
            print(name, data)


Example scenario
----------------

In order to read the metrics, e.g. using ElasticSearch as a backend, the following lucene query could be used:

.. code-block::

    name:"__main__.hello" AND hostname:dfaa4928109f

For more advanced query syntax, please have a look at the `Lucene documentation`_ and the `ElasticSearch Query DSL`_ reference.


Contribute
----------

You have something to contribute? Great! There are a few things that may come in handy.

Testing in this project is done via tox with the use of docker.

There is a Makefile with a few targets that we use often:

- ``make test``
- ``make format``
- ``make lint``
- ``make build``

``make test`` command will run tests for the python versions specified in ``tox.ini`` spinning up all necessary services via docker.

In some cases (on Ubuntu 18.04) the Elasticsearch Docker image might not be able to start and will exit with the following error::

    max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
    
This can be solved by adding the following line to `/etc/sysctl.conf`::

    vm.max_map_count=262144

.. _Grafana: http://grafana.org/
.. _Kibana: https://www.elastic.co/products/kibana
.. _Lucene Documentation: https://lucene.apache.org/core/documentation.html
.. _ElasticSearch Query DSL: https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html
