==========================
PBench server installation
==========================


.. contents::



1 Introduction
--------------

Server installation is done through two packages: an externally
available, upstream package containing all the relevant scripts and an
internal package that contains the local configuration files [1]_ . The
upstream bits include a script that the internal package calls to
install the local config files in the appropriate places. It also
creates a crontab from the information in the configuration files
which needs to be installed manually and creates the top level
directories needed to store results.

The server RPM consists of the scripts that are run (currently from cron)
on the server side: they include scripts to check, unpack and archive the
incoming tarballs, reorganize the results hierarchy, index the results into
ElasticSearch, etc.

In addition to the generic bits, some localization is needed. This is
discussed in some detail below.

2 Overview
----------

A user running the pbench agent produces results that are stored
locally, until the user calls ``pbench-move-results``. The results are then
packaged up as a compressed tarball which is copied to the server.
Once there, the server scripts are run through ``cron`` to unpack
the tarball and store the results for later visualization. Additional
processing (which is not described in this document any further) might
include indexing results, creating backups of the tarballs, and sending
sosreports on to other services for further processing.

3 Prerequisites
---------------

The host that runs the server has to run apache. It should also
have a fairly hefty filesystem for storing results. Of course, the
more results you produce, the bigger the filesystem should be.

In our setup, we have a machine that runs apache and mounts a 27TB
gluster volume on /pbench. For experimental purposes, a filesystem
on a SATA/SAS disk with capacity of a few TB should be more than enough.

In addition, we index the results into ElasticSearch. This setup
is beyond the scope of this document.

If your local pbench admins provide an "internal" RPM available, you should use it
for installation: it takes care of the drudgery of installing the "external" RPM
and then customizing the installation with the local settings.

If there is no "internal" RPM available, install the "external" RPM (see `4 Server installation (generic) :external:`_ for details).
You will then have to customize the installation by hand (see `4.2 Customizing the installation with the local settings.`_ for details).

4 Server installation (generic) :external:
-------------------------------

4.1 Installing the RPM
~~~~~~~~~~~~~~~~~~~~~~

The repo is the standard `COPR pbench repo <https://copr.fedorainfracloud.org/coprs/ndokos/pbench/>`_, installed with

::

    dnf copr enable ndokos/pbench

or manually if dnf is not available - see the `COPR pbench page <https://copr.fedorainfracloud.org/coprs/ndokos/pbench/>`_ for instructions.

The server is installed normally:

::

    dnf install pbench-server
    # or
    # yum install pbench-server

That creates a ``pbench`` user with home directory ``/home/pbench`` if not
present already. The server bits can be installed anywhere, but the
default location is ``/opt/pbench-server``: scripts under
``/opt/pbench-server/bin``, libraries, config files, crontabs under
``/opt/pbench-server/lib``.

4.2 Customizing the installation with the local settings.
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The home directory of the ``pbench`` user is only used for ssh purposes:
the public key of the ``pbench agent`` is copied in to allow ``pbench-move-results``
to be able to copy tarballs to the server without requiring a password.

The installation directory (``/opt/pbench-server`` by default) holds all
the scripts, config files and the crontab. There may be more than one
config file, depending on what services the server provides, but this
document only describes the basic setup of the server, for which the
config file is ``/opt/pbench-server/lib/config/pbench-server.cfg``
(there is another config file,
``/opt/pbench-server/lib/config/pbench-server-default.cfg``, that
contains settings that are independent of any particular setup. You
should not modify this file at all).

4.3 Config files
~~~~~~~~~~~~~~~~

Example config files are installed under
``/opt/pbench-server/lib/config``. You should make copies of these
files:

::

    cp pbench-server.cfg.example pbench-server.cfg
    cp pbench-index.cfg.example pbench-index.cfg

and modify them appropriately (``pbench-index.cfg`` is optional - see
below). The minimal changes required are marked with ``# CHANGE ME``
comments: basically, hostnames and e-mail addresses will need to be
modified for your environment.

In addition, you will need to decide which roles and tasks to enable
under cron.  The default roles are ``pbench-results`` and
``pbench-backup``.

The ``pbench-results`` role consists of a number of tasks: unpacking
incoming tarballs (task ``pbench-unpack-tarballs``), copying sosreports
(task ``pbench-copy-sosreports``), indexing metadata (and real-soon-now
results) into ElasticSearch (task ``pbench-index``) and reorganizing
results on user demand (task ``pbench-edit-prefixes``). The minimal set
is unpacking the tarballs and reorganizing results. The rest are
optional and not covered in this document. In particular, if you don't
run the ``pbench-index`` task, you don't need to worry about the
``pbench-index.cfg`` file above.

The ``pbench-backup`` role consists of two tasks: actually backing up the
tarballs (task ``pbench-backup-tarballs``) and verifying the integrity
of the tarballs (task ``pbench-verify-tarballs``).

You might want to save the config file(s) in some safe place for future
reference. If you need to reinstall the ``pbench-server`` RPM, you can
then just generate the rest of the setup from the saved config file
as described in the next section.

4.4 The rest of the setup
~~~~~~~~~~~~~~~~~~~~~~~~~

Let us assume you now have a saved ``pbench-server.cfg`` file in some
safe place. The rest of the setup goes as follows:

::

    PATH=/opt/pbench-server/bin:$PATH
    pbench-server-config-activate /path/to/saved/pbench-server.cfg
    pbench-server-activate /opt/pbench-server/lib/config/pbench-server.cfg

The first script copies the config file(s) to the standard place
``/opt/pbench-server/lib/config/``. N.B. the name ``pbench-server.cfg`` is
fixed: there **must** be a file of that name in
``/opt/pbench-server/lib/config/`` at the end of this step and it is
**the** config file that is used in the second step, and is made
available to the cron jobs.

The second step consists of a number of substeps:

- Create the crontab, based on the roles and tasks defined in the
  config file. The crontab is **not** activated: you should examine it
  carefully and, assuming that it passes muster, activate it (see below).

- Create the results host info structure that the agent depends on to
  send results to the server.

- Create a directory structure to store results, by default under
  ``/pbench``. It is up to you to make sure that there is enough space
  there for the results that will be generated by your users.

The final step is to manually activate the crontab (as user ``pbench``,
**not** as root):

::

    su - pbench
    crontab /opt/pbench-server/lib/crontab/crontab
    # or ...
    crontab -u pbench /opt/pbench-server/lib/crontab/crontab

5 Server installation (internal RPM) :internal:
------------------------------------

The internal RPM contains config files tailored for our internal
installation.  It depends on the generic RPM, so when it (the internal
RPM) is installed, the generic RPM is installed as well. You need to
install the internal repo, in addition to the COPR pbench repo:

::

    dnf copr enable pbench/ndokos
    wget -O /etc/yum.repos.d/pbench.repo http://pbench.perf.lab.eng.bos.redhat.com/repo/yum.repos.d/pbench.repo

Alternatively (and preferably), you can use the Ansible scripts in the ``perf-dept`` repo
to install the repos:

::

    ansible-playbook -i <inventory file> /path/to/Ansible/script/pbench-repo-install.yml

See the `Ansible section of the PBench agent installation guide <../agent/installation.rst>`_ for more details.
The installation step is now

::

    dnf install pbench-server-internal

As noted above, the dependency is going to take care of installing the generic
RPM. In addition, the internal RPM is going to set up the internal config files
at the proper place, and generate a crontab. It does **not** activate the crontab.
You are supposed to do that after checking it carefully:

::

    su - pbench
    crontab /opt/pbench-server/lib/crontab/crontab

6 Testing
---------

You should then test the whole shebang by setting up a ``pbench-agent``
(see `PBench agent installation <../agent/installation.rst>`_), running a simple benchmark and moving
the results to the server (see the `PBench agent user guide <../agent/user-guide.rst>`_ for
details).  Watch the log files on the server (``/pbench-local/logs`` and
subdirs thereof by default) to make sure that all stages of processing
are correctly done.


.. [1] Although this document describes installation in terms of an "internal"
    package, note that that may be a convenient fiction. If there **is** one available,
    then installing it should take care of the config files and the rest of the setup
    described in `4 Server installation (generic) :external:`_.
