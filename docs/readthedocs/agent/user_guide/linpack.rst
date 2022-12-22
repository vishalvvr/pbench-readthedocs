=========================
Installing pbench-linpack
=========================


.. contents::



1 Introduction
--------------

The ``linpack`` RPM we use with the ``pbench-linpack`` bench-script
wrapper is a perculiar beast.  It is not built from source: it is an
Intel executable that was packaged up as a tarball and we built RPMs
from that. In particular, it is old, it only works on x86\_64 and it
cannot be stored in a publicly accessible repo. We recently expunged
it from the Fedora COPR repo. We have rebuilt it in the `internal COPR
repo <https://copr.devel.redhat.com/coprs/ndokos/pbench/>`_ for all the RHEL platforms (RHEL7, RHEL8 and RHEL9).  This note
describes what you have to do to install it, assuming that the node
you are installing on is inside the Red Hat firewall. Installing it
outside the firewall is not supported.

2 Repo installation and checking
--------------------------------

You will need to install a repo file in ``/etc/yum.repos.d/`` on the node(s)
where you want to install the RPM. You can get the repo file from
`https://copr.devel.redhat.com/coprs/ndokos/pbench/ <https://copr.devel.redhat.com/coprs/ndokos/pbench/>`_ (N.B. there is an extraneous
``rhel-8.dev`` chroot in that list that has been **deleted**, but COPR wants to keep it around
for another four days for some reason: it should go away by <2022-02-20 Sun>, but you should
not use it, since there is nothing available there).

Install the repo file under ``/etc/yum.repos.d`` and do

.. code:: shell

    dnf --refresh list pbench-linpack

to make sure you can get the RPM.

If you have **any** difficulty, please post questions in the ``pbench``
GChat channel or send mail to `mailto:pbench-team@redhat.com <mailto:pbench-team@redhat.com>`_.
