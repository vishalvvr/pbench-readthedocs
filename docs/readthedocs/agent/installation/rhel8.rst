====================================
Installing the Pbench Agent on RHEL8
====================================


.. contents::



1 Introduction
--------------

This note describes what is needed in order to install the
pbench-agent on a set of remote nodes that are running RHEL8 (the
"remotes"). The way to do that is to run a bunch of Ansible playbooks
on some host (the "ansible host"). The ansible host could be running
any version of RHEL or Fedora, as long as it can run ansible
playbooks.

There is no difference in outward appearance between installing on
RHEL8 and installing on RHEL9 (or RHEL7 for that matter). This
document describes the underlying differences, but then refers to the
`RHEL9 <./rhel9.rst>`_ document for the details.

The primary quirk of RHEL8 is that there is no python installation to
begin with. The repo installation playbook and roles are written
specifically to not require python for their operation; instead, they
use low-level functionality to boostrap a set of repos onto the
remotes.

After the repos have been installed, the ``quirks-boostrap`` playbook is
run to deal with any special issues. In the case of RHEL8, python is
installed to allow the normal operation of future playbooks (e.g. the
pbench installation playbook).

At this point in time however, the Subscription Management repos are
to be preferred over the ad-hoc repos that the boostrap playbooks
perform. If you have a choice, use Subscription Management.

For the detailed commands, please refer to the `RHEL9 <rhel9.rst>`_
instructions. They apply mutatis-mutandis to RHEL8 and RHEL7 (but in
both cases, we recommend that you use Subscription Management instead).
