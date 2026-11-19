==================
Starting the IOCs
==================

None of the 19-BM IOCs start at boot. After a machine reboot they must
each be started by hand, and nothing on the control screen will populate
until they are.

Where they run
==============

.. list-table::
   :header-rows: 1
   :widths: 24 20 56

   * - IOC
     - Host
     - What it drives
   * - ``19bmSoft``
     - ``hounsfield``
     - Rotation stage, white-beam slits, filter banks, optics stages
   * - ``19bmHXP``
     - ``hounsfield``
     - Hexapod (Aerotech Automation1)
   * - ``19bmBLEPS``
     - ``hounsfield``
     - Equipment protection readback
   * - ``19bmVieworks``
     - ``factdetector``
     - Vieworks VP-61MX camera
   * - ``ORCA:``
     - ``factdetector``
     - Hamamatsu ORCA Fire camera
   * - ``tomoScanApp``
     - ``factdetector``
     - tomoScan IOC and its Python server

Order of events after a reboot
==============================

**1. Wait about three minutes for the directory service.**

``caDirServ`` on ``hounsfield`` is the one thing that does start
automatically, from a ``@reboot`` cron entry -- but that entry sleeps 180
seconds first:

.. code-block:: none

    @reboot sleep 180 && /APSshare/bin/caDirServ

Until it is up, PVs on the private subnet cannot be resolved and every
screen comes up white. That is expected. Check with::

    ssh hounsfield 'pgrep -a caDirServ'

**2. Start the IOCs.**

``softIOC`` is the single command for all of them. Run it on the host
that owns the IOC:

.. code-block:: bash

    ssh factuser@hounsfield
    softIOC 19bmSoft  start
    softIOC Hexapod   start
    softIOC 19bmBLEPS start

It takes ``start``, ``stop``, ``restart``, ``status`` and ``console``,
and knows ``19bmSoft``, ``19bmBLEPS``, ``Hexapod``, ``LabJack``,
``Vieworks``, ``TomoScan`` and ``TomoScan_Python`` among others.

The detector host has wrapper scripts in ``~/bin`` which do the same
thing over ``ssh``, so they work from any machine -- these are also what
the buttons on the control screen call:

.. code-block:: bash

    detector_ORCA.sh                 /  detector_IOC_ORCA_stop.sh
    detector_IOC_vieworks_61mp.sh    /  ..._stop.sh
    tomoscan_IOC.sh                  /  tomoscan_IOC_stop.sh
    tomoscan_server.sh               /  tomoscan_server_stop.sh
    bleps_IOC.sh                     [start|stop|status|console]
    hexapod_IOC.sh                   [start|stop|status|console]

**3. Check they answer.**

Do not trust exit status -- read a PV::

    caget 19bmSoft:m1.RBV 19bmHXP:m2.RBV ORCA:cam1:Acquire \
          19bm:TomoScan:ServerRunning

.. note::

   ``screen -ls`` on the IOC host lists the running soft IOCs. To leave
   a console again press ``^a d`` -- **not** ``^c``, which stops the IOC.

   The ORCA camera IOC is the exception: it is a Python program running
   in the foreground of its own terminal, not a soft IOC, so it has no
   screen session and is stopped with ``^C`` or its stop script.

If the hexapod lost power
=========================

A computer reboot does not need this, but if the Automation1 controller
itself was power-cycled the IOC coming back is not enough -- the hexapod
must be initialised::

    ~/bin/start_hexapod_after_power_cycle

Related
=======

For the static hardware reference, see `Beamline components
<https://img.xray.aps.anl.gov/source/internal/19bm/19bm_002.html>`__.
For structured procedures, see `Procedures
<https://img.xray.aps.anl.gov/source/internal/19bm/Procedures/procedures_001.html>`__.
