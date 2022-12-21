=================
PBench User Guide
=================


.. contents::



1 What is ``pbench``?
---------------------

PBench is a harness that allows data collection from a variety of tools
while running a benchmark. PBench has some built-in script that run some
common benchmarks, but the data collection can be run separately as well
with a benchmark that is not built-in to pbench, or a pbench script can
be written for the benchmark. Such contributions are more than welcome!

2 TL;DR - How to set up pbench and run a benchmark
--------------------------------------------------

Prerequisite: Somebody has already done the server setup.

The following steps assume that only a single node participates in the benchmark run. If you
want a multi-node setup, you have to read up on the ``--remote`` options of various commands
(in particular, ``pbench-register-tool-set``):

- `Install the agent <installation.rst>`_.

- Run your benchmark with a default set of tools:

  ::

      . /etc/profile.d/pbench-agent.sh                          # or log out and log back in
      pbench-register-tool-set
      pbench-user-benchmark --config test1 -- ./your_cmd.sh
      pbench-move-results

- Visit the Results URL in your browser to see the results: the URL
  depends on who the server is; assuming that the server is
  "pbench.example.com" and assuming you ran the above on a host named
  "myhost", the results will be found at (**N.B.**: this is a fake link
  serving as an example only - talk to your local administrator to find out
  what server to use to get to pbench results):
  `http://pbench.example.com/results/myhost/pbench-user-benchmark_test1_yyyy-mm-dd_HH:MM:SS <http://pbench.example.com/results/myhost/pbench-user-benchmark_test1_yyyy-mm-dd_HH:MM:SS>`_.

For explanations and details, see subsequent sections.

3 How to install
----------------

See  the `installation instructions <./installation.rst>`_.

4 First steps
-------------

All of the commands take a ``--help`` option and produce a terse
usage message.

The default set of tools for data collection can be enabled with

::

    pbench-register-tool-set

You can then run a built-in benchmark by invoking its pbench script -
pbench will install the benchmark if necessary [1]_ :

::

    pbench-fio

When the benchmark finishes, the tools will be stopped as well. The
results can be collected and shipped to the standard storage location [2]_ 
with:

::

    pbench-move-results

or

::

    pbench-copy-results

4.1 First steps with pbench-user-benchmark
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you want to run something that is not already packaged up as a benchmark script,
you may be able to use the ``pbench-user-benchmark`` script: it takes a command as argument,
starts the collection tools, invokes the command, stops the collection tools and
postprocesses the results. So the workflow becomes:

::

    pbench-register-tool-set
    pbench-user-benchmark --config=foo -- myscript.sh
    pbench-move-results

See for more information on that.

4.2 First steps with remote hosts and pbench-user-benchmark
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Running a multihost benchmark involves registering the tools on all the hosts,
but assuming you have a script that will execute your benchmark that can be
used with ``pbench-user-benchmark``, the workflow is not much different:

::

    for host in $hosts ;do
        pbench-register-tool-set --remote=$host
    done
    pbench-user-benchmark --config=foo -- myscript.sh
    pbench-move-results

Apart from having to register the collection tools on **all** the hosts, the rest
is the same: ``pbench-user-benchmark`` will start the collection tools on all the hosts,
run ``myscript.sh``, stop the tools and run the postprocessing phase, gathering up
all the remote results to the local host (the local host may be just a controller,
not running any collection tools itself, or it may be part of the set of hosts where
the benchmark is run, with collection tools running).

The underlying assumption is that ``myscript.sh`` will run your
benchmark on all the relevant hosts and will copy all the results into
the standard directory which postprocessing will copy over to the
controller host. ``pbench-user-benchmark`` calls the script in its command-line
arguments (everything after the -- is just execed by ``-pbench-user-benchmark``)
and redirects its ``stdout`` to a file in that directory:
``$benchmark_run_dir/result.txt``.

5 Defaults
----------

The benchmark scripts source the base script (``/opt/pbench-agent/base``)
which sets a bunch of defaults, the most important of which is where the
results of a run are gathered:

::

    pbench_run=/var/lib/pbench-agent

The config file ``/opt/pbench-agent/config/pbench-agent.cfg`` contains the
default values for a variety of settings.

6 Available tools
-----------------

The configured default set of tools (what you would get by running
``pbench-register-tool-set``) is:

- sar, iostat, mpstat, pidstat, proc-vmstat, proc-interrupts, perf

In addition, there are tools that can be added to the default set
with ``pbench-register-tool``:

- blktrace, cpuacct, dm-cache, docker, kvmstat, kvmtrace, lockstat,
  numastat, perf, porc-sched\_debug, proc-vmstat, qemu-migrate, rabbit,
  strace, sysfs, systemtap, tcpdump, turbostat, virsh-migrate, vmstat

There is a ``default`` group of tools (that's what ``pbench-register-tool-set`` uses), but
tools can be registered in other groups using the ``--group`` option of ``pbench-register-tool``.
The group can then be started and stopped using ``pbench-start-tools`` and ``pbench-stop-tools``
using their ``--group`` option.

Additional tools can be registered:

::

    pbench-register-tool --name blktrace

or unregistered (e.g. some people prefer to run without perf):

::

    pbench-unregister-tool --name perf

Note that perf is run in a "low overhead" mode with options "record -a
--freq=100", but if you want to run it differently, you can always
unregister it and register it again with different options:

::

    pbench-unregister-tool --name=perf
    pbench-register-tool --name=perf -- --record-opts="record -a --freq=200"

Tools can be also be registered, started and stopped on remote hosts
(see the ``--remote`` option described in ).

7 Available benchmark scripts
-----------------------------

PBench provides a set of pre-packaged script to run some common benchmarks
using the collection tools and other facilities that pbench provides.  These
are found in the ``bench-scripts`` directory of the pbench installation
(``/opt/pbench-agent/bench-scripts`` by default). The current set includes:

- ``pbench-fio``

- ``pbench-linpack``

- ``pbench-uperf``

- ``pbench-user-benchmark`` (see `10 Running pbench collection tools with an arbitrary benchmark`_ below for more on this)

You can run any of these with the ``--help`` option to get basic
information about how to run the script. Most of these scripts accept
a standard set of generic options, some semi-generic ones that are
common to a bunch of benchmarks, as well as some benchmark specific
options that vary from benchmark to benchmark.

The generic options are:

``--help``
    show the set of options that the benchmark accepts.

``--config``
    the name of the testing configuration (user specified).

``--tool-group``
    the name of the tool group specifying the tools to run during execution of the benchmark.

``--install``
    just install the benchmark (and any other needed packages) - do not run the benchmark.

The semi-generic ones are:

``--test-types``
    the test types for the given benchmark - the values are benchmark-specific and can be obtained using ``--help``.

``--runtime``
    maximum runtime in seconds.

``--clients``
    list of hostnames (or IPs) of systems that run the client (drive the test).

``--samples``
    the number of samples per iteration.

``--max-stddev``
    the percent maximum standard deviation allowed in order to consider the iteration to pass.

``--max-failures``
    the maximum number of failures to achieve the allowed standard deviation.

``--postprocess-only``

``--run-dir``

``--start-iteration-num``

``--tool-label-pattern``

Benchmark-specific options are called out in the following sections for each benchmark.

[XXX: Is this true?] Note that in some of these scripts the default tool group is hard-wired: if you want them to run
a different tool group, you need to edit the script [3]_ . 

7.1 pbench-dbench
~~~~~~~~~~~~~~~~~

``--threads``

7.2 pbench-fio
~~~~~~~~~~~~~~

Iterations are the cartesian product ``targets X test-types X block-sizes``.
More information on many of the following can be obtained from the ``fio`` man page.

``--direct``
    O\_DIRECT enabled or not (1/0) - default is 1.

``--sync``
    O\_SYNC enabled or not (1/0) - default is 0.

``--rate-iops``
    IOP rate not to be exceeded (per job, per client)

``--ramptime``
    seconds - time to warm up test before measurement.

``--block-sizes``
    list of block sizes - default is 4, 64, 1024.

``--file-size``
    fio will create files of this size during the job run.

``--targets``
    file locations (list of directory/block device).

``--job-mode``
    serial/concurrent - default is ``concurrent``.

``--ioengine``
    any IO engine that fio supports (see the fio man page) - default is ``psync``.

``--iodepth``
    number of I/O units to keep in flight against the file.

``--client-file``
    file containing list of clients, one per line.

``--numjobs``
    number of clones (processes/threads performing the same workload) of this job - default is 1.

``--job-file``
    if you need to go beyond the recognized options, you can use a fio job file.

7.3 pbench-linpack
~~~~~~~~~~~~~~~~~~

TBD

7.4 pbench-uperf
~~~~~~~~~~~~~~~~

``--kvm-host``

``--message-sizes``

``--protocols``

``--instances``

``--servers``

``--server-nodes``

``--client-nodes``

``--log-response-times``

7.5 pbench-user-benchmark
~~~~~~~~~~~~~~~~~~~~~~~~~

TBD

8 Utility scripts
-----------------

This section is needed as preparation for the `9 Second steps`_ section below.

PBench uses a bunch of utility scripts to do common operations. There
is a common set of options for some of these: ``--name`` to specify a
tool, ``--group`` to specify a tool group, ``--with-options`` to list or
pass options to a tool, ``--remote`` to operate on a remote host (see
entries in the `15 FAQ`_ section below for more details on these options).

The first set is for registering and unregistering tools and getting
some information about them:

``pbench-list-tools``
    list the tools in the default group or in the
    specified group; with the --name option, list the groups that the
    named tool is in. TBD: how do you list **all** available tools
    whether in a group or not?

``pbench-register-tool-set``
    call ``pbench-register-tool`` on each tool in the default list.

``pbench-register-tool``
    add a tool to a tool group (possibly remotely).

``pbench-clear-tools``
    remove a tool or all tools from a specified tool
    group (including remotely). Used with a ``--name`` option, it replaces ``pbench-unregister-tool``.

The second set is for handling the results and doing cleanup:

``pbench-postprocess-tools``
    run all the relevant postprocessing scripts
    on the tool output - this step also gathers up tool output from
    remote hosts to the local host in preparation for copying it to
    the results repository.

``pbench-clear-results``
    start with a clean slate.

``pbench-copy-results``
    copy results to the results repo.

``pbench-move-results``
    move the results to the results repo and delete
    them from the local host.

XXX ``pbench-cleanup``
    clean up the pbench run directory - after this step,
    you will need to register any tools again.

``pbench-register-tool-set`` and ``pbench-register-tool`` can also
take a ``--remote`` option (see `15.3 What does --remote do?`_) in order to
allow the starting/stopping of tools and the postprocessing of results
on multiple remote hosts.

9 Second steps
--------------

WARNING: It is **highly** recommended that you use one of the ``pbench-<benchmark>``
scripts for running your benchmark. If one does not exist already, you might be
able to use the ``pbench-user-benchmark`` script to run your own script. The advantage
is that these scripts already embody some conventions that pbench and associated
tools depend on, e.g. using a timestamp in the name of the results directory to
make the name unique. If you cannot use ``pbench-user-benchmark`` and a ``pbench-<benchmark>``
script does not exist already, consider writing one or helping us write one. The
more we can encapsulate all these details into generally useful tools, the easier
it will be for everybody: people running it will not need to worry about all these
details and people maintaining the system will not have to fix stuff because the
script broke some assumptions. The easiest way to do so is to crib an existing
``pbench-<benchmark>`` script, e.g ``pbench-fio``.

Once collection tools have been registered, the work flow of a
benchmark script is as follows:

- Process options (see `9.1 Benchmark scripts options`_).

- Check that the necessary prerequisites are installed and if not, install them.

- Iterate over some set of benchmark characteristics
  (e.g. ``pbench-fio`` iterates over a couple test types: read, randread
  and a bunch of block sizes), with each iteration doing the following:

  - create a benchmark\_results directory

  - start the collection tools

  - run the benchmark

  - stop the collection tools

  - postprocess the collection tools data

The tools are started with an invocation of ``pbench-start-tools`` like this:

::

    pbench-start-tools --group=$group --iteration=$iteration --dir=$benchmark_tools_dir

where the group is usually "default" but can be changed to taste as
described above, iteration is a benchmark-specific tag that
disambiguates the separate iterations in a run (e.g. for ``pbench-fio``
it is a combination of a count, the test type, the block size and a
device name), and the benchmark\_tools\_dir specifies where the collection
results are going to end up (see the section for much
more detail on this).

The stop invocation is exactly parallel, as is the postprocessing invocation:

::

    pbench-stop-tools --group=$group --iteration=$iteration --dir=$benchmark_tools_dir
    pbench-postprocess-tools --group=$group --iteration=$iteration --dir=$benchmark_tools_dir

9.1 Benchmark scripts options
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Generally speaking, benchmark scripts do not take any pbench-specific
options except ``--config`` (see below).
Other options tend to be benchmark-specific [4]_ .

9.2 Collection tools options
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``--help`` can be used to trigger the usage message on all of the tools (even though it's
an invalid option for many of them). Here is a list of gotcha's:

- blktrace: you need to pass ``--devices=/dev/sda,/dev/sdb`` when you register the tool:

  ::

      pbench-register-tool --name=blktrace [--remote=foo] -- --devices=/dev/sda,/dev/sdb

  There is no default and leaving it empty causes errors in
  postprocessing (this should be flagged).

9.3 Utility script options
~~~~~~~~~~~~~~~~~~~~~~~~~~

Note that ``pbench-move-results``, ``pbench-copy-results`` and ``pbench-clear-results`` always
assume that the run directory is the default ``/var/lib/pbench-agent``.

``pbench-move-results`` and ``pbench-copy-results`` now (starting with pbench version 0.31-108gf016ed6)
take a ``--prefix`` option. This is explained in the `13.1 Accessing results on the web`_ section
below.

``--remote``
    specify a remote host on which a collection tool (or set of collection tools)
    is to be registered:

    ::

        pbench-register-tool --name=<tool> --remote=<host>

10 Running pbench collection tools with an arbitrary benchmark
--------------------------------------------------------------

If you want to take advantage of pbench's data collection and other
goodies, but your benchmark is not part of the set above (see
`7 Available benchmark scripts`_), or you want to run it differently so
that the pre-packaged script does not work for you, that's no problem
(but, if possible, heed the `9 Second steps`_ above). The various pbench phases
can be run separately and you can fit your benchmark into the
appropriate slot:

::

    group=default
    benchmark_tools_dir=TBD

    pbench-register-tool-set --group=$group
    pbench-start-tools --group=$group --iteration=$iteration --dir=$benchmark_tools_dir
    <run your benchmark>
    pbench-stop-tools --group=$group --iteration=$iteration --dir=$benchmark_tools_dir
    pbench-postprocess-tools --group=$group --iteration=$iteration --dir=$benchmark_tools_dir
    pbench-copy-results

Often, multiple experiments (or "iterations") are run as part of a single run. The modified
flow then looks like this:

::

    group=default
    experiments="exp1 exp2 exp3"
    benchmark_tools_dir=TBD

    pbench-register-tool-set --group=$group
    for exp in $experiments ;do
        pbench-start-tools --group=$group --iteration=$exp
        <run the experiment>
        pbench-stop-tools --group=$group --iteration=$exp
        pbench-postprocess-tools --group=$group --iteration=$exp
    done
    pbench-copy-results

Alternatively, you may be able to use the ``pbench-user-benchmark`` script as follows:

::

    pbench-user-benchmark --config="specjbb2005-4-JVMs" -- my_benchmark.sh

which is going to run ``my_benchmark.sh`` in the ``<run your benchmark>``
slot above. Iterations and such are your responsibility.

``pbench-user-benchmark`` can also be used for a somewhat more specialized
scenario: sometimes you just want to run the collection tools for a
short time while your benchmark is running to get an idea of how the
system looks. The idea here is to use ``pbench-user-benchmark`` to run a sleep
of the appropriate duration in parallel with your benchmark:

::

    pbench-user-benchmark --config="specjbb2005-4-JVMs" -- sleep 10

will start data collection, sleep for 10 seconds, then stop data collection
and gather up the results. The config argument is a tag to distinguish this data
collection from any other: you will probably want to make sure it's unique.

This works well for one-off scenarios, but for repeated usage on well defined phase
changes you might want to investigate `14.1 Triggers`_.

11 Remote hosts
---------------

11.1 Multihost benchmarks
~~~~~~~~~~~~~~~~~~~~~~~~~

Usually, a multihost benchmark is run using a host that acts as the "controller"
of the run. There is a set of hosts on which data collection is to be performed while
the benchmark is running. The controller may or may not be itself part of that set.
In what follows, we assume that the controller has password-less ssh access to the
relevant hosts.

The recommended way to run your workload is to use the generic ``pbench-user-benchmark`` script.
The workflow in that case is:

- Register the collection tools on **each** host in the set:

::

    for host in $hosts ;do
        pbench-register-tool-set --remote=$host
    done

- Invoke ``pbench-user-benchmark`` with your workload generator as argument: that will start the
  collection tools on all the hosts and then run your workload generator; when that
  finished, it will stop the collection tools on all the hosts and then run the postprocessing
  phase which will gather the data from all the remote hosts and run the postprocessing tools
  on everything.

- Run ``pbench-copy-results`` or ``pbench-move-results`` to upload the data to the results server.

If you cannot use the ``pbench-user-benchmark`` script, then the process becomes more manual.
The workflow is:

- Register the collection tools on **each** host as above.

- Invoke ``pbench-start-tools`` on the controller: that will start data collection on
  all of the remote hosts.

- Run the workload generator.

- Invoke ``pbench-stop-tools`` on the controller: that will stop data collection on
  all of the remote hosts.

- Invoke ``pbench-postprocess-tools`` on the controller: that will gather all the data
  from the remotes and run the postprocessing tools on all the data.

- Run ``pbench-copy-results`` or ``pbench-move-results`` to upload the data to the results server.

12 Best practices
-----------------

12.1 Clear results
~~~~~~~~~~~~~~~~~~

The ``pbench-move-results`` script removes the results directory (assumed to be
within the ``/var/lib/pbench-agent`` hierarchy) after copying it the results
repo. But if there are previous results present (perhaps because
``pbench-move-results`` was never invoked, or perhaps because ``pbench-copy-results``
was invoked instead), ``pbench-move-results`` will copy **all** of them: you
probably do not want that.

It's a good idea in general to invoke ``pbench-clear-results``, which cleans
``/var/lib/pbench-agent``, **before** running your benchmark.

12.2 Kill tools
~~~~~~~~~~~~~~~

If you interrupt a built-in benchmark script (or your own script perhaps),
the collection tools are **not** going to be stopped. If you don't stop them
explicitly, they can severely affect subsequent runs that you make. So it
is strongly recommended that you invoke ``pbench-kill-tools`` before you start your
run:

::

    pbench-kill-tools --group=$group

If you run pbench from your own script, you should add a signal handler to
do this:

::

    trap "pbench-kill-tools --group=$group" EXIT INT QUIT

12.3 Clear tools
~~~~~~~~~~~~~~~~

This tool will delete the tools.$group file on the local host as well
as on all the remote hosts specified therein.  After doing that, you
will need to re-register all the tools that you want to use. In
combination with ``pbench-clear-results``, this tool creates a blank slate
where you can start from scratch. You probably don't want to call
this much, but it may be useful in certain cases (e.g. when the
remotes are created for the test and then disappear at the end - it's
a good idea to call ``pbench-clear-tools`` from a trap in that case).

12.4 Register tools
~~~~~~~~~~~~~~~~~~~

Some tools have **required** options [5]_  and you **have** to specify
them when you register the tool. One example is the ``blktrace`` tool
which requires a ``--devices=/dev/sda,dev/sdb=`` argument. ``pbench-register-tool-set``
knows about such options for the default set of tools, but with other
tools, you are on your own.

The trouble is that registration does not invoke the tool and does not
know what options are required. So the best thing to do is invoke the
tool with ``--help``: the ``--help`` option may or may not be recognized
by any particular tool, but either way you should get a usage message
that labels required options. You can then register the tool by using
an invocation similar to:

::

    pbench-register-tool --name=blktrace -- --devices=/dev/sda,/dev/sdb

12.5 Using ``--dir``
~~~~~~~~~~~~~~~~~~~~

If you use the tool scripts explicitly, specify ``--dir=/var/lib/pbench-agent/<run-id>``
so that all the data are collected in the specified directory. Also, save any data
that your benchmark produces inside that directory: that way, ``pbench-move-results``
can move everything to the results warehouse.

Make the ``<run-id>`` as detailed as possible to disambiguate results. The built-in
benchmark scripts use the following form: ``<benchmark>_<config>-<ts>``, e.g

::

    fio_bagl-16-4-ceph_2014-12-15_15:58:51

where the ``<config>`` part (``bagl-16-4-ceph``) comes from the ``--config`` option and
can be as detailed as you want to make it.

12.6 Using ``--remote``
~~~~~~~~~~~~~~~~~~~~~~~

If you are running multihost benchmarks, we strongly encourage you to set up the
tool collections using ``--remote``. Choose a driver host (which might or might not
participate in the tool data collection: in the first case, you register tools locally
as well as remotely; in the second, you just register them remotely) and run everything
from it. During the data collection phase, everything will be pulled off the remotes and
copied to the driver host, so it can be moved to the results repo as a single unit.
Consider also using ``--label`` to label sets of hosts - see `12.7 Using ``--label```_ for more information.

12.7 Using ``--label``
~~~~~~~~~~~~~~~~~~~~~~

When you register remotes, ``--label`` can be used to give a meaningful
label to the results subdirectories that come from remote hosts. For
example, use =--label=server" (or client, or vm, or capsule or
whatever else is appropriate for your use case).

13 Results handling
-------------------

13.1 Accessing results on the web
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This section describes how to get to your results using a web browser. It describes
how ``pbench-move-results`` moves the results from your local controller to a centralized
location and what happens there. It also describes the ``--prefix`` option to ``pbench-move-results``
(and ``pbench-copy-results``) and a utility script, ``pbench-edit-prefix``, that allows you to change how
the results are viewed.

**N.B.** This section applies to the pbench RPM version 0.31-108gf016ed6 and later. If you are
using an earlier version, please upgrade at your earliest convenience.

13.1.1 Where to go to see results :internal:
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The canonical place is

`http://pbench.perf.lab.eng.bos.redhat.com/results/ <http://pbench.perf.lab.eng.bos.redhat.com/results/>`_

There are subdirectories there for each controller host (the host on
which ``pbench-move-results`` was executed) and underneath those, there are
subdirectories for each pbench run.

The leaves of the hierarchy are actually symlinks that point to the
corresponding results directory in the old, flat ``incoming/``
hierarchy. Direct access to ``incoming/`` is now deprecated (and will
eventually go away).

The advantage is that the ``results/`` hierarchy can be manipulated to
change one's view of the results [6]_ , while leaving the ``incoming/``
hierarchy intact, so that tools manipulating it can assume a fixed
structure.

In the interim, a simple script is running once an hour creating any
missing links from ``results/`` to ``incoming/``. It will be turned off
eventually after everybody has upgraded to this or a later version
of pbench.

13.1.2 Where to go to see results :external:
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Where ``pbench-move/copy-results`` copies the results is site-dependent. Check with
the admin who set up the pbench server and provided you with the configuration file
for the ``pbench-agent`` installation.

14 Advanced topics
------------------

14.1 Triggers
~~~~~~~~~~~~~

Triggers are groups of tools that are started and stopped on specific events.
They are registered with ``pbench-register-tool-trigger`` using the ``--start-trigger``
and ``--stop-trigger`` options. The output of the benchmark is piped into the
``pbench-tool-trigger`` tool which detects the conditions for starting and stopping
the specified group of tools.

There are some commands specifically for triggers:

``pbench-register-tool-trigger``
    register start and stop triggers for a tool group.

``pbench-list-triggers``
    list triggers and their start/stop criteria.

``pbench-tool-trigger``
    this is a Perl script that looks for the
    start-trigger and end-trigger markers in the benchmark's output,
    starting and stopping the appropriate group of tools when it
    finds the corresponding marker.

As an example, ``pbench-dbench`` uses three groups of tools: warmup, measurement
and cleanup. It registers these groups as triggers using

::

    pbench-register-tool-trigger --group=warmup --start-trigger="warmup" --stop-trigger="execute"
    pbench-register-tool-trigger --group=measurement --start-trigger="execute" --stop-trigger="cleanup"
    pbench-register-tool-trigger --group=cleanup --start-trigger="cleanup" --stop-trigger="Operation"

It then pipes the output of the benchmark into ``pbench-tool-trigger``:

::

    $benchmark_bin --machine-readable --directory=$dir --timelimit=$runtime
                   --warmup=$warmup --loadfile $loadfile $client |
    	           tee $benchmark_results_dir/result.txt |
                   pbench-tool-trigger "$iteration" "$benchmark_results_dir" no

``pbench-tool-trigger`` will then start the warmup group when it encounters the
string "warmup" in the benchmark's output and stop it when it
encounters "execute". It will also start the measurement group when it
encounters "execute" and stop it when it encounters "cleanup" - and so
on.

Obviously, the start/stop conditions will have to be chosen with some
care to ensure correct actions.

15 FAQ
------

15.1 What does --name do?
~~~~~~~~~~~~~~~~~~~~~~~~~

This option is recognized by ``pbench-register-tool`` and ``pbench-unregister-tool``: it
specifies the name of the tool that is to be (un)registered. ``pbench-list-tools``
with the ``--name`` option list all the groups that contain the named tool [7]_ .

15.2 What does --config do?
~~~~~~~~~~~~~~~~~~~~~~~~~~~

This option is recognized by the benchmark scripts (see `7 Available benchmark scripts`_ above) which use it as a tag for the directory where the benchmark is
going to run. The default value is empty.  The run directory for the benchmark
is constructed this way:

::

    $pbench_run/${benchmark}_${config}_$date

where ``$pbench_run`` and ``$date`` are set by the ``/opt/pbench-agent/base`` script
and ``$benchmark`` is set to the obvious value by the benchmark script; e.g. a
fio run with config=foo would run in the directory
``/var/lib/pbench-agent/fio_foo_2014-11-10_15:47:04``.

15.3 What does --remote do?
~~~~~~~~~~~~~~~~~~~~~~~~~~~

pbench can register tools on remote hosts, start them and stop them remotely and gather up
the results from the remote hosts for post-processing. The model is that one has a controller
or orchestrator and a bunch of remote hosts that participate in the benchmark run.

The pbench setup is as follows: ``pbench-register-tool-set`` or ``pbench-register-tool``
is called on the controller with the ``--remote`` option, once for each
remote host:

::

    for remote in $remotes ;do
        pbench-register-tool-set --remote=$remote --label=foo --group=$group
    done

15.4 What does --label do?
~~~~~~~~~~~~~~~~~~~~~~~~~~

TBD

15.5 How to add a benchmark
~~~~~~~~~~~~~~~~~~~~~~~~~~~

TBD

15.6 How do I collect data for a short time while my benchmark is running?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Running

::

    pbench-user-benchmark -- sleep 60

will start whatever data collections are specified in the default tool
group, then sleep for 60 seconds. At the end of that period, it will
stop the running collections tools and postprocess the collected data.
Running ``pbench-move-results`` afterwards will move the results to the results
server as usual.

15.7 I have a script to run my benchmark - how do I use it with pbench?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

pbench is a set of building blocks, so it allows you to use it in many different
ways, but it also makes certain assumptions which if not satisfied, lead to problems.

Let's assume that you want to run a number of ``iozone`` experiments, each with different
parameters. Your script probably contains a loop, running one experiment each time around.
If you can change your script so that it executes **one** experiment specified by an argument,
then  the best way is to use the ``pbench-user-benchmark`` script:

::

    pbench-register-tool-set
    for exp in experiment1 experiment2 experiment3 ;do
        pbench-user-benchmark --config $exp -- my-script.sh $exp
    done
    pbench-move-results

The results are going to end up in directories named ``/var/lib/pbench-agent/pbench-user-benchmark_$exp_$ts``
for each experiment (unfortunately, the timestamp will be recalculated at the beginning of
each ``pbench-user-benchmark`` invocation), before being uploaded to the results server.

Alternatively, you can modify your script so that each experiment is wrapped with start/stop/postprocess-tools
and then call ``pbench-move-results`` at the end:

::

    pbench-register-tool-set
    dir=/var/lib/pbench-agent
    tool_group=default
    typeset -i iter=1
    for exp in experiment1 experiment2 experiment3 ;do
        pbench-start-tools --group=$tool_group --dir=$dir --iteration=$iter
        <run the experiment>
        pbench-stop-tools --group=$tool_group --dir=$dir --iteration=$iter
        pbench-postprocess-tools --group=$tool_group --dir=$dir --iteration=$iter
        iter=$iter+1
    done
    pbench-move-results

**N.B.** You need to invoke the ``pbench-{start,stop,postprocess}-tools`` scripts
with the **same** arguments.

15.8 How do I install pbench-agent?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

See the `installation instructions <./installation.rst>`_.


.. [1] The current version of pbench-agent yum installs prebuilt RPMs of
    various common benchmarks: dbench, fio, iozone, linpack, smallfile and uperf,
    as well as the most recent version of the sysstat tools. We are planning to
    add more benchmarks to the list: iperf, netperf, streams, maybe the phoronix
    benchmarks. If you want some other benchmark (AIM7?), let us know.

.. [2] The "standard storage location" is site-dependent. Check with the admin
    who set up the pbench server at your site.

.. [3] That will be handled by a configuration file in the future.

.. [4] It is probably better to bundle these options in a configuration file,
    but that's still WIP.

.. [5] Yes, I know: it's an oxymoron.

.. [6] E.g. A performance engineer was NFS-mounting the ``incoming/``
    hierarchy, grouping his results under separate subdirectories for fio, iozone
    and smallfile, and grouping them further under thematically created
    subdirectories ("baremetal results for this configuration", "virtual host
    results under that configuration" etc.), primarily because having them all in
    a single directory was slow, as well as confusing. There were two problems
    with this approach which motivated the prefix approach described above. One
    was that the NFS export of the FUSE mount of the gluster volume that houses
    the result is extremetly flakey. The other is that the ``incoming/`` hierarchy
    is modified, which makes the writing of tools to extract data harder: they
    have to figure out arbitrary changes, instead of being able to assume a fixed
    structure.

.. [7] A list of available tools in a specific group can be obtained with the
    ``--group`` option of ``pbench-list-tools``; unfortunately, there is no option to list
    all available tools - the current workaround is to check the contents of
    ``/opt/pbench-agent/tool-scripts``.
