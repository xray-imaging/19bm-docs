===============
Data management
===============

Data ownership and per-experiment user access at 19-BM are managed by
`DMagic <https://dmagic.readthedocs.io/en/latest/index.html>`_, which reads
the APS scheduling system and the experiment ESAF, creates the experiment on
**Sojourner** (the APS Data Management store), shares it with the proposal
and ESAF users over Globus, and updates the tomoscan user-info PVs.

.. note::

   Run DMagic from **radon**, the 19-BM production console.


Everyday use
============

Tag the current experiment -- this is what fills in the user name, email and
proposal fields that end up in the HDF5 metadata::

  (base) [factuser@radon]$ bash
  (base) [factuser@radon]$ conda activate dm
  (dm)   [factuser@radon]$ dmagic show
  (dm)   [factuser@radon]$ dmagic tag

``dmagic show`` prints the active proposal and the resolved configuration
without changing anything; run it first. ``dmagic tag`` then writes the
user-info PVs.

You can also type the user last name, email and ``YYYY-MM`` directly into
the MEDM screen if the scheduling system does not have what you need.

If the tomoscan prefix ever needs overriding::

  (dm) [factuser@radon]$ dmagic show --tomoscan-prefix 19bm:TomoScan: --set 0


Subcommands
===========

Read-only, safe to run any time:

``show``
   Print today's proposal and the resolved configuration.
``list-beamtimes``
   All beamtimes scheduled on the beamline in a date range.
``list-esafs``
   ESAFs for the station in a date range.
``list-users``
   Users with access to a DM experiment.
``daq-status``
   Running DM DAQs for the station.
``collected``
   For each scheduled beamtime, the matching DM experiment (or
   ``(no DM experiment)``), joined by GUP number.

Write operations -- these create records, move data or send mail:

``create`` / ``delete``
   Create or remove a DM experiment on Sojourner.
``add-user`` / ``remove-user``
   Change access on an existing experiment, by badge number.
``daq-start`` / ``daq-stop``
   Upload existing files and keep syncing new ones as they arrive.
``upload``
   One-shot upload of everything present -- the fallback for when
   ``daq-start`` was not running during collection.
``email``
   Send the Globus data-access message to every user on the experiment.
   Uses ``dmagic/message-19bm.txt``.


Storage
=======

The canonical home for each experiment is Sojourner, at::

  /gdata/dm/19BM/<yyyy-mm>/<yyyy-mm>-<PIlastname>-<GUP#>/
      data/      raw acquisition
      analysis/  reconstructions
      system/    DMagic-internal state

A GUP number of ``0`` means internal beamtime rather than a user proposal.

.. warning::

   **The 19-BM acquisition-to-Sojourner pipeline is not yet established.**
   What follows is the state as of commissioning, not a description of
   routine operation.

Sojourner is **not** directly mounted at 19-BM -- ``dm-direct-mount = False``
in ``~/dmagic.conf``, and no host checked (radon, tomo3) has ``/gdata``.
Data therefore reaches Sojourner through ``dmagic daq-start`` or
``dmagic upload``, not by copying to ``/gdata``.

Current state:

- The camera IOC writes to ``/local/vieworks_test`` on the detector host.
  This is a commissioning path, not a data path.
- ``/data2/19BM`` exists on **tomo3** (``tomodata2-ib:/data2``, 253 T, 43 %
  used) and is empty. This is the configured ``data-top-dir``.
- ``tomo3`` is the configured ``data-host``. It has Infiniband-fast access
  to ``/data2`` but no ``/gdata``.

Settling the staging route between the detector host and ``/data2/19BM``,
and where reconstruction runs, is commissioning work still to be done.


Configuration
=============

``~/dmagic.conf`` holds the site settings. The ones worth knowing:

================================  ==========================================
``beamline``                      ``19-BM-D``
``experiment-type``               ``19BM``
``globus-server-uuid``            ``5a7ce40f-75c4-4bdd-a1e7-8246054afc02``
``globus-message-file``           ``message-19bm.txt``
``tomoscan-prefix``               ``19bm:TomoScan:``
``data-host``                     ``tomo3``
``data-top-dir``                  ``/data2/19BM/``
``dm-direct-mount``               ``False``
================================  ==========================================

Scheduling credentials live in ``~/.scheduling_credentials`` as a single
``username|password`` line, mode ``600``. They authenticate against
``https://beam-api.aps.anl.gov`` and carry 19-BM authority.

The software is a conda environment ``dm`` under ``~/miniconda3``, holding
DMagic itself (editable, from ``~/conda/dmagic-decarlof``) and the APS DM
SDK (``aps-dm-api``, a conda package -- it is not on PyPI).

.. note::

   The ``DM_*`` variables in ``~/.bashrc`` must be **exported**. The SDK
   reads them from the environment and silently falls back to
   ``127.0.0.1`` when they are absent, so a bare ``DM_DAQ_WEB_SERVICE_URL=``
   assignment leaves the import working, prints no warning, and fails at
   the first DAQ call with
   ``Could not list DAQs: URL https://127.0.0.1:33336 refused connection.``
