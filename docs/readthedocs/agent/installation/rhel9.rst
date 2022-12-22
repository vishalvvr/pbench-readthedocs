====================================
Installing the Pbench Agent on RHEL9
====================================


.. contents::



1 Introduction
--------------

This note describes what is needed in order to install the
pbench-agent on a set of remote nodes that are running RHEL9 (the
"remotes"). The way to do that is to run a bunch of Ansible playbooks
on some host (the "ansible host").  The ansible host **could** be
running RHEL9, but it's more likely to be running an earlier version
of RHEL or a version of Fedora.

IMPORTANT: the 0.69 version of ``pbench-agent`` does **NOT** work on RHEl9.
There is a bug in ``scp -r`` which prevents you from sending results to the
server. We have worked around the problem in version 0.71, but we are not
planning to backport the workaround to v0.69: the only supported version
on RHEL9 is v0.71 - that's the only version available in the COPR pbench-repo
for RHEL9.

2 Overview
----------

Here's a simple example script, without any explanations. The rest of
the document expands on these steps, providing details and explanations:

.. code:: shell

    mkdir pbenchrhel9
    cd pbenchrhel9/

    git config --global http.sslVerify false
    git clone https://code.engineering.redhat.com/gerrit/perf-dept .
    git config --global http.sslVerify true

    ansible-galaxy role install geerlingguy.repo-epel --force
    ansible-galaxy collection install pbench.agent --force

    cat - <<HEREDOC > inventory
    [servers]
    myserver.example.com

    [servers:vars]
    #where to get the key
    pbench_key_url = http://git.app.eng.bos.redhat.com/git/perf-dept.git/plain/bench/pbench/agent/{{ pbench_configuration_environment }}/ssh

    #where to get the config file
    pbench_config_url = http://git.app.eng.bos.redhat.com/git/perf-dept.git/plain/bench/pbench/agent/{{ pbench_configuration_environment }}/config
    HEREDOC

    ansible-playbook -i inventory perf-dept/sysadmin/Ansible/repo-bootstrap.yml
    ansible-playbook -i inventory perf-dept/sysadmin/Ansible/quirks-bootstrap.yml
    ansible-playbook -i inventory perf-dept/sysadmin/Ansible/epel-repo-install.yml
    export ANSIBLE_ROLES_PATH=$HOME/.ansible/collections/ansible_collections/pbench/agent/roles:$ANSIBLE_ROLES_PATH
    ansible-playbook -i inventory perf-dept/sysadmin/Ansible/pbench-agent-install.yml

3 Prerequisites
---------------

On the ansible host, you will have to make sure that the following
prerequisites (described in detail in the following sections) are met:

- You need to ``git clone`` the ``perf-dept`` repo. If you have cloned it
  previously, make sure that you update your clone with the latest
  bits. See the `3.1 Clone/Update the perf-dept repo`_ section below for details.

- You need to install a few Ansible role collections from Ansible Galaxy.

- You need to set up an inventory file containing all the remotes (which
  are in general going to be the nodes in your SUT).

- You need to set up password-less ssh from the ansible host to the
  **root** account of each of the remotes.

**N.B. The ansible host has got to be inside the firewall. You will also need to enable the VPN in order to access the various resources needed. Even if your SUT is e.g. on AWS, the ansible host has to be inside the firewall. Our suggestion would be that you do this on your laptop and connect to the VPN: you will want to keep these things around for future installations in general. For a different SUT, all you will need to do is create a new inventory file and set up ssh access to the root accounts of the new remotes.**

Each of these prerequisites is covered in its own subsection in the
remainder of this section. Once the prerequisites have been dealt
with, you can proceed to the `4 Installation of pbench-agent on RHEL9`_ section. Note
the first three of these steps will only need to be done once (well,
from time to time you might have to update the Galaxy roles or the
``perf-dept`` repo to pick up the latest changes, but generally
speaking, these are relatively rare occurrences).

3.1 Clone/Update the perf-dept repo
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We assume that you've created a working directory and we refer to it as ``$WORKDIR`` in what follows:

.. code:: shell

    export WORKDIR=/path/to/working/directory

You may need to turn off SSL [1]_  temporarily to be able to clone or
update the perf-dept repo:

.. code:: shell

    git config --global http.sslVerify false

In the working directory, either clone the ``perf-repo``

.. code:: shell

    cd ${WORKDIR}
    git clone https://code.engineering.redhat.com/gerrit/perf-dept

or, if you have cloned it previously, just update it to the latest bits:

.. code:: shell

    git remote update --prune
    git rebase

The Ansible playbooks that will be used below are all in ``$WORKDIR/perf-dept/sysadmin/Ansible``.

3.2 Install collections from Ansible Galaxy
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Install the following collections from Ansible Galaxy [2]_ :

.. code:: shell

    ansible-galaxy role install geerlingguy.repo-epel
    ansible-galaxy collection install pbench.agent

You might have installed earlier versions of these already. If so, you
will need to update them:

.. code:: shell

    ansible-galaxy role install geerlingguy.repo-epel --force
    ansible-galaxy collection install pbench.agent --force

The docs say that ``--update`` is needed, but at least my version of
Galaxy does not recognize the ``--update`` option.

3.3 Set up an inventory file
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The inventory file should list all the nodes in the SUT under a
``[servers]`` heading, plus the variables that define where to get the config and key files -
that should be sufficient for now:

::

    [servers]
    dhcp31-111.perf.example.com
    dhcp31-112.perf.example.com
    dhcp31-113.perf.example.com

    [servers:vars]
    pbench_key_url:  http://git.app.eng.bos.redhat.com/git/perf-dept.git/plain/bench/pbench/agent/{{ pbench_configuration_environment }}/ssh
    pbench_config_url: http://git.app.eng.bos.redhat.com/git/perf-dept.git/plain/bench/pbench/agent/{{ pbench_configuration_environment }}/config

A convenient place to put the inventory file is in a
``$WORKDIR/Inventories`` directory.

3.4 Set up password-less ssh to the root account of all the remotes.
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Make sure that ``root`` is allowed to login with a password: that's a
temporary measure to allow you to set up keys. On each remote, as
root, edit the ``/etc/ssh/sshd_config`` file and uncomment/modify/add
the line that defines ``PermitRootLogin`` to say:

::

    PermitRootLogin yes

Make sure to restart ``sshd``:

.. code:: shell

    systemctl restart sshd

Then from your ansible host, install your public key on each remote:

.. code:: shell

    ssh-copy-id root@<remote_node>

After you have set up keys and tested that it all works as it should,
you can comment out ``PermitRootLogin`` or switch it to a more
restrictive setting.  Don't forget to restart sshd!

4 Installation of pbench-agent on RHEL9
---------------------------------------

There are currently (<2022-01-12 Wed>) no subscription management
repos for RHEL9, so we need to install a set of repos to provide the
basics. After that, we can install EPEL and then the pbench-agent.

4.1 Bootstrap the nodes with a basic set of repos
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

N.B. RHEL9 standard installations currently fail to install the
``python3-libselinux`` package and that makes running playbooks
that access selinux-enabled remote nodes impossible. The workaround
below involves running a low-level playbook that installs the
relevant package (it's ``python3-libselinux`` on RHEL9, but its
name has varied historically).

Assuming that the variable ``inv`` has been set to the path of your
inventory file, run the following playbook (the playbooks
are found in ``${WORKDIR}/perf-dept/sysadmin/Ansible``).

.. code:: shell

    ansible-playbook -i ${inv} ${WORKDIR}/perf-dept/sysadmin/Ansible/repo-bootstrap.yml

This will install an ad-hoc set of repos where RHEL9 packages can be
found. This is a temporary, stop-gap measure: eventually, when the
subscription management repos become available, they can be used
instead, but for now, this is all we've got.

**N.B. These repos are inside the firewall: you will not be able to use them to install on nodes outside the firewall**. How to deal with
package requirements outside the firewall is beyond the scope of these
notes.

To allow "normal" playbooks to run, you will have to install the ``python3-libselinux``
package on each remote. You can use the following playbook to do so if desired, or you
can do it manually:

.. code:: shell

    ansible-playbook -i ${inv} ${WORKDIR}/perf-dept/sysadmin/Ansible/quirks-bootstrap.yml

In addition, there are some dependencies that need EPEL for their
resolution, so the EPEL repo must be installed as well. That's done by
yet another playbook:

.. code:: shell

    ansible-playbook -i ${inv} ${WORKDIR}/perf-dept/sysadmin/Ansible/epel-repo-install.yml

4.2 Run the pbench-agent installation playbook
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Once there are repos that can provide all the necessary dependencies,
then ``pbench-agent`` can be installed. You need to tell Ansible where
to find the roles that were installed above, by setting the
environment variable ANSIBLE\_ROLES\_PATH (this is a good one to put
into your ``.profile`` or similar: you'll always need it) and then run the
following playbook which first installs the ``pbench.repo`` file and
then uses it to install the ``pbench-agent`` RPM:

.. code:: shell

    export ANSIBLE_ROLES_PATH=${HOME}/.ansible/collections/ansible_collections/pbench/agent/roles:${ANSIBLE_ROLES_PATH}
    ansible-playbook -i ${inv} ${WORKDIR}/perf-dept/sysadmin/Ansible/pbench-agent-install.yml

In addition to the ``pbench-agent`` RPM (plus the client-specific config
file), the playbook also installs a pbench-compatible older version of
the ``sysstat`` package (the package is called ``pbench-sysstat``).  If
you don't use the playbook, you will have to install this package (as
well as the pbench-agent package and the appropriate config file) manually. [3]_ 

N.B. the agent does **NOT** install benchmarks like ``fio`` or ``uperf``:
the packages are available, either in the pbench COPR repo if
necessary or, preferably, built by the distro in one of its standard
repos, or available in EPEL.  However, it is **your** responsibility to
install these benchmarks: pbench will complain if the benchmark that
you want to run is not installed, but it will **not** try to install it.


.. [1] That should only be done temporarily. After cloning the repo
    you should turn SSL acces on globally: ``git config --global sslVerify true`` and then turn it off only in repos that need it: ``git config sslVerify false``.  Even that would allow MITM attacks when you access
    the particular repo, but since that is internal to Red Hat, the danger
    is mitigated somewhat. Do not leave it off globally!

.. [2] If your ansible host runs RHEL9, you will need to install the
    ``ansible.posix`` collection as well. That collection was bundled with
    ansible in earlier versions of Ansible, but on RHEL9 many of the
    formerly built-in modules have been unbundled. As far as we know,
    there is no RPM package for it, so you have to get it from
    Galaxy. BTW, the module is needed when you are installing a 0.71 (or
    later) agent: the Tool Meister implementation requires opening a
    couple of ports on the pbench-agent controller host.

.. [3] ``pbench-sysstat`` is one of the packages that we still have to build for
    backwards compatibility. There are also a few dependencies
    (e.g. ``screen``) that, for various reasons, are not available in **any**
    repo. We generally build them (if possible) and make them available
    in the same repo where the ``pbench-agent`` package is made available.
    That way, the installation of the ``pbench.repo`` makes all these
    packages available. Since we are using COPR, these packages are
    available on the web outside the firewall. OTOH, building packages is
    a pain, so that's a task we would like to jettison.
