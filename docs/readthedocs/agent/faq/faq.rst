    :Author: Nick Dokos

.. contents::

Frequently asked questions
--------------------------

1. I want to group related results together - how do I do that? (Part 1)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Use ``--config`` when running a benchmark.

This allows you to group together related results by using similar
config strings (and conversely, use dissimilar strings to separate
unrelated results), within the universe of your own results.

All of the benchmark scripts that pbench provides, use a standardized
name for the results directory (and hence the tarball that is packaged
up and sent to the server). E.g. ``pbench-fio`` creates a directory (under
``/var/lib/pbench-agent``) with the name ``fio__2019.01.29T19.24.29`` with just
the name of the benchmark and a time stamp.

But all the benchmark scripts take a ``--config`` option that allows an
(almost) arbitrary string to be added to the name of the results
directory: e.g ``pbench-fio --config=foo`` will produce a results
directory named ``fio_foo_2019.01.29T19.24.29``. The only character that
is not allowed is the forward slash ``/`` for obvious reasons, but you
probably want to avoid spaces, newlines, colons and characters that
have special meaning to the shell (asterisks, backslashes, question
marks, etc.) 

2. I like to rename the directory that pbench produces before I send it to the server so I can group results together - that's a good idea, right?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

No.

Well, it can be done, but please don't do that. During the run, pbench
gathers up information about the run that the indexer uses later on:
the name of the directory is one of those pieces of information, so if
you change it, it will either confuse the indexer or it will be
indexed differently than you expect, making it difficult to retrieve
those results.

From v0.55 on, ``pbench-move/copy-results`` tries to detect this situation
and will abort the move/copy, leaving the results as-is. If this happens
frequently, we'll have to add another question to the FAQ !-)

3. I want to group related results together - how do I do that? (Part 2)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Use ``--prefix`` and ``--user`` when moving/copying results to the server.

After you've run your benchmark and you are ready to send the results
to the server, there are a couple of things you should know about
``pbench-move/copy-results``.

There are two useful options to ``pbench-move-results``: ``--prefix`` and
``--user--``. The latter was introduced in v0.51, the former has existed
for some time now. Basic usage is

.. code:: shell

    pbench-move-results --user=<user> --prefix=foo/bar

Let's say there are three result directories in your controller's ``/var/lib/pbench-agent`` directory:

- ``benchmark_config_2018-08-15T20:12:21/``

- ``benchmark_config_2018-08-16T18:02:34/``

- ``benchmark_config_2018-08-15T17:55:32/``

Then the three tarballs that ``pbench-move-results`` moves to the server
are organized as follows:

- controller view: under the controller name
  ~results/<controller>/foo/bar, you will find (symlinks to) the
  directories created from unpacking the tarballs.

- user view: under ``users/<user>/<controller>/foo/bar``, you will find
  the same entries.

The value of ``prefix`` is an arbitrary sequence of names separated by ``/``.
Each component is mapped to a subdirectory of each hierarchy. The value
of ``user`` is also arbitrary, but the expectation is that users are going
to use something line an IRC nick or email address to identify themselves.
If you are calling ``pbench-move-results`` from a script, you might want to
set the value of the ``user`` option using an environment variable, instead
of passing it as an option:

.. code:: shell

    export PBENCH_USER=<user>
    ...
    pbench-move-results --prefix=foo/bar

is equivalent to

.. code:: shell

    pbench-move-results --user=<user> --prefix=foo/bar

The values of both of these options are indexed, so the dashboard will
have access to them as well.  Currently, the dashboard ignores both of
them, but we anticipate that they will soon be used as selection criteria
for the results that a user might be interested in.

4. How do I update the pbench agent?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Read the `Installation instructions <installation.rst>`_.

5. Why is the ``metadata.log`` file important?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The metadata log file collects information about the run as the
benchmark script is running (name of the run, start and end times,
controller name, tool information, etc.)  The indexer will look for
the metadata log file in the top level directory of the tarball that
is sent to the server and will **NOT** process the tarball if the
metadata log file is missing or it does not contain the information
the indexer requires.

Right now, you can look at your results because we expand and store
the tarballs on the server, but this does **not** scale: we soon will
depend **solely** on the index for visualizing the results, so it
behooves us to make sure that the indexing works. In particular, if
you write your own benchmark scripts, please make sure that it
produces a metadata log file (see e.g. ``pbench-user-bencmark``, the
simplest benchmark script, to see how to go about doing that).

Do's and don'ts
-----------------

1. Do
~~~~~~

- namespace your own results by using the ``--config`` option to the
  benchmark script - see `1.1 I want to group related results together - how do I do that? (Part 1)`_

- namespace your results (within the universe of **all** results) by
  using the ``--prefix`` and ``--user`` options of ``pbench-move-results`` -
  see: `1.3 I want to group related results together - how do I do that? (Part 2)`_

- make sure that the metadata log file exists and is complete (at
  least with a start and end time for the run) - otherwise, the
  indexer will not index the tarball.

2. Do not
~~~~~~~~~~

- move multiple results into a subdirectory of ``/var/lib/pbench-agent``
  in an effort to namespace the results: ``pbench-move-results`` will
  create a single tarball and it will unpack it seemingly
  successfully, but your results will fail to index, because the
  indexer cannot make heads or tails of the structure.

- rename results directories before you use ``pbench-move/copy-results``.
