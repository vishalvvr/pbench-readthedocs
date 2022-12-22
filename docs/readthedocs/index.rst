.. pbench documentation master file, created by
   sphinx-quickstart on Fri Nov  6 22:27:57 2020.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Pbench
########

A Benchmarking and Performance Analysis Framework.

Pbench Agent
============

The Agent is responsible for providing commands for running benchmarks across one or more systems, 
while properly collecting the configuration of those systems, their logs, and specified telemetry from various tools (sar, vmstat, perf, etc).

Pbench Server
=============
The second sub-system included here is the Server, which is responsible for archiving results and indexing them to
allow the dashboard to prepare visualizations of the results.

Pbench Dashboard
================
Lastly, the Dashboard is used to display visualizations in graphical and other forms of the results that were collected
by the Agent and indexed by the Server.

..  toctree::
    :caption: Pbench Agent
    :hidden:

    readthedocs/agent/installation/bootstrapping
    readthedocs/agent/installation/rhel8
    readthedocs/agent/installation/rhel9
    readthedocs/agent/faq/faq
..  toctree::
    :caption: Pbench Server
    :hidden:

    readthedocs/server/deployment/index

..  toctree::
    :caption: Pbench Dashboard
    :hidden:

    readthedocs/dashboard/index

..  toctree::
    :caption: Developers Guide
    :hidden:

    readthedocs/developer/CONTRIBUTING.md
