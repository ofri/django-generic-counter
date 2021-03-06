|Build Status| |Coverage Status|

======================
django-generic-counter
======================

This small Django app provides a simple ``Counter`` model which consists
of a ``name`` and an integer ``count``. It can be used any time you need
to keep a tally of something, or store an arbitrary number in the
database.

For example, when using a `CouchDB change listener`_ it may be useful to
keep a track of the ``seq`` number you’ve reached in the database in
order to continue from the point at which you left off.

Usage
-----

Include ``django_generic_counter`` in your ``INSTALLED_APPS``, then use
the ``Counter`` like so:

.. code:: python

    from django_generic_counter import Counter

    c = Counter(name="Green bottles hanging on the wall", count=10)
    assert int(c) == 10, "Ten green bottles hanging on the wall"
    assert c.count == 10, "Ten green bottles hanging on the wall"
    c -= 1  # And if one green bottle should accidentally fall
    assert int(c) == 9, "There'll be nine green bottles hanging on the wall"

The following are features of the Counter object: 

- Assignment addition ``c += 10``
- Assignment subtraction ``c -= 10``
- Cast to int ``int(c)``
- Set count to absolute value immediately locally and in database ``c.set_count(1337)``

Usage with South
----------------

The migrations for this project in the ``migrations`` directory have
been made using Django 1.8. This means that if you’re using South you
must tell it to look in the ``south_migrations`` directory for its
migrations by putting the following in your settings file:

.. code:: python

    SOUTH_MIGRATION_MODULES = {
        'django_generic_counter': 'django_generic_counter.south_migrations'
    }

Example
-------

The following is a concrete example of the usage of a ``Counter`` when
applied to keep track of the ``seq`` number of a CouchDB change
listener.

.. code:: python

    class ChangeListener:

        def __init__(self, db_name, seq_counter_name, change_handler):
            self.db_name = db_name
            self.seq_counter_name = seq_counter_name
            self.change_handler = change_handler

        @property
        def seq(self):
            """
            Gets the seq counter for this change listener and returns its count. If no counter is
            found, returns 0.
            """
            try:
                return int(Counter.objects.get(name=self.seq_counter_name))
            except Counter.DoesNotExist:
                return 0

        @seq.setter
        def seq(self, seq):
            """
            Gets or creates the seq counter for this change listener and sets its count to 'seq'.
            """
            counter, _ = Counter.objects.get_or_create(name=self.seq_counter_name)
            counter.set_count(seq)

        def run(self):
            """
            Wait for changes and pass them to `self.change_handler` as they occur.
            """
            db = get_db(self.db_name)
            changes_stream = ChangesStream(db, feed="continuous", heartbeat=True, since=self.seq)
            for change in changes_stream:
                if self.change_handler(db, change):
                    self.seq = change["seq"]

.. _CouchDB change listener: http://guide.couchdb.org/draft/notifications.html

.. |Build Status| image:: https://travis-ci.org/0x07Ltd/django-generic-counter.svg?branch=master
   :target: https://travis-ci.org/0x07Ltd/django-generic-counter
.. |Coverage Status| image:: https://coveralls.io/repos/0x07Ltd/django-generic-counter/badge.svg?branch=master&service=github
   :target: https://coveralls.io/github/0x07Ltd/django-generic-counter?branch=master