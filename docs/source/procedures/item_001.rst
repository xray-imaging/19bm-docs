==================
Procedure template
==================

This is the canonical skeleton for procedure pages on this site.

**To create a new procedure:** copy this file as
``procedures/item_NNN.rst``, replace the title and the **Name** value
with your procedure's identifier, and fill in every section below.
Sections marked *required* must be present even if the value is
"none"; sections marked *optional* can be omitted entirely.

Each procedure page is meant to register one-to-one with a cora
``Method`` (see `cora <https://github.com/xmap/cora>`__). The
sections below map onto cora as follows:

======================  ====================================================
Section                 Maps to in cora
======================  ====================================================
**Name**                ``Method.name``
**Devices**             ``Plan.wiring`` targets (Assets from item_001)
**Preconditions**       Guard clauses checked before the Method runs
**Parameters**          ``Method.parameters_schema``
**Steps**               Method body — ideally one atomic action per row
**Postconditions**      State asserted on success
**Failure modes**       ``caution`` BC entries
======================  ====================================================


Procedure name (short imperative phrase)
========================================

One short paragraph for human readers: what this procedure does,
when an operator would run it, and the high-level outcome. Avoid
clicking-here-clicking-there detail — that belongs in the
:doc:`../ops` page this procedure links to.


Name
----

``procedure_short_identifier``
*(required — snake_case; becomes ``Method.name`` in cora)*


Devices
-------

*(required — list every Asset / Component this procedure reads
from or writes to. The first time a Device appears it should link
back to :doc:`../manual/item_001` so cora can resolve the wiring.)*

- :doc:`../manual/item_001`: ``<asset_name_1>``
  (e.g. ``Aerotech_ABRS_rotary``)
- :doc:`../manual/item_001`: ``<asset_name_2>``


Preconditions
-------------

*(required — system state that must be true before the procedure
starts. One bullet per check, phrased as an assertion. "none" is
acceptable for procedures with no entry guards.)*

- Shutter is closed.
- MCTOptics IOC is running.
- Rotary is homed.


Parameters
----------

*(required — may be "none")*

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 50

   * - Name
     - Type
     - Unit
     - Description
   * - ``param_1``
     - integer
     - —
     - What the parameter controls.
   * - ``param_2``
     - number > 0
     - mm
     - Range / sensible default if any.


Steps
-----

*(required — one atomic operation per row.)*

.. list-table::
   :header-rows: 1
   :widths: 5 30 65

   * - #
     - Action
     - PV / call
   * - 1
     - Short imperative description of the action.
     - ``caput 2bmb:mNN.VAL <value>`` — or a ``TomoScan`` API
       call, or a softGlue PV write.
   * - 2
     - Wait for completion.
     - ``cawait 2bmb:mNN.DMOV == 1 timeout=30``
   * - 3
     - …
     - …


Postconditions
--------------

*(required — state that must be true after the procedure has
succeeded. Same style as Preconditions, one assertion per bullet.)*

- Rotary is at the requested angle.
- Detector trigger mode is restored to ``FreeRun``.


Failure modes
-------------

*(required — may be "none known")*

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Symptom
     - Recovery
   * - Step N times out
     - What to do (abort, retry once, escalate). Reference a
       ``caution`` if cora already tracks this failure.
   * - Bad parameter combination
     - How the procedure should refuse to start (add a
       precondition).


Operator walkthrough (optional)
-------------------------------

Free-form prose for the human operator: MEDM clicks, sanity checks,
what to expect on the camera live view, etc. Link to the
corresponding :doc:`../ops` page where one exists — do not duplicate
it here.


Notes (optional)
----------------

Anything that doesn't fit the structured block: rationale, history,
links to commissioning logs, references to vendor procedures.
