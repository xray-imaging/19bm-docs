===============
Data management
===============

Data ownership and per-experiment user access at 19-BM are managed by
`DMagic <https://dmagic.readthedocs.io/en/latest/index.html>`_, which reads
the APS scheduling system and the experiment ESAF, creates the experiment on
**Sojourner** (the APS Data Management store), shares it with the proposal
and ESAF users over Globus, and updates the tomoscan user-info PVs.

.. note::

   Run DMagic from **radon**. The Data Management subcommands need
   ``/dm/19bm``, which is mounted there but not on ``hounsfield`` --
   so this is the one task that does not move with the control screen,
   which is opened on ``hounsfield`` (see :doc:`../manual/item_003`).


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

Current state, as of 2026-11-18:

- ``/gdata`` **is** mounted on ``factdetector`` (``nvgerfs``, 23 P), and
  the camera IOC writes straight into the experiment directory::

      /gdata/dm/19BM/<yyyy-mm>/<yyyy-mm>-<PIlastname>-<GUP#>/data/

  So raw data now lands on Sojourner as it is collected. The earlier
  commissioning path ``/local/vieworks_test`` is no longer in use.
- ``/gdata`` is **not** mounted on ``radon``. DMagic runs there, because
  the Data Management subcommands need ``/dm/19bm``, so the host that
  manages the experiment cannot see the files the detector writes.
- ``dmagic.conf`` names ``tomodata2`` as ``data-host`` and
  ``/data2/19BM`` as ``data-top-dir``, which is not where data is
  actually being written.

Reconciling the DMagic configuration with the direct ``/gdata`` mount --
and deciding whether ``daq-start`` / ``upload`` are needed at all now
that the detector writes to Sojourner directly -- is commissioning work
still to be done.


Configuration
=============

``~/dmagic.conf`` holds the site settings. The ones worth knowing:

================================  ==========================================
``beamline``                      ``19-BM-D``
``experiment-type``               ``19BM``
``globus-server-uuid``            ``5a7ce40f-75c4-4bdd-a1e7-8246054afc02``
``globus-message-file``           ``message-19bm.txt``
``tomoscan-prefix``               ``19bm:TomoScan:``
``data-host``                     ``tomodata2``
``data-top-dir``                  ``/data2/19BM/``
``dm-direct-mount``               ``True``
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
