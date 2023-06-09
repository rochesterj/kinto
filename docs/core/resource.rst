.. _resource:

Resource
########

*Kinto-Core* provides a basic component to build resource oriented APIs.
In most cases, the main customization consists in defining the schema of the
records for this resource.


Full example
============

.. code-block:: python

    import colander

    from kinto.core import resource
    from kinto.core import utils


    class BookmarkSchema(resource.ResourceSchema):
        url = colander.SchemaNode(colander.String(), validator=colander.url)
        title = colander.SchemaNode(colander.String())
        favorite = colander.SchemaNode(colander.Boolean(), missing=False)
        device = colander.SchemaNode(colander.String(), missing='')

        class Options:
            readonly_fields = ('device',)


    @resource.register()
    class Bookmark(resource.Resource):
        schema = BookmarkSchema

        def process_object(self, new, old=None):
            new = super().process_object(new, old)
            if new['device'] != old['device']:
                new['device'] = self.request.headers.get('User-Agent')

            return new

See the `ReadingList <https://github.com/mozilla-services/readinglist>`_ and
`Kinto <https://github.com/mozilla-services/kinto>`_ projects source code for real use cases.


.. _resource-urls:

URLs
====

By default, a resource defines two URLs:

* ``/{classname}s`` for the list of records
* ``/{classname}s/{id}`` for single records

Since adding an ``s`` suffix for the plural form might not always be relevant,
URLs can be specified during registration:

.. code-block:: python

    @resource.register(plural_path='/user/bookmarks',
                       object_path='/user/bookmarks/{{id}}')
    class Bookmark(resource.Resource):
        schema = BookmarkSchema

.. note::

    The same resource can be registered with different URLs.



Schema
======

Override the base schema to add extra fields using the `Colander API <http://docs.pylonsproject.org/projects/colander/>`_.

.. code-block:: python

    class Movie(resource.ResourceSchema):
        director = colander.SchemaNode(colander.String())
        year = colander.SchemaNode(colander.Int(),
                                   validator=colander.Range(min=1850))
        genre = colander.SchemaNode(colander.String(),
                                    validator=colander.OneOf(['Sci-Fi', 'Comedy']))

See the :ref:`resource schema options <resource-schema>` to define *schema-less*
resources or specify rules like readonly fields.


HTTP methods and options
========================

In order to specify which HTTP verbs (``GET``, ``PUT``, ``PATCH``, ...)
are allowed on the resource, as well as specific custom Pyramid (or `Cornice <https://cornice.readthedocs.io>`_)
view arguments, refer to the :ref:`viewset section <viewset>`.


Events
======

When a record is created/deleted in a resource, an event is sent.
See the `dedicated section about notifications <notifications>`_ to plug events
in your Pyramid/*Kinto-Core* application or plugin.


Model
=====

Plug custom model
-----------------

In order to customize the interaction of a HTTP resource with its storage,
a custom model can be plugged-in:

.. code-block:: python

    from kinto.core import resource


    class TrackedModel(resource.Model):
        def create_record(self, record, parent_id=None):
            record = super().create_record(record, parent_id)
            trackid = index.track(record)
            record['trackid'] = trackid
            return record


    class Payment(resource.Resource):
        default_model = TrackedModel


Relationships
-------------

With the default model and storage backend, *Kinto-Core* does not support complex
relations.

However, it is possible to plug a custom :ref:`model class <resource-model>`,
that will take care of saving and retrieving records with relations.

.. note::

    This part deserves more love, `please come and discuss <https://github.com/mozilla-services/cliquet/issues/135>`_!


In Pyramid views
----------------

In Pyramid views, a ``request`` object is available and allows to use the storage
configured in the application:

.. code-block:: python

    from kinto.core import resource

    def view(request):
        registry = request.registry

        flowers = resource.Model(storage=registry.storage,
                                 resource_name='app:flowers')

        flowers.create_object({'name': 'Jonquille', 'size': 30})
        flowers.create_object({'name': 'Amapola', 'size': 18})

        min_size = resource.Filter('size', 20, resource.COMPARISON.MIN)
        objects = flowers.get_objects(filters=[min_size])

        flowers.delete_object(records[0])


Outside views
-------------

Outside views, an application context has to be built from scratch.

As an example, let's build a code that will copy a collection into another:

.. code-block:: python

    from kinto.core import resource, DEFAULT_SETTINGS
    from pyramid import Configurator


    config = Configurator(settings=DEFAULT_SETTINGS)
    config.add_settings({
        'kinto.storage_backend': 'kinto.core.storage.postgresql'
        'kinto.storage_url': 'postgresql://user:pass@db.server.lan:5432/dbname'
    })
    kinto.core.initialize(config, '0.0.1')

    local = resource.Model(storage=config.registry.storage,
                           parent_id='browsing',
                           resource_name='history')

    remote = resource.Model(storage=config_remote.registry.storage,
                            parent_id='',
                            resource_name='history')

    records, total = in remote.get_records():
    for record in records:
        local.create_record(record)


Custom record ids
=================

By default, records ids are `UUID4 <http://en.wikipedia.org/wiki/Universally_unique_identifier>`_.

A custom record ID generator can be set globally in :ref:`configuration`,
or at the resource level:

.. code-block:: python

    from kinto.core import resource
    from kinto.core import utils
    from kinto.core.storage import generators


    class MsecId(generators.Generator):
        def __call__(self):
            return '%s' % utils.msec_time()


    @resource.register()
    class Mushroom(resource.Resource):
        def __init__(request):
            super().__init__(request)
            self.model.id_generator = MsecId()


Python API
==========

.. autofunction:: kinto.core.resource.register

.. _resource-class:

Resource
--------

.. autoclass:: kinto.core.resource.Resource
    :members:

.. _resource-schema:

Schema
------

.. automodule:: kinto.core.resource.schema
    :members:

.. _resource-model:

Model
-----

.. automodule:: kinto.core.resource.model
    :members:

.. _resource-generators:

Generators
----------

.. automodule:: kinto.core.storage.generators
    :members:
