==================================================
Beamline Equipment Protection System (BLEPS)
==================================================

**BLEPS** (Beamline Equipment Protection System) is the PLC-based
safety and interlock system used at the APS to protect beamline
hardware — optics, vacuum components, water-cooled stops — from
physical damage during user experiments. It continuously monitors
the operating state of every protected component and, if any of
them deviates from its safe envelope, either warns staff or
trips a **Fault** condition that automatically closes the main
beamline shutters to block the X-ray beam.

BLEPS protects **equipment**; the **Personnel Safety System (PSS)**
protects **people**. The two are distinct but cooperate — a BLEPS
fault asks the PSS-controlled shutter chain to close. PSS is the
authoritative safety layer for people-in-hutch interlocks; BLEPS
is the layer between PSS and the wear-and-tear of the hardware.
The 19-BM PSS scope is documented separately under ICMS
APS_1181415; see the *Personnel safety system* section of
:doc:`item_001`.


What BLEPS monitors
===================

Typical quantities continuously read by the PLC and compared against
component-specific limits:

- **Water flow** on every water-cooled element. At 19-BM the
  synchrotron-radiation-intercepting components are all
  water-cooled (per the APSU BM thermal waiver, ICMS
  APSU_2286907): the exit mask ``A359-M20``, the white-beam
  slits and the filter unit in 19-BM-A, and — on a single
  series loop — the 250 µm Be window at the 19-BM-D entrance
  together with the downstream copper photon stop ``A359-M100``.
- **Vacuum pressure** at the ion pumps and ion gauges along the
  front end and the 19-BM-A → 19-BM-C → 19-BM-D transport,
  grouped into two interlocked vacuum sections.
- **Position / state** of the beamline isolation valve, the
  front-end shutter and the 19-BM-A gate valve.

Each input has a **Warning** threshold (notify staff, scan
continues) and a **Fault** threshold (close shutters, latch the
fault until manually cleared).

.. note::

   Because the Be window and the photon stop share **one cooling
   loop in series**, a single loss-of-flow event trips both
   protections at once — see :doc:`item_001`.


Response to a fault
===================

When a Fault threshold is crossed, BLEPS:

1. Latches the fault on the PLC and propagates it to the PSS
   shutter chain.
2. Drops the ``BIV_PERMIT`` and ``FES_PERMIT`` outputs, closing
   the **Beamline Isolation Valve (BIV)** and the **Front End
   Shutter (FES)** to block the X-ray beam, in accordance with
   standard APS policy. The 19-BM-A gate valve ``GV1``, which
   isolates 19-BM-A from the remainder of the beamline, is
   interlocked to the same system.
3. Drives the beacon lights and buzzer (``RED_LIGHT`` /
   ``YELLOW_LIGHT`` / ``GREEN_LIGHT`` / ``BUZZER``) and surfaces
   the latched fault on the BLEPS operator screen, so staff can
   identify the root cause before clearing.

The fault is **latched** — the shutters do not re-open automatically
when the offending parameter returns to normal. A staff member must
verify the underlying condition, acknowledge the fault (via
``FAULT_RESET`` / ``TRIP_RESET``, or the physical BLEPS panel
button), and then re-arm the shutter chain.


19-BM implementation
====================

The full 19-BM-specific BLEPS implementation — interlock matrix
(which inputs trip which outputs), PLC programme, operator
screens, fault-recovery procedure — is documented in:

`19-BM BLEPS implementation
<https://anl.box.com/s/2nlednd6d8e2n3085wj7ylsxoa4hegkz>`__.

The governing requirements document is the **19-BM Beamline
Equipment Protection System (BLEPS) User Requirements Document**,
ICMS **APS_2388098**.

Those documents are the source of truth for the 19-BM interlock
configuration. This page is the conceptual overview plus the
deployed tag inventory.


Scope of the 19-BM configuration
================================

19-BM is a considerably smaller BLEPS installation than a
multi-station beamline such as 2-BM, because the beam path has
no mirror, no monochromator and no second station shutter. The
channel counts actually enabled in the transfer table are:

.. list-table::
   :header-rows: 1
   :widths: 34 16 50

   * - Channel class
     - Count
     - Tags
   * - Cooling-water flows
     - 6
     - ``Flow1``..``Flow6``
   * - Temperatures (thermocouples)
     - 0
     - none enabled
   * - Gate valves
     - 1
     - ``GV1``
   * - Shutters / isolation valves
     - 2
     - ``BIV``, ``FES``
   * - Ion pumps
     - 4
     - ``IP1``..``IP4``
   * - Ion gauges
     - 3
     - ``IG1``..``IG3``
   * - Vacuum sections
     - 2
     - ``VS1``, ``VS2``

.. important::

   **No temperature channels are enabled at 19-BM.** The
   ``Temps`` worksheet exists in the master but every row is
   unflagged, so there are no ``TEMPn_CURRENT`` /
   ``TEMPn_SET_POINT`` PVs and no thermocouple trips. Equipment
   protection for the water-cooled components rests entirely on
   the six flow interlocks.

Two naming differences are worth noting for anyone porting
scripts from another beamline:

- 19-BM uses ``Fault_Exists`` / ``Trip_Exists`` and
  ``FAULT_RESET`` / ``TRIP_RESET`` — **without** the ``A_``
  prefix used at some other beamlines
  (``A_Fault_Exists``, ``A_FAULT_RESET``).
- The beacon lights and buzzer are the only outputs that do
  **not** use the ``BL:`` prefix — they are published under
  ``P$(L):$(xx)$(yy):`` rather than ``BL:$(xx)$(yy):``.

A ``PLC_Battery_Dead_Wrn`` warning channel is enabled at 19-BM.


Not yet documented
==================

Two things are deliberately absent from this page because they
are not carried in the transfer table:

- **Physical assignment of the flow channels.** The table refers
  to the cooling loops only by index (``Flow1``..``Flow6``). Which
  physical component each index monitors is configured in the PLC
  and read off the Allen-Bradley operator panel, not the
  spreadsheet.
- **EPICS IOC deployment.** Host, account, IOC directory,
  concrete PV prefix and the start / stop wrapper scripts are a
  property of the soft IOC that bridges the PLC to Channel
  Access, and are not part of the PLC tag set.


BLEPS PV inventory (19-BM)
==========================

The Excel transfer table that defines the 19-BM BLEPS PLC tag set
contains one worksheet per logical class of signals (FIFOs, Faults,
Trips, Warnings, Info, Flows, Temps, Inputs, Outputs, Display,
EPICS_Inputs). Each row has a ``Used`` flag that marks whether
the row is in the deployed configuration; the subsections below
reproduce **only the rows currently in use** at 19-BM — 329 of
the 1 020 rows in the master — and give the two fields needed to
address each one over Channel Access:

* **EPICS Ethernet Tag** — the controller-side tag that the PLC
  publishes on the EtherNet/IP-to-EPICS gateway.
* **Short Description** — the human-readable label the PLC team
  attached to each row.

Source of truth is the Excel master
(``BLEPS EPICS Transfer Table Master 19BM.xls``, last saved
2026-08-19), itself part of the wider 19-BM BLEPS document set on
Box (see the Box link in the *19-BM implementation* section
above). The full PV name on the EPICS gateway is the
``Base Name`` + ``PV Name`` template
``BL:$(xx)$(yy):<base name in caps>``, which is also in the
master Excel.

.. note::

   The ``Temps`` worksheet has no rows in use and is therefore
   omitted below.

.. warning::

   In the master template the ``Flow5.Scaling_Factor`` row
   carries the PV name ``BL:$(xx)$(yy):FLOW5_SET_POINT`` — the
   same PV as ``Flow5.Set_Point`` — although its ``Base Name``
   column correctly reads ``Flow5_Scaling``. This looks like a
   copy/paste error inherited from the shared master (the same
   collision is present in the 2-BM table) rather than a 19-BM
   configuration choice. Expect the scaling factor for Flow 5 to
   be unreachable under the name the table implies.

``FIFOs`` (210 entries)
-----------------------

.. list-table::
   :header-rows: 1
   :widths: 45 55

   * - EPICS Ethernet Tag
     - Short Description
   * - ``Faults.Number[0]``
     - Fault Number 01
   * - ``Faults.Number[1]``
     - Fault Number 02
   * - ``Faults.Number[2]``
     - Fault Number 03
   * - ``Faults.Number[3]``
     - Fault Number 04
   * - ``Faults.Number[4]``
     - Fault Number 05
   * - ``Faults.Number[5]``
     - Fault Number 06
   * - ``Faults.Number[6]``
     - Fault Number 07
   * - ``Faults.Number[7]``
     - Fault Number 08
   * - ``Faults.Number[8]``
     - Fault Number 09
   * - ``Faults.Number[9]``
     - Fault Number 10
   * - ``Faults.Year[0]``
     - Fault Year 01
   * - ``Faults.Year[1]``
     - Fault Year 02
   * - ``Faults.Year[2]``
     - Fault Year 03
   * - ``Faults.Year[3]``
     - Fault Year 04
   * - ``Faults.Year[4]``
     - Fault Year 05
   * - ``Faults.Year[5]``
     - Fault Year 06
   * - ``Faults.Year[6]``
     - Fault Year 07
   * - ``Faults.Year[7]``
     - Fault Year 08
   * - ``Faults.Year[8]``
     - Fault Year 09
   * - ``Faults.Year[9]``
     - Fault Year 10
   * - ``Faults.Month[0]``
     - Fault Month 01
   * - ``Faults.Month[1]``
     - Fault Month 02
   * - ``Faults.Month[2]``
     - Fault Month 03
   * - ``Faults.Month[3]``
     - Fault Month 04
   * - ``Faults.Month[4]``
     - Fault Month 05
   * - ``Faults.Month[5]``
     - Fault Month 06
   * - ``Faults.Month[6]``
     - Fault Month 07
   * - ``Faults.Month[7]``
     - Fault Month 08
   * - ``Faults.Month[8]``
     - Fault Month 09
   * - ``Faults.Month[9]``
     - Fault Month 10
   * - ``Faults.Day[0]``
     - Fault Day 01
   * - ``Faults.Day[1]``
     - Fault Day 02
   * - ``Faults.Day[2]``
     - Fault Day 03
   * - ``Faults.Day[3]``
     - Fault Day 04
   * - ``Faults.Day[4]``
     - Fault Day 05
   * - ``Faults.Day[5]``
     - Fault Day 06
   * - ``Faults.Day[6]``
     - Fault Day 07
   * - ``Faults.Day[7]``
     - Fault Day 08
   * - ``Faults.Day[8]``
     - Fault Day 09
   * - ``Faults.Day[9]``
     - Fault Day 10
   * - ``Faults.Hour[0]``
     - Fault Hour 01
   * - ``Faults.Hour[1]``
     - Fault Hour 02
   * - ``Faults.Hour[2]``
     - Fault Hour 03
   * - ``Faults.Hour[3]``
     - Fault Hour 04
   * - ``Faults.Hour[4]``
     - Fault Hour 05
   * - ``Faults.Hour[5]``
     - Fault Hour 06
   * - ``Faults.Hour[6]``
     - Fault Hour 07
   * - ``Faults.Hour[7]``
     - Fault Hour 08
   * - ``Faults.Hour[8]``
     - Fault Hour 09
   * - ``Faults.Hour[9]``
     - Fault Hour 10
   * - ``Faults.Minute[0]``
     - Fault Minute 01
   * - ``Faults.Minute[1]``
     - Fault Minute 02
   * - ``Faults.Minute[2]``
     - Fault Minute 03
   * - ``Faults.Minute[3]``
     - Fault Minute 04
   * - ``Faults.Minute[4]``
     - Fault Minute 05
   * - ``Faults.Minute[5]``
     - Fault Minute 06
   * - ``Faults.Minute[6]``
     - Fault Minute 07
   * - ``Faults.Minute[7]``
     - Fault Minute 08
   * - ``Faults.Minute[8]``
     - Fault Minute 09
   * - ``Faults.Minute[9]``
     - Fault Minute 10
   * - ``Faults.Second[0]``
     - Fault Second 01
   * - ``Faults.Second[1]``
     - Fault Second 02
   * - ``Faults.Second[2]``
     - Fault Second 03
   * - ``Faults.Second[3]``
     - Fault Second 04
   * - ``Faults.Second[4]``
     - Fault Second 05
   * - ``Faults.Second[5]``
     - Fault Second 06
   * - ``Faults.Second[6]``
     - Fault Second 07
   * - ``Faults.Second[7]``
     - Fault Second 08
   * - ``Faults.Second[8]``
     - Fault Second 09
   * - ``Faults.Second[9]``
     - Fault Second 10
   * - ``Trips.Number[0]``
     - Trip Number 01
   * - ``Trips.Number[1]``
     - Trip Number 02
   * - ``Trips.Number[2]``
     - Trip Number 03
   * - ``Trips.Number[3]``
     - Trip Number 04
   * - ``Trips.Number[4]``
     - Trip Number 05
   * - ``Trips.Number[5]``
     - Trip Number 06
   * - ``Trips.Number[6]``
     - Trip Number 07
   * - ``Trips.Number[7]``
     - Trip Number 08
   * - ``Trips.Number[8]``
     - Trip Number 09
   * - ``Trips.Number[9]``
     - Trip Number 10
   * - ``Trips.Year[0]``
     - Trip Year 01
   * - ``Trips.Year[1]``
     - Trip Year 02
   * - ``Trips.Year[2]``
     - Trip Year 03
   * - ``Trips.Year[3]``
     - Trip Year 04
   * - ``Trips.Year[4]``
     - Trip Year 05
   * - ``Trips.Year[5]``
     - Trip Year 06
   * - ``Trips.Year[6]``
     - Trip Year 07
   * - ``Trips.Year[7]``
     - Trip Year 08
   * - ``Trips.Year[8]``
     - Trip Year 09
   * - ``Trips.Year[9]``
     - Trip Year 10
   * - ``Trips.Month[0]``
     - Trip Month 01
   * - ``Trips.Month[1]``
     - Trip Month 02
   * - ``Trips.Month[2]``
     - Trip Month 03
   * - ``Trips.Month[3]``
     - Trip Month 04
   * - ``Trips.Month[4]``
     - Trip Month 05
   * - ``Trips.Month[5]``
     - Trip Month 06
   * - ``Trips.Month[6]``
     - Trip Month 07
   * - ``Trips.Month[7]``
     - Trip Month 08
   * - ``Trips.Month[8]``
     - Trip Month 09
   * - ``Trips.Month[9]``
     - Trip Month 10
   * - ``Trips.Day[0]``
     - Trip Day 01
   * - ``Trips.Day[1]``
     - Trip Day 02
   * - ``Trips.Day[2]``
     - Trip Day 03
   * - ``Trips.Day[3]``
     - Trip Day 04
   * - ``Trips.Day[4]``
     - Trip Day 05
   * - ``Trips.Day[5]``
     - Trip Day 06
   * - ``Trips.Day[6]``
     - Trip Day 07
   * - ``Trips.Day[7]``
     - Trip Day 08
   * - ``Trips.Day[8]``
     - Trip Day 09
   * - ``Trips.Day[9]``
     - Trip Day 10
   * - ``Trips.Hour[0]``
     - Trip Hour 01
   * - ``Trips.Hour[1]``
     - Trip Hour 02
   * - ``Trips.Hour[2]``
     - Trip Hour 03
   * - ``Trips.Hour[3]``
     - Trip Hour 04
   * - ``Trips.Hour[4]``
     - Trip Hour 05
   * - ``Trips.Hour[5]``
     - Trip Hour 06
   * - ``Trips.Hour[6]``
     - Trip Hour 07
   * - ``Trips.Hour[7]``
     - Trip Hour 08
   * - ``Trips.Hour[8]``
     - Trip Hour 09
   * - ``Trips.Hour[9]``
     - Trip Hour 10
   * - ``Trips.Minute[0]``
     - Trip Minute 01
   * - ``Trips.Minute[1]``
     - Trip Minute 02
   * - ``Trips.Minute[2]``
     - Trip Minute 03
   * - ``Trips.Minute[3]``
     - Trip Minute 04
   * - ``Trips.Minute[4]``
     - Trip Minute 05
   * - ``Trips.Minute[5]``
     - Trip Minute 06
   * - ``Trips.Minute[6]``
     - Trip Minute 07
   * - ``Trips.Minute[7]``
     - Trip Minute 08
   * - ``Trips.Minute[8]``
     - Trip Minute 09
   * - ``Trips.Minute[9]``
     - Trip Minute 10
   * - ``Trips.Second[0]``
     - Trip Second 01
   * - ``Trips.Second[1]``
     - Trip Second 02
   * - ``Trips.Second[2]``
     - Trip Second 03
   * - ``Trips.Second[3]``
     - Trip Second 04
   * - ``Trips.Second[4]``
     - Trip Second 05
   * - ``Trips.Second[5]``
     - Trip Second 06
   * - ``Trips.Second[6]``
     - Trip Second 07
   * - ``Trips.Second[7]``
     - Trip Second 08
   * - ``Trips.Second[8]``
     - Trip Second 09
   * - ``Trips.Second[9]``
     - Trip Second 10
   * - ``Warnings.Number[0]``
     - Warning Number 01
   * - ``Warnings.Number[1]``
     - Warning Number 02
   * - ``Warnings.Number[2]``
     - Warning Number 03
   * - ``Warnings.Number[3]``
     - Warning Number 04
   * - ``Warnings.Number[4]``
     - Warning Number 05
   * - ``Warnings.Number[5]``
     - Warning Number 06
   * - ``Warnings.Number[6]``
     - Warning Number 07
   * - ``Warnings.Number[7]``
     - Warning Number 08
   * - ``Warnings.Number[8]``
     - Warning Number 09
   * - ``Warnings.Number[9]``
     - Warning Number 10
   * - ``Warnings.Year[0]``
     - Warning Year 01
   * - ``Warnings.Year[1]``
     - Warning Year 02
   * - ``Warnings.Year[2]``
     - Warning Year 03
   * - ``Warnings.Year[3]``
     - Warning Year 04
   * - ``Warnings.Year[4]``
     - Warning Year 05
   * - ``Warnings.Year[5]``
     - Warning Year 06
   * - ``Warnings.Year[6]``
     - Warning Year 07
   * - ``Warnings.Year[7]``
     - Warning Year 08
   * - ``Warnings.Year[8]``
     - Warning Year 09
   * - ``Warnings.Year[9]``
     - Warning Year 10
   * - ``Warnings.Month[0]``
     - Warning Month 01
   * - ``Warnings.Month[1]``
     - Warning Month 02
   * - ``Warnings.Month[2]``
     - Warning Month 03
   * - ``Warnings.Month[3]``
     - Warning Month 04
   * - ``Warnings.Month[4]``
     - Warning Month 05
   * - ``Warnings.Month[5]``
     - Warning Month 06
   * - ``Warnings.Month[6]``
     - Warning Month 07
   * - ``Warnings.Month[7]``
     - Warning Month 08
   * - ``Warnings.Month[8]``
     - Warning Month 09
   * - ``Warnings.Month[9]``
     - Warning Month 10
   * - ``Warnings.Day[0]``
     - Warning Day 01
   * - ``Warnings.Day[1]``
     - Warning Day 02
   * - ``Warnings.Day[2]``
     - Warning Day 03
   * - ``Warnings.Day[3]``
     - Warning Day 04
   * - ``Warnings.Day[4]``
     - Warning Day 05
   * - ``Warnings.Day[5]``
     - Warning Day 06
   * - ``Warnings.Day[6]``
     - Warning Day 07
   * - ``Warnings.Day[7]``
     - Warning Day 08
   * - ``Warnings.Day[8]``
     - Warning Day 09
   * - ``Warnings.Day[9]``
     - Warning Day 10
   * - ``Warnings.Hour[0]``
     - Warning Hour 01
   * - ``Warnings.Hour[1]``
     - Warning Hour 02
   * - ``Warnings.Hour[2]``
     - Warning Hour 03
   * - ``Warnings.Hour[3]``
     - Warning Hour 04
   * - ``Warnings.Hour[4]``
     - Warning Hour 05
   * - ``Warnings.Hour[5]``
     - Warning Hour 06
   * - ``Warnings.Hour[6]``
     - Warning Hour 07
   * - ``Warnings.Hour[7]``
     - Warning Hour 08
   * - ``Warnings.Hour[8]``
     - Warning Hour 09
   * - ``Warnings.Hour[9]``
     - Warning Hour 10
   * - ``Warnings.Minute[0]``
     - Warning Minute 01
   * - ``Warnings.Minute[1]``
     - Warning Minute 02
   * - ``Warnings.Minute[2]``
     - Warning Minute 03
   * - ``Warnings.Minute[3]``
     - Warning Minute 04
   * - ``Warnings.Minute[4]``
     - Warning Minute 05
   * - ``Warnings.Minute[5]``
     - Warning Minute 06
   * - ``Warnings.Minute[6]``
     - Warning Minute 07
   * - ``Warnings.Minute[7]``
     - Warning Minute 08
   * - ``Warnings.Minute[8]``
     - Warning Minute 09
   * - ``Warnings.Minute[9]``
     - Warning Minute 10
   * - ``Warnings.Second[0]``
     - Warning Second 01
   * - ``Warnings.Second[1]``
     - Warning Second 02
   * - ``Warnings.Second[2]``
     - Warning Second 03
   * - ``Warnings.Second[3]``
     - Warning Second 04
   * - ``Warnings.Second[4]``
     - Warning Second 05
   * - ``Warnings.Second[5]``
     - Warning Second 06
   * - ``Warnings.Second[6]``
     - Warning Second 07
   * - ``Warnings.Second[7]``
     - Warning Second 08
   * - ``Warnings.Second[8]``
     - Warning Second 09
   * - ``Warnings.Second[9]``
     - Warning Second 10


``Faults`` (20 entries)
-----------------------

.. list-table::
   :header-rows: 1
   :widths: 45 55

   * - EPICS Ethernet Tag
     - Short Description
   * - ``Fault_Exists``
     - Any Fault Exists
   * - ``Communications_Fault``
     - 1 Communications Fault
   * - ``Flow1.Over_Range_Fault``
     - 11 Flow 1 Over Range Fault
   * - ``Flow2.Over_Range_Fault``
     - 12 Flow 2 Over Range Fault
   * - ``Flow3.Over_Range_Fault``
     - 13 Flow 3 Over Range Fault
   * - ``Flow4.Over_Range_Fault``
     - 14 Flow 4 Over Range Fault
   * - ``Flow5.Over_Range_Fault``
     - 15 Flow 5 Over Range Fault
   * - ``Flow6.Over_Range_Fault``
     - 16 Flow 6 Over Range Fault
   * - ``BIV.Fail_to_Close``
     - 100 BIV Fail to Close
   * - ``FES.Fail_to_Close``
     - 101 FES Fail to Close
   * - ``GV1.Faulted``
     - GV1_Faulted
   * - ``GV1.Fault.Both_Switch``
     - 1011 GV1 Both Switch
   * - ``GV1.Fault.Opened_Switch``
     - 1012 GV1 Opened Switch
   * - ``GV1.Fault.Fail_to_Open``
     - 1013 GV1 Fail to Open
   * - ``GV1.Fault.Fully_Open``
     - 1014 GV1 Fail To Fully Open
   * - ``GV1.Fault.Closed_Switch``
     - 1015 GV1 Closed Switch
   * - ``GV1.Fault.Fail_to_Close``
     - 1016 GV1 Fail to Close
   * - ``GV1.Fault.Fully_Close``
     - 1017 GV1 Fail To Fully Close
   * - ``GV1.Fault.Beam_Exposure``
     - 1018 GV1 Beam Exposure
   * - ``GV1.Fault.No_Switch``
     - 1019 GV1 No Switch


``Trips`` (9 entries)
---------------------

.. list-table::
   :header-rows: 1
   :widths: 45 55

   * - EPICS Ethernet Tag
     - Short Description
   * - ``Trip_Exists``
     - Any Trip Exists
   * - ``Flow1.Below_Set_Point_Trip``
     - 11 Flow 1 Below Set Point
   * - ``Flow2.Below_Set_Point_Trip``
     - 12 Flow 2 Below Set Point
   * - ``Flow3.Below_Set_Point_Trip``
     - 13 Flow 3 Below Set Point
   * - ``Flow4.Below_Set_Point_Trip``
     - 14 Flow 4 Below Set Point
   * - ``Flow5.Below_Set_Point_Trip``
     - 15 Flow 5 Below Set Point
   * - ``Flow6.Below_Set_Point_Trip``
     - 16 Flow 6 Below Set Point
   * - ``VS1.Trip``
     - 101 Vacuum Section 1
   * - ``VS2.Trip``
     - 102 Vacuum Section 2


``Warnings`` (18 entries)
-------------------------

.. list-table::
   :header-rows: 1
   :widths: 45 55

   * - EPICS Ethernet Tag
     - Short Description
   * - ``Warning_Exists``
     - Any Warn Exists
   * - ``PLC_Battery_Dead_Wrn``
     - 2 PLC Battery Dead Warn
   * - ``Flow1.Under_Range_Warning``
     - 11 Flow 1 Under Range Warn
   * - ``Flow2.Under_Range_Warning``
     - 12 Flow 2 Under Range Warn
   * - ``Flow3.Under_Range_Warning``
     - 13 Flow 3 Under Range Warn
   * - ``Flow4.Under_Range_Warning``
     - 14 Flow 4 Under Range Warn
   * - ``Flow5.Under_Range_Warning``
     - 15 Flow 5 Under Range Warn
   * - ``Flow6.Under_Range_Warning``
     - 16 Flow 6 Under Range Warn
   * - ``IP1.Warning``
     - 101 Ion Pump 1 Warning
   * - ``IP2.Warning``
     - 102 Ion Pump 2 Warning
   * - ``IP3.Warning``
     - 103 Ion Pump 3 Warning
   * - ``IP4.Warning``
     - 104 Ion Pump 4 Warning
   * - ``IG1.Warning``
     - 201 Ion Gauge 1 Warning
   * - ``IG2.Warning``
     - 202 Ion Gauge 2 Warning
   * - ``IG3.Warning``
     - 203 Ion Gauge 3 Warning
   * - ``Power Supply 1.Warning``
     - 3 Power Supply 1 Warning
   * - ``Power Supply 2.Warning``
     - 4 Power Supply 2 Warning
   * - ``OR'ing Module.Warning``
     - 5 OR'ing Module Warning


``Info`` (29 entries)
---------------------

.. list-table::
   :header-rows: 1
   :widths: 45 55

   * - EPICS Ethernet Tag
     - Short Description
   * - ``Info.Beamline_Number``
     - Sector Number
   * - ``Info.Beamline_Type``
     - BM/ID, 0=BM, 1=ID
   * - ``Info.Software_Version``
     - 
   * - ``Info.CPU_Product_Code``
     - Processor Version
   * - ``Info.CPU_Product_Revision``
     - Processor revision
   * - ``Info.CPU_Serial_Number``
     - Processor Serial Number
   * - ``Info.Year``
     - Current Year
   * - ``Info.Month``
     - Current Month
   * - ``Info.Day``
     - Current Day
   * - ``Info.Hour``
     - Current Hour
   * - ``Info.Minute``
     - Current Minute
   * - ``Info.Second``
     - Current Second
   * - ``Info.Last_Scan_Time``
     - Time for Last scan
   * - ``Info.Max_Scan_Time``
     - Time for Longest scan
   * - ``Info.Major_Events``
     - Processor Major Events
   * - ``Info.Major_Fault_Bits``
     - Processor Major Event Bits
   * - ``Info.Major_Fault.Time_Low``
     - Major Fault Time Low
   * - ``Info.Major_Fault.Time_High``
     - Major Fault Time High
   * - ``Info.Major_Fault.Type``
     - Major Fault Type
   * - ``Info.Major_Fault.Code``
     - Major Fault Code
   * - ``Info.Major_Fault.Info``
     - Major Fault info
   * - ``Info.Minor_Events``
     - Processor Minor Fault
   * - ``Info.Minor_Fault_Bits``
     - Processor Minor Event Bits
   * - ``Info.Minor_Fault.Time_Low``
     - Minor Fault Time Low
   * - ``Info.Minor_Fault.Time_High``
     - Minor Fault Time High
   * - ``Info.Minor_Fault.Type``
     - Minor Fault Type
   * - ``Info.Minor_Fault.Code``
     - Minor Fault Code
   * - ``Info.Minor_Fault.Info``
     - Minor Fault info
   * - ``Info.Communication_Faults``
     - Control Net Faults


``Flows`` (18 entries)
----------------------

.. list-table::
   :header-rows: 1
   :widths: 45 55

   * - EPICS Ethernet Tag
     - Short Description
   * - ``Flow1.Scaling_Factor``
     - Scaling factor for 1st Flow
   * - ``Flow1.Set_Point``
     - Low Setpoint for 1st Flow
   * - ``Flow1.Current_Value``
     - Current value for 1st Flow
   * - ``Flow2.Scaling_Factor``
     - Scaling factor for 2nd Flow
   * - ``Flow2.Set_Point``
     - Low Setpoint for 2nd Flow
   * - ``Flow2.Current_Value``
     - Current value for 2nd Flow
   * - ``Flow3.Scaling_Factor``
     - Scaling factor for 3rd Flow
   * - ``Flow3.Set_Point``
     - Low Setpoint for 3rd Flow
   * - ``Flow3.Current_Value``
     - Current value for 3rd Flow
   * - ``Flow4.Scaling_Factor``
     - Scaling factor for 4th Flow
   * - ``Flow4.Set_Point``
     - Low Setpoint for 4th Flow
   * - ``Flow4.Current_Value``
     - Current value for 4th Flow
   * - ``Flow5.Scaling_Factor``
     - Scaling factor for 5th Flow
   * - ``Flow5.Set_Point``
     - Low Setpoint for 5th Flow
   * - ``Flow5.Current_Value``
     - Current value for 5th Flow
   * - ``Flow6.Scaling_Factor``
     - Scaling factor for 6th Flow
   * - ``Flow6.Set_Point``
     - Low Setpoint for 6th Flow
   * - ``Flow6.Current_Value``
     - Current value for 6th Flow


``Inputs`` (11 entries)
-----------------------

.. list-table::
   :header-rows: 1
   :widths: 45 55

   * - EPICS Ethernet Tag
     - Short Description
   * - ``BIV.Closed``
     - Beamline Iso Valve Closed
   * - ``FES.Closed``
     - Front End Shutter Closed
   * - ``IP1.Status``
     - Ion Pump 1 Vacuum Status
   * - ``IP2.Status``
     - Ion Pump 2 Vacuum Status
   * - ``IP3.Status``
     - Ion Pump 3 Vacuum Status
   * - ``IP4.Status``
     - Ion Pump 4 Vacuum Status
   * - ``IG1.Status``
     - Ion Gauge 1 Vacuum Status
   * - ``IG2.Status``
     - Ion Gauge 2 Vacuum Status
   * - ``IG3.Status``
     - Ion Gauge 3 Vacuum Status
   * - ``GV1.Closed_LS``
     - Vacuum Gate Valve 1 Closed
   * - ``GV1.Opened_LS``
     - Vacuum Gate Valve 1 Opened


``Outputs`` (7 entries)
-----------------------

.. list-table::
   :header-rows: 1
   :widths: 45 55

   * - EPICS Ethernet Tag
     - Short Description
   * - ``BIV.Permit``
     - Beamline Iso Valve Permit
   * - ``FES.Permit``
     - Front End Shutter Permit
   * - ``GV1.Open_Command``
     - Gate Valve 1 Open Command
   * - ``Red_Light``
     - Red Beacon Light
   * - ``Yellow_Light``
     - Yellow Beacon Light
   * - ``Green_Light``
     - Green Beacon Light
   * - ``Buzzer``
     - Audio Buzzer


``Display`` (3 entries)
-----------------------

.. list-table::
   :header-rows: 1
   :widths: 45 55

   * - EPICS Ethernet Tag
     - Short Description
   * - ``GV1.Type``
     - GV1 Type
   * - ``GV1.Open_Permit``
     - GV1 Open Permit
   * - ``GV1.Close_Permit``
     - GV1 Close Permit


``EPICS_Inputs`` (4 entries)
----------------------------

.. list-table::
   :header-rows: 1
   :widths: 45 55

   * - EPICS Ethernet Tag
     - Short Description
   * - ``Fault_Reset_EPICS``
     - EPICS Fault Reset
   * - ``Trip_Reset_EPICS``
     - EPICS Trip Reset
   * - ``GV1.EPICS_Open``
     - EPICS GV1 Open Request
   * - ``GV1.EPICS_Close``
     - EPICS GV1 Close Request

.. rubric:: See also

- :doc:`item_001` — beamline components reference (per-component
  cooling and vacuum details; the gate valve and window / photon
  stop blocks document the BLEPS-monitored hardware).
- BLEPS at the APS is a facility-wide standard; equivalent
  systems exist on every beamline, with the interlock matrix
  tailored to that beamline's hardware. For the analogous 2-BM
  page, see the 2-BM-docs project at ``docs2bm.readthedocs.io``.
