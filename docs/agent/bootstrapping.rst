==========================
Installing necessary repos
==========================


.. contents::



1 Introduction
--------------

A brand new installation of RHEL[789] might not have **any** repo files
installed. There are two methods to populate the ``/etc/ym.repos.d/``
directory with repo files: use a subscription or install an ad-hoc set
of repo files using an ansible playbook. You might also need to
install EPEL. Each of these is described in the next three sections.

**N.B.** If you use the method in the `1.1 Subscription manager`_ section
below, do **NOT** use the installation described in the `1.2 Internal installation`_ section - and vice versa: use one or the other, **not**
both.

1.1 Subscription manager
~~~~~~~~~~~~~~~~~~~~~~~~

**N.B.** If you use the method in this section, do **NOT** use the
installation described in the `1.2 Internal installation`_ section below.

This is the preferred method of installing repos for RHEL7 or RHEL8.
Currently (<2021-11-19 Fri>), this does **not** work for RHEL9, so using
`1.2 Internal installation`_ is the only option for RHEL9. However, this **will**
change before the RHEL9 release, so this note will become obsolete.

You will need some sort of credenttials to use a subscription. For
most of us, that means a developer account: you need a user id and a
password in order to register a new machine with a subscription
server.

N.B. You need root privileges to run ``subscription-manager``. You'll
either have to provide the root password on each invocation, or use
``sudo``.

In each case, you will need to install the EPEL repos as well. You
can use the ansible playbook that is used for `1.2.5 Run the ansible playbook to install the EPEL repos`_,
but you might not want to do all the internal setup just for this step,
in which case you can use `these instructions on the Fedara Project site <https://docs.fedoraproject.org/en-US/epel/#_quickstart>`_.

1.1.1 RHEL7
^^^^^^^^^^^

.. code:: shell

    subscription-manager register
    # enter user id and password when prompted
    subscription-manager attach
    subscription-manager repos
    # list the repos

1.1.2 RHEL8
^^^^^^^^^^^

The same commands as above are used in RHEL8, but in addition to the
"normal" repos that you get with the commands above, you also need to
enable the CRB repo and the ansible repo explicitly:

.. code:: shell

    subscription-manager repos --enable codeready-builder-for-rhel-8-x86_64-rpms
    subscription-manager repos --enable ansible-2.9-for-rhel-8-x86_64-rpms

1.1.3 RHEL9
^^^^^^^^^^^

Subscription management repos are not yet available for RHEL9 as
of <2022-01-31 Mon>, so the instructions for `1.2 Internal installation`_
below should be followed for now.

1.2 Internal installation
~~~~~~~~~~~~~~~~~~~~~~~~~

**N.B.** If you use the method in the `1.1 Subscription manager`_ section
above, do **NOT** use the installation described in this section.

Many internal installations do not use the subscription manager. In
this case, we provide some more-or-less standard repo files for each
platform. The easiest way to copy them to the remote is to clone the
perf-dept repo, create an inventory file and use an ansible playbook.

The following sections provide an overview but you should consult the
separate instructions for `RHEL8 <rhel8.rst>`_ and `rhel9.rst <rhel9.rst>`_ for more details, specific
issues and gotchas.

1.2.1 Clone the perf-dept repo
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code:: shell

    git clone https://code.engineering.redhat.com/gerrit/perf-dept /path/to/perf-dept

If this fails with the message

::

    fatal: unable to access 'https://code.engineering.redhat.com/gerrit/perf-dept/': Peer's certificate issuer has been marked as not trusted by the user.

the current (temporary) workaround is to disable SSL verification by git:

.. code:: shell

    git config --global http.sslVerify false

and then trying the ``git clone`` again. You should also turn SSL verification back on afterwards:

.. code:: shell

    git config --global http.sslVerify true

1.2.2 Create an inventory file
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

I create them in ``~/.config/Inventory/`` but they can be anywhere.
Create a file ``repo-bootstrap.hosts`` with contents like this:

.. code:: shell

    [servers]
    host1
    host2
    host3

The hosts can be running RHEL7, RHEL8 or RHEL9.

You **must** set up passwordless ssh between the machine where you will
run the ansible playbook and all the machines in your inventory file,
not only for your own user id, but also for root. In all cases, the
remote user should be "root".

.. code:: shell

    ssh-copy-id root@host1
    sudo ssh-copy-id root@host1
    ssh-copy-id root@host2
    sudo ssh-copy-id root@host2
    ssh-copy-id root@host3
    sudo ssh-copy-id root@host3

You may have to create keys for your user id and/or for root on your machine,
before executing the above.

1.2.3 Run the ansible playbook to install the repos
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code:: shell

    inv=/path/to/inventory/repo-bootstrap.hosts
    cd /path/to/perf-dept
    cd sysadmin/Ansible
    ansible-playbook  --user=root -i ${inv} repo-bootstrap.yml

1.2.4 Run the ansible playbook to fix particular quirks
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code:: shell

    inv=/path/to/inventory/repo-bootstrap.hosts
    cd /path/to/perf-dept
    cd sysadmin/Ansible
    ansible-playbook  --user=root -i ${inv} quirks-boostrap.yml

1.2.5 Run the ansible playbook to install the EPEL repos
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code:: shell

    inv=/path/to/inventory/repo-bootstrap.hosts
    cd /path/to/perf-dept
    cd sysadmin/Ansible
    ansible-playbook  --user=root -i ${inv} epel-repo-install.yml
